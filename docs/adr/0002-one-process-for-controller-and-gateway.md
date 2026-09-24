# One process holds the controller and the Gateway

The Gateway must read PromptRollout objects on every request. A second process means a second informer cache and a second watch. We put the controller and the Gateway in one binary, on one `controller-runtime` manager, so they share one cache.

Leader election makes this work with more than one replica. The reconciler is a leader-election runnable, so only the leader reconciles. The Gateway HTTP server is a non-leader-election runnable, so every replica serves prompt text from its own warm cache.

## Consequences

- The controller and the Gateway scale together and share a failure domain. Gateway request load can slow reconciliation. This is acceptable for v0.1.
- The rule in `docs/best-practices.md` stands: do not open a second watch on PromptRollout.
- Section 7 of the design document is now wrong. There is one Deployment, not two.
- A split into two Deployments later is a real cost: new RBAC, a new Service, and a second cache.
