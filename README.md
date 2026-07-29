# Máquina GitHub Actions

Precompiled, versioned GitHub Actions for Máquina's governed software-delivery workflows.

## PR operational advisory

`maquina-la/maquina-actions/pr-advisory@v0.1.4` runs the portable Máquina evaluator **inside the customer's GitHub Actions runner**. It requires no Go installation, source checkout from Máquina, external service, or model-provider credential.

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
      # Contract and evaluator inputs must come from the trusted base revision,
      # never from untrusted pull-request content.
      - uses: actions/checkout@v7
        with:
          ref: ${{ github.event.pull_request.base.sha }}
          path: trusted
          persist-credentials: false

      - id: maquina
        uses: maquina-la/maquina-actions/pr-advisory@v0.1.4
        with:
          github-token: ${{ github.token }}
          repository-root: trusted
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

The repository's trusted base revision must include a valid `.maquina/factory-contract.json` and every path it declares. Start with the contract schema and a small PR-first policy; do not copy Máquina's own contract blindly. Run candidate validation in a separate, untrusted checkout, then pass only its bounded `NAME=CONCLUSION` result through `checks`—do not execute candidate code in the trusted advisory step.

## Trust boundary

The action currently supports Linux x64 GitHub-hosted runners. Its binary is built from the public [`maquina-la/maquina`](https://github.com/maquina-la/maquina) source at the release noted below. The action makes GitHub API calls only to hydrate the pull-request facts necessary for the advisory; the GitHub token remains in the runner. The current action neither sends code, diffs, PR text, prompts, nor raw model output to Máquina services.

For higher-assurance use, pin the Action to an immutable commit SHA rather than a moving major tag, and mirror this repository inside the customer's GitHub Enterprise organization when external Actions are prohibited.

## Release policy

- `v0` is the moving compatibility tag for the pre-1.0 action.
- Every release receives an immutable semantic tag and a GitHub Release with the binary's SHA-256 digest and the matching Máquina source commit.
- `v0.1.4` is built from Máquina source commit `0cc0ef5f80cae081aec2c44d43534bfefee0ad50`; it permits Git-only contracts and adds opt-in, same-repository PR comment projection.

## Optional PR comment

The Action defaults to no publication. To create or update one marked **operational advisory** comment, explicitly set `publish-comment: true` and grant only `pull-requests: write` in the calling workflow. Publication is automatically suppressed for fork pull requests. The comment is rendered only from the sanitized record and does not act as a code review, quality judgment, or merge decision.

Customer workflows must never grant write permissions merely to run the advisory. Comment projection is an explicit, same-repository-only concern.
