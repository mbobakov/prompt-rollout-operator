# The Stable text lives only in `.status`

`.spec` holds one prompt text: the one you want live. `.status` holds the Stable text that serves now and, during a rollout, the Canary text too. `.spec` never holds a Stable field.

The alternative was to keep both Slots in `.spec` and make Promotion a Git commit. That was rejected because Promotion is the operator's job. An operator that climbs 10 → 50 → 100 by itself and then stops to wait for a human has automated the easy part and left the last step manual.

## Consequences

- **Promotion is automatic.** The controller copies the Canary text into the Stable slot. No commit, no Git credentials in the operator.
- **A rebuilt cluster recovers the live prompt, not the rollout.** `.status` is empty after a rebuild, so the cold-start rule promotes `spec.canary.text` at once, and the prompt you declared is live again. What is lost is the previous Stable, so there is no Rollback target until the next rollout.
- **During a rollout, the serving text is not in Git.** Most callers still get the older Stable, and that text exists only in the cluster. Between rollouts the two match.
- The v0.2 way to get both: the controller opens a pull request that records the Promotion. That needs Git credentials and a PR client in the operator, so it is out of v0.1.
