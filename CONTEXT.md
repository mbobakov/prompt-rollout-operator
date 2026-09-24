# PromptRollout

A Kubernetes operator that rolls out LLM prompt text as a versioned artifact: gradual canary, automatic gate, automatic rollback.

## Language

### Slots

**Slot**:
One of the two places a prompt text can sit: Stable or Canary. A Slot names a prompt text. There are no version ids.
_Avoid_: version, variant, revision

**Stable**:
The prompt text that serves every caller outside the Canary bucket. It keeps serving through the whole rollout.
_Avoid_: active version, current version, baseline

**Canary**:
The prompt text under test. There is at most one Canary per PromptRollout.
_Avoid_: candidate, challenger, new version

### Rollout

**PromptRollout**:
One named prompt, its two Slots, and its rollout plan. The custom resource, and the unit a caller asks for by name.

**Canary Weight**:
The percentage of callers that get the Canary. The Stable serves the rest.
_Avoid_: traffic split, canary percentage

**Rollout Step**:
One Canary Weight in the ordered plan, for example `10`, `50`, `100`.
_Avoid_: stage, phase

**Dwell Time**:
The minimum time the Canary must hold a Rollout Step before an advance.
_Avoid_: soak time, bake time, step interval

**Force**:
A flag in Git that makes the controller skip the Eval. Every other gate still applies. See [ADR-0003](./docs/adr/0003-force-skips-the-eval.md).
_Avoid_: override, bypass, skip

**Promotion**:
The act that makes the Canary the new Stable.
_Avoid_: cutover, graduation

**Rollback**:
The controller sets the Canary Weight to 0. The Canary stays declared, but takes no traffic.
_Avoid_: revert, abort

### Quality signals

**Eval**:
The offline score of a Canary against a fixed dataset. It runs before any live traffic reaches the Canary.
_Avoid_: test, benchmark, offline test

**Eval Score**:
The result of an Eval, from 0 to 1. It must pass a threshold before the Canary Weight can leave 0.

**Feedback**:
A quality signal that a caller reports for the Slot it received.
_Avoid_: rating, vote, telemetry

**Feedback Score**:
The aggregate of Feedback for one Slot within the current Rollout Step. It gates the advance to the next Rollout Step.
_Avoid_: live score, quality score

### Components

**Gateway**:
The service that gives prompt text to applications, and that collects Feedback. It picks the Slot per caller.
_Avoid_: proxy, router, prompt server

**Eval Runner**:
The Kubernetes Job that produces an Eval Score for a Canary.
