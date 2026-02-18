# Contributing to TIHLDE Reusable Workflows

Thanks for your interest in improving our shared CI/CD workflows!

## How to contribute

1. **Fork** this repository and create a feature branch from `main`.
2. Make your changes — keep commits small and focused.
3. **Test** your workflow changes in a fork before opening a PR (use `workflow_dispatch` to trigger manually).
4. Open a **Pull Request** against `main` with a clear description of what changed and why.

## Guidelines

- Keep workflows **generic and reusable** — avoid repo-specific logic.
- Pin all third-party Actions to a **full commit SHA** (not a tag) for supply-chain security.
- Add a trailing comment with the Action version for readability, e.g.:
  ```yaml
  uses: actions/checkout@abc123def456 # v4.1.0
  ```
- Update the README if you add, remove, or change any inputs/secrets/outputs.
- Run `actionlint` locally if available to catch YAML issues early.

## Code review

All changes to `.github/workflows/` require approval from a CODEOWNER (see `CODEOWNERS`).

## Questions?

Open a [Discussion](https://github.com/TIHLDE/workflows/discussions) or reach out via the TIHLDE Slack/Discord.
