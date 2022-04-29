# gitops-cluster-releases

This repository contains GitOps definitions for cluster releases used by Giant Swarm.

If you want to use them as a customer, please check the
[gitops-template repo](https://github.com/giantswarm/gitops-template).

If you want to contribute and add bases for a new releases, please check [docs](/docs/add_release.md).

## Contributing

To ensure your YAML and Markdown formatting is OK even before you push to the repository,
we have prepared [`pre-commit` config](.pre-commit-config.yaml). To use it, make sure to:

- [install](https://pre-commit.com/#install) `pre-commit`
- when contributing to the repo for the first time, run `pre-commit install --install-hooks`

Remember:

- `pre-commit` is optional and opt-in: you have to set it up yourself.
- To check your code without doing git commit, you can run `pre-commit run -a`
- To force a git commit without running `pre-commit` hook, run `git commit --no-verify ...`
