# Force skips the Eval

`spec.rollout.force` lets the rollout start without a passing Eval Score. Nothing else changes: the Rollout Steps, the Dwell Time, the minimum request count, and the Rollback on a bad Feedback Score all still apply.

A safety operator with a bypass looks wrong at first sight, so the reason is here. An Eval dataset cannot cover an incident that nobody predicted, and a hotfix must not wait for a threshold that somebody wrote last month. Live traffic still gates the rollout, so the Canary is still tested. Users test it instead of a dataset.

ADR-0001 keeps the controller out of `.spec`, so only a Git commit can set `force`. Every use is reviewed and audited like any other change.
