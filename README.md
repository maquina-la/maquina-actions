# Máquina GitHub Actions

Precompiled, versioned GitHub Actions for Máquina's governed software-delivery workflows.

## PR operational advisory

`maquina-la/maquina-actions/pr-advisory@v0` runs the portable Máquina evaluator **inside the customer's GitHub Actions runner**. It requires no Go installation, source checkout from Máquina, external service, or model-provider credential.

The action reads the customer's checked-out Factory Contract and GitHub pull-request metadata using the supplied GitHub token. It writes only a sanitized JSON/Markdown evidence pair into the customer's workspace. It is an operational advisory, not a code review, code-quality judgment, or merge gate.

```yaml
name: Máquina PR advisory

on:
  pull_request:
    types: [opened, reopened, synchronize]

permissions:
  contents: read
  pull-requests: read
  checks: read

jobs:
  advise:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          persist-credentials: false

      - id: maquina
        uses: maquina-la/maquina-actions/pr-advisory@v0
        with:
          github-token: ${{ github.token }}
          checks: |
            customer-ci=success

      - uses: actions/upload-artifact@v7
        with:
          name: maquina-sanitized-evidence
          path: |
            ${{ steps.maquina.outputs.evidence-json }}
            ${{ steps.maquina.outputs.evidence-markdown }}
          if-no-files-found: error
          retention-days: 7
```

The repository must include a valid `.maquina/factory-contract.json` and every path it declares. Start with the contract schema and a small PR-first policy; do not copy Máquina's own contract blindly.

## Trust boundary

The action currently supports Linux x64 GitHub-hosted runners. Its binary is built from the public [`maquina-la/maquina`](https://github.com/maquina-la/maquina) source at the release noted below. The action makes GitHub API calls only to hydrate the pull-request facts necessary for the advisory; the GitHub token remains in the runner. The current action neither sends code, diffs, PR text, prompts, nor raw model output to Máquina services.

For higher-assurance use, pin the Action to an immutable commit SHA rather than a moving major tag, and mirror this repository inside the customer's GitHub Enterprise organization when external Actions are prohibited.

## Release policy

- `v0` is the moving compatibility tag for the pre-1.0 action.
- Every release receives an immutable semantic tag and a GitHub Release with the binary's SHA-256 digest and the matching Máquina source commit.
- The initial `v0.1.0` package is built from Máquina source commit `55e31af4394df7f1c925b6887350b7a5bda12c2e`.

Customer workflows must never grant write permissions merely to run the advisory. Evidence publication or PR-comment projection is a separate, explicitly configured concern.
