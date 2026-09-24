# Prompt text lives in the custom resource

Prompt text sits inline in the PromptRollout resource. A `configMapKeyRef` was the alternative, and it is rejected: it splits one truth across two objects, so a reviewer reads a rollout plan in the pull request without the prompt text that it rolls out. Inline text puts the prompt and its plan in one commit and one diff.

## Consequences

- A text field is capped at 32768 bytes by `+kubebuilder:validation:MaxLength`. Three copies can exist at once: the Canary in `.spec`, and both Slots in `.status`. The worst case is 96 KB. That stays under the 256 KB annotation limit, so plain `kubectl apply` keeps working, and far under the 1.5 MB object limit.
- A prompt longer than 32 KB is not supported. External storage is a v0.2 problem.
- `kubectl get -o yaml` prints the prompt text. `additionalPrinterColumns` keeps `kubectl get` readable.
