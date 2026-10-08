# LiteLLM chart preview for PR #45316

This branch hosts a standard Helm HTTP repository containing an unchanged package of `helm/litellm` from contribution commit `6f82d81bd5c44e8f55db6de0f0ae6a439fe5eefe`. It is a temporary integration-test artifact, not an upstream release.

Chart version and appVersion are preserved at `0.1.0`; integration values explicitly pin published backend, gateway and migration images and the contributed UI image. Consumers pin this repository by commit, rather than by its mutable branch name.

Source: https://github.com/BerriAI/litellm/pull/45316

Packaged with `helm package helm/litellm` and indexed with `helm repo index`. Relative package URLs resolve within the pinned repository commit.
