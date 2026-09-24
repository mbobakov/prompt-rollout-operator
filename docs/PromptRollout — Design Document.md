# PromptRollout — Design Document

Sep 23, 2026 · @Someone

## 1. Overview

**PromptRollout** is a Kubernetes operator that treats LLM prompt text the way Kubernetes treats container images: as a versioned artifact that gets rolled out gradually, monitored, and automatically rolled back if it underperforms.

Today, teams manage prompts either as hardcoded strings, plain ConfigMaps (no versioning, no gradual rollout, no safety net), or bespoke application-level logic. There is no Kubernetes-native primitive for "ship this new prompt to 10% of traffic, watch the quality signal, and promote or revert automatically" — the same operational rigor that exists for deployments (via Argo Rollouts, Flagger) or model versions (via KServe's `LLMInferenceService` canary support) does not exist for prompt text itself.

PromptRollout closes that gap with:

- A **CRD** (`PromptRollout`) that stores prompt versions and a declarative rollout plan (canary weights, eval gate).
- A **controller** that reconciles the CRD, runs eval jobs, and advances or rolls back the canary based on results.
- A **gateway** service that in-cluster applications call to fetch the currently active prompt text, with per-request version selection according to the live rollout weights.

As a showcase project, it demonstrates: custom controllers with a real state machine (not just CRUD), a companion service sharing the controller's informer cache, integration with Prometheus for rollout gating, and a genuinely novel niche — nothing in the ecosystem currently combines Kubernetes-native canary rollout mechanics with prompt-level (rather than model- or deployment-level) versioning.

## 2. Naming

|  |  |
| --- | --- |
| Repository | `prompt-rollout-operator` |
| CRD group | `prompts.mbobakov.github.io` |
| CRD kind | `PromptRollout` |
| CLI | `kubectl get promptrollout`, `kubectl describe promptrollout <name>` |
| Container image | `ghcr.io/mbobakov/prompt-rollout-operator` |

**Rationale:**

- `PromptRollout` mirrors Argo's `Rollout` kind directly — the name signals "this behaves like a Deployment/Rollout, just for prompt text instead of a container image" immediately to anyone who knows the ecosystem.
- The `-operator` suffix keeps the repo name unambiguous and searchable, at the cost of being a common pattern — acceptable tradeoff for a showcase project where clarity beats cleverness.
- `mbobakov.github.io` is used as the CRD group domain in place of a registered domain, following the common convention for personal Kubernetes projects (also what Kubebuilder suggests when scaffolding with `--domain <you>.github.io`).

**Kubebuilder scaffolding:**

```bash
kubebuilder init --domain mbobakov.github.io --repo github.com/mbobakov/prompt-rollout-operator
kubebuilder create api --group prompts --version v1alpha1 --kind PromptRollout
```

## 3. Prior Art

No existing project combines Kubernetes-native canary rollout mechanics with prompt-text-level versioning. The closest adjacent work:

- **PromptCanary** (Ruby gem) — canary deployment of LLM prompts with traffic splitting, telemetry, and auto-rollback on error rate/latency. Same conceptual shape, but it's a Rails/Ruby library, not a Kubernetes operator: no CRD, no controller, no cluster-native rollout, no GitOps story.
- **KServe `LLMInferenceService` canary rollout** — weighted traffic splitting between two versions of a *model serving endpoint*, via Gateway API `HTTPRoute` weights. This is the closest Kubernetes-native pattern, but it canaries the whole inference service (model + runtime), not individual prompt strings sent to a shared model.
- **Argo Rollouts + KServe (GitOps writeups)** — combines Argo Rollouts' weighted canary steps with KServe `InferenceService`, gated on Prometheus metrics. Same shape as PromptRollout's rollout mechanics, again applied to model versions rather than prompts.
- **Argo Rollouts AnalysisTemplate + LLM-judge** — uses an LLM as the *judge* of a canary (scores canary vs. stable pods, exits 0/1 to gate promotion). This is the inverse of PromptRollout's idea: AI judging a rollout, rather than a rollout of AI content.
- **LiteLLM/llama-stack operators** — manage LLM gateway/serving infrastructure lifecycle (Helm-style CRDs for connection config), not prompt content versioning.

**Positioning:** PromptRollout is the natural extension of the pattern KServe established for models, applied one layer up — to the prompt text itself, which is where product/eng teams iterate far more frequently than on model versions.
## 4. Architecture

**Components:**

1. **`PromptRollout` CRD** — `.spec` declares the prompt text you want live, plus the rollout plan and the eval settings. `.status` holds what is live now. See [ADR-0005](../docs/adr/0005-stable-lives-only-in-status.md).
2. **Controller** — a `controller-runtime` reconciler. It runs the Eval Job, drives the rollout state machine, and writes the result to `.status`. It never writes `.spec` ([ADR-0001](../docs/adr/0001-controller-never-writes-spec.md)).
3. **Gateway** — an HTTP server in the **same process** as the controller, on the same manager and the same cache ([ADR-0002](../docs/adr/0002-one-process-for-controller-and-gateway.md)). It serves prompt text from `.status` only, and it collects Feedback.
4. **Eval Runner** — a Kubernetes `Job` that the controller starts. It scores the Canary against a dataset in a ConfigMap, through an OpenAI-compatible endpoint.

**The one-way path.** Prompt text enters `.status` only through the controller, and only after the Eval passes. The Gateway reads `.status` only. Untested prompt text therefore cannot reach a user. This is structural, not a rule that some code path must remember.

**Data flow:**

```
Git commit (new prompt text + rollout plan)
   ↓
PromptRollout .spec updated
   ↓
Controller sees spec text != status Stable text
   ↓
Eval Job runs against the dataset          (skipped if force: true)
   ↓ (pass)                                  ↓ (fail)
Copy text into .status Canary,           Hold. Canary Weight stays 0.
set the first Rollout Step weight        The Stable keeps serving.
   ↓
Gateway serves the Canary to that percentage of callers, and
tags the response X-Prompt-Slot: canary
   ↓
Callers POST Feedback → controller compares Canary against Stable
within the current step → advance, or roll back to weight 0
   ↓ (reached 100%)
Promotion: .status Stable ← Canary text. No commit needed.
```

No external database. The cluster holds the live prompt; Git holds the desired one.

## 5. CRD Spec

```yaml
apiVersion: prompts.mbobakov.github.io/v1alpha1
kind: PromptRollout
metadata:
  name: support-bot
  namespace: default
spec:
  canary:
    text: "You are a concise support agent..."   # the prompt you want live
  rollout:
    steps: [10, 50, 100]          # Canary Weight percentages, in order
    stepIntervalSeconds: 300      # Dwell Time
    minRequestsPerStep: 50        # requests the Canary must serve before an advance
    force: false                  # skip the Eval only (ADR-0003)
  evalRef:
    datasetConfigMap: support-evals
    endpoint: http://eval-llm.default.svc/v1
    minScore: 0.85
status:
  stable:
    text: "You are a helpful support agent..."   # what serves now
  canary:
    text: "You are a concise support agent..."   # copied in only after the Eval passed
    weight: 10
  lastEvalScore: 0.91
  conditions:
    - type: EvalPassed
      status: "True"
      reason: ScoreAboveThreshold
      lastTransitionTime: "2026-09-23T10:00:00Z"
    - type: Progressing
      status: "True"
      reason: AdvancingCanary
    - type: RolledBack
      status: "False"
```

**Field notes:**

- `spec.canary.text` is the prompt you want live. It is the Canary while it differs from `status.stable.text`. After a Promotion the two match, and the controller has nothing to do.
- There is no version id and no version list. A Slot names a prompt text. Feedback that arrives just after a Promotion lands on the wrong Slot; this is a known limitation, recorded in the README.
- Both text fields carry `+kubebuilder:validation:MaxLength=32768`. Three copies can exist at once, so the worst case is 96 KB ([ADR-0004](../docs/adr/0004-prompt-text-lives-in-the-custom-resource.md)).
- `status.conditions` follows the standard Kubernetes condition pattern, so `kubectl describe` and `kubectl wait` work without extra work.

## 6. Controller

**States**, derived from `.status`:

```
Synced ── spec text changes ──→ Pending ──→ Progressing ──→ Promoted ──→ Synced
                                              ↓
                                          RolledBack
```

- **Synced** — `spec.canary.text` equals `status.stable.text`. Nothing to do.
- **Pending** — the texts differ, and the Eval has not passed yet. Canary Weight is 0.
- **Progressing** — the Eval passed. The Canary text is in `.status` and takes live traffic.
- **Promoted** — the Canary reached 100% and held. `status.stable.text` takes the Canary text. No commit needed.
- **RolledBack** — the Feedback Score fell behind the Stable. Canary Weight returns to 0. The Stable never stopped serving.

**Cold start.** When `status.stable.text` is empty, nothing serves at all, so any wait is a full outage. The controller copies the text straight into the Stable slot and promotes at once. The Eval still runs, and its score goes into `status.lastEvalScore` for the record, but it gates nothing.

**Reconcile loop (pseudocode):**

```go
func (r *PromptRolloutReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var pr promptsv1alpha1.PromptRollout
    if err := r.Get(ctx, req.NamespacedName, &pr); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    desired := pr.Spec.Canary.Text

    // Nothing serves yet, so any gate here is a full outage.
    if pr.Status.Stable.Text == "" {
        return r.promote(ctx, &pr, desired)
    }
    if desired == pr.Status.Stable.Text {
        return ctrl.Result{}, nil // Synced
    }
    // The declared text moved under a running rollout. Start again.
    if desired != pr.Status.Canary.Text {
        return r.restart(ctx, &pr)
    }
    if !pr.Spec.Rollout.Force && !pr.Status.EvalPassed() {
        return r.runEval(ctx, &pr)
    }
    if pr.Status.Canary.Weight == 0 {
        return r.startCanary(ctx, &pr) // copy text into .status, take the first step
    }

    canary, stable := r.stepFeedback(ctx, &pr) // current Rollout Step only
    if canary.Count >= pr.Spec.Rollout.MinRequestsPerStep {
        if canary.Score < stable.Score {
            return r.rollBack(ctx, &pr)
        }
        if r.dwellElapsed(&pr) {
            return r.advanceStep(ctx, &pr) // next step, or Promotion at 100
        }
    }
    return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
}
```

**Design notes:**

- `restart` is what makes an edit to a running Canary safe: the weight returns to 0 and the Eval runs again before any user sees the new text.
- The Eval catches a regression before live traffic. The Feedback catches a regression that the dataset did not predict. `force` removes the first gate only.
- A Rollback is idempotent and needs no cleanup, because the Stable text never moved.
- Finalizers are not needed. The Eval Job carries an `OwnerReference` and is garbage-collected.

## 7. Gateway

An HTTP server on the controller's manager, started as a non-leader-election runnable, so every replica serves while one replica reconciles.

**Endpoints:**

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/prompts/{name}?userID=<id>` | Returns the prompt text for the caller, Slot chosen by the live Canary Weight |
| `POST` | `/prompts/{name}/feedback` | The caller reports a score from 0 to 1, with the Slot it received |
| `GET` | `/scores/{name}` | Per-Slot aggregate for the current Rollout Step. The controller reads this |
| `GET` | `/metrics` | Prometheus scrape endpoint, for humans and for later |

**Response shape:**

```json
{ "slot": "canary", "text": "You are a concise support agent..." }
```

The same value goes in an `X-Prompt-Slot` header.

**Slot selection (deterministic hashing):**

```go
func SelectSlot(st PromptRolloutStatus, userID string) Slot {
    if userID == "" {
        userID = uuid.NewString() // no stable identity -> a fresh bucket per request
    }
    h := fnv.New32a()
    h.Write([]byte(userID))
    if h.Sum32()%100 < uint32(st.Canary.Weight) {
        return SlotCanary
    }
    return SlotStable
}
```

Hashing a stable `userID` pins one caller to one Slot across calls. Without that, a caller flips between Slots, and the Feedback Score of each Slot measures a mixture instead of a prompt.

**Metrics:**

- `prompt_requests_total{name, slot}` — counter
- `prompt_feedback_score{name, slot, weight}` — histogram; the `weight` label is what makes the per-step window possible
- `prompt_request_duration_seconds{name, slot}` — histogram

## 8. Eval and Feedback Loop

**1. Eval (before traffic)**

- The controller starts it when `spec.canary.text` differs from both Slots in `.status`.
- It runs as a Kubernetes `Job`, owned by the PromptRollout.
- It reads a dataset from the ConfigMap named by `evalRef.datasetConfigMap`: a JSON list of input and expected-output pairs.
- It sends each input through `evalRef.endpoint`, an OpenAI-compatible URL, using the Canary text as the system prompt. It compares each answer to the expected output with a deterministic scorer, and writes the mean to the Job result.
- The demo points `evalRef.endpoint` at a stub server in the cluster. The demo then needs no API key and gives the same score every run. A real user points the same field at a real model.
- The score must reach `evalRef.minScore` before any text enters the `.status` Canary slot. `force: true` skips this gate, and only this gate.

**2. Feedback (during traffic)**

- Callers POST a score from 0 to 1 with the Slot they received.
- The Gateway aggregates per Slot **within the current Rollout Step**. A bad first step cannot poison a later one, and a good first step cannot hide a bad later one.
- The controller reads `/scores/{name}` and compares the Canary against the Stable.
- Nothing stops a caller sending a false score. This is a known limitation, called out in the README, not solved in v0.1.

**Decision table:**

| Condition | Action |
| --- | --- |
| No Stable in `.status` | Promote at once. Nothing serves otherwise |
| Eval score < `minScore`, `force` false | Hold. Canary Weight stays 0 |
| Spec text changed mid-rollout | Restart: weight 0, Eval again |
| Requests in this step < `minRequestsPerStep` | Hold at the current weight |
| Canary Feedback Score < Stable Feedback Score | Roll back. Weight → 0 |
| Dwell Time elapsed and requests enough and score holds | Advance to the next Rollout Step |
| At 100% and the score holds | Promote. `status.stable.text` ← Canary text |

## 9. MVP Scope

**In scope for v0.1:**

- `PromptRollout` CRD and controller, Kubebuilder scaffolded
- Rollout state machine: Synced, Pending, Progressing, Promoted, RolledBack, with `status.conditions`
- Eval Runner Job: JSON dataset in a ConfigMap, OpenAI-compatible endpoint, deterministic scorer
- Gateway: `GET /prompts/{name}`, `POST /prompts/{name}/feedback`, `GET /scores/{name}`, `/metrics`
- Deterministic hash-based Slot selection
- `envtest` controller tests, plus the kubebuilder `test-e2e` suite on kind
- Install through the generated `config/` kustomize tree and one `install.yaml`
- README with an architecture diagram, a demo GIF, and the known limitations

**Deferred to v0.2 and later:**

- Helm chart. Kustomize covers v0.1, and a second packaging path adds nothing to the demo
- Prometheus as the metrics backend. The controller reads `/scores/{name}` in v0.1
- LLM-as-judge scorer. The scorer is pluggable, so this is a new scorer, not a redesign
- The controller opening a pull request to record a Promotion in Git
- Client SDK, admission webhook, multi-tenancy, a dashboard
- Prompt text longer than 32 KB, which needs external storage

## 10. Demo Script

1. `kubectl apply -f examples/support-bot.yaml` on an empty cluster. There is no Stable, so the prompt goes live at once. `kubectl get promptrollout` shows it Synced.
2. Edit the prompt text in the manifest and apply again. The Eval Job starts: `kubectl get jobs -w`. Then `kubectl describe promptrollout support-bot` shows `EvalPassed: True`.
3. Loop `curl` against `/prompts/support-bot?userID=<random>`. About 10% of the responses carry `X-Prompt-Slot: canary`.
4. POST good Feedback for the canary Slot. `kubectl get promptrollout support-bot -w` shows the weight climb 10 → 50 → 100, then the Promotion. No human touched the cluster.
5. Reverse it. Apply a worse prompt, feed it poor scores, and watch the controller roll back: `RolledBack: True`, weight 0, and the Stable served every request throughout.
6. Close on `git log` of the manifest: every prompt change is a reviewed commit.

## 11. Tech Stack and Repo Layout

**Stack:**

- Go, Kubebuilder and `controller-runtime`
- One manager, one cache, shared by the reconciler and the Gateway
- `net/http` for the Gateway
- Prometheus client library for metrics
- `envtest` for controller tests, kind for `make test-e2e`
- Kustomize for install

**Repo layout:**

```
prompt-rollout-operator/
├─ api/v1alpha1/
│  └─ promptrollout_types.go       # spec, status, MaxLength markers
├─ internal/controller/
│  ├─ promptrollout_controller.go  # Reconcile loop
│  ├─ eval.go                      # Eval Job start and result read
│  └─ statemachine.go              # step advance, rollback, promotion
├─ internal/gateway/
│  ├─ server.go                    # HTTP handlers
│  ├─ selector.go                  # SelectSlot hashing
│  └─ metrics.go
├─ internal/evalrunner/            # dataset load, endpoint call, scorer
├─ cmd/
│  ├─ manager/main.go              # controller + Gateway, one process
│  └─ evalrunner/main.go           # the Job entrypoint
├─ config/                         # Kubebuilder CRD, RBAC, manager manifests
├─ examples/support-bot.yaml
└─ test/e2e/
```

One image, two entrypoints. The manager and the Eval Runner share the API types, so one build and one version number cover both.

## 12. Open Questions

The open questions from the first draft are now decided, and the decisions are in `docs/adr/`. Remaining questions live as decision tickets on the wayfinder map, not in this document.
