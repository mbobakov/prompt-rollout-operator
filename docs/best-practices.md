# Best Practices

Repo-specific conventions not obvious from the code or `kubebuilder --help`. Living document — add a rule here the moment a review or a bug reveals one; delete a rule the moment the code makes it obsolete.

## Stack

- Kubebuilder scaffold, `controller-runtime` reconciler. CRD group `prompts.mbobakov.github.io`, kind `PromptRollout`.
- Gateway service shares the controller's informer cache — do not open a second watch on `PromptRollout` from the gateway process.

## Conventions

_(none yet — add here as they emerge)_
