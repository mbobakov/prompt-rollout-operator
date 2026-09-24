# The controller never writes to `.spec`

The first draft made Promotion set `spec.rollout.stable` to the Canary. A GitOps engine such as Argo CD owns `.spec`, so it reverts that write at the next sync, and the operator and the engine fight each other. The controller therefore writes only to `.status`.

## Consequences

- The prompt that serves now is a `.status` fact. `.spec` declares the prompt you want. See [ADR-0005](./0005-stable-lives-only-in-status.md).
- Promotion and Rollback both need no commit, because both move only `.status`.
- `spec.rollout.force` is readable but never writable by the controller, so every use of it is a reviewed commit. See [ADR-0003](./0003-force-skips-the-eval.md).
