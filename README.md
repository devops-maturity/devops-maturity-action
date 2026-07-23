# devops-maturity-action

[![CI](https://github.com/devops-maturity/devops-maturity-action/actions/workflows/ci.yml/badge.svg)](https://github.com/devops-maturity/devops-maturity-action/actions/workflows/ci.yml)

A GitHub Action that automatically runs the [devops-maturity](https://github.com/devops-maturity/devops-maturity) CLI whenever `devops-maturity.yml` is updated, generates a new badge reflecting the current maturity level, and opens a pull request for team review.

## Overview

When a team edits their `devops-maturity.yml` assessment file the action will:

1. Install the `devops-maturity` CLI.
2. Run `dm config --file devops-maturity.yml` to calculate the score and badge.
3. Update the `README.md` badge in-place (or prepend one if none exists).
4. Open (or update) a pull request containing the badge change for review.

Set `report: true` to also generate a human-readable maturity report in the
workflow run summary (see [Reports](#reports)).

## Usage

### Minimal setup

Add the following workflow file to your repository at
`.github/workflows/devops-maturity.yml`:

```yaml
name: DevOps Maturity Check

on:
  push:
    branches:
      - main
    paths:
      - 'devops-maturity.yml'
  workflow_dispatch:

jobs:
  update-badges:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write

    steps:
      - uses: actions/checkout@v4

      - uses: devops-maturity/devops-maturity-action@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

### Assessment file

Place a `devops-maturity.yml` file at the root of your repository. Use
`dm assess` interactively to generate the initial file, or start from the
template below:

```yaml
# DevOps Maturity Assessment
# https://devops-maturity.github.io/
project_name: my-project

# Basics
D101: true    # Branch Builds (must have)
D102: true    # Pull Request Builds (must have)
D103: false   # Clean Build Environments (nice to have)

# Quality
D201: true    # Unit Testing (must have)
D202: false   # Functional Testing (must have)
# … (see devops-maturity.yml in this repo for the full list)
```

## Inputs

| Input            | Required | Default                                    | Description                                              |
|------------------|----------|--------------------------------------------|----------------------------------------------------------|
| `github-token`   | **yes**  | —                                          | Token used to create the pull request. Pass `${{ secrets.GITHUB_TOKEN }}`. |
| `file`           | no       | `devops-maturity.yml`                      | Path to the assessment YAML file.                        |
| `readme-path`    | no       | `README.md`                                | Path to the README file to update with the badge.        |
| `pr-branch`      | no       | `chore/update-devops-maturity-badges`      | Branch name for the pull request.                        |
| `commit-message` | no       | `chore: update devops-maturity badges`     | Commit message for the badge update.                     |
| `pr-title`       | no       | `chore: update devops-maturity badges`     | Title of the pull request.                               |
| `pr-body`        | no       | *(auto-generated)*                         | Body text of the pull request.                           |
| `cli-version`    | no       | *(latest)*                                 | Pin a specific CLI version for reproducible results (e.g. `0.1.0`). |
| `report`         | no       | `false`                                    | When `true`, write a maturity report (score, category breakdown, recommendations) to the run summary. |

## Outputs

| Output               | Description                                               |
|----------------------|-----------------------------------------------------------|
| `score`              | Overall maturity score as a percentage (e.g. `"72.3%"`). |
| `level`              | Maturity level: `WIP`, `PASSING`, `BRONZE`, `SILVER`, or `GOLD`. |
| `badge-url`          | shields.io badge URL for the current maturity level.      |
| `badge-markdown`     | Ready-to-paste Markdown snippet for the badge.            |
| `report-path`        | Path to the generated Markdown report (only when `report: true`). |
| `pull-request-number`| Number of the created (or updated) pull request.          |
| `pull-request-url`   | HTML URL of the created (or updated) pull request.        |

### Using outputs in subsequent steps

```yaml
- id: maturity
  uses: devops-maturity/devops-maturity-action@v1
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}

- name: Print results
  run: |
    echo "Score : ${{ steps.maturity.outputs.score }}"
    echo "Level : ${{ steps.maturity.outputs.level }}"
    echo "PR    : ${{ steps.maturity.outputs.pull-request-url }}"
```

## Reports

Set `report: true` to generate a human-readable maturity report — the overall
score and level, a per-category breakdown, and the list of unmet criteria with
improvement recommendations. The report is written to the workflow run's
[job summary](https://docs.github.com/actions/using-workflows/workflow-commands-for-github-actions#adding-a-job-summary),
so it shows up directly on the Actions run page — no artifact download needed.

```yaml
- uses: devops-maturity/devops-maturity-action@v1
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    report: true
```

The same Markdown is also written to a file, exposed via the `report-path`
output, so you can upload it as an artifact if you want to keep it:

```yaml
- id: maturity
  uses: devops-maturity/devops-maturity-action@v1
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    report: true

- uses: actions/upload-artifact@v4
  with:
    name: devops-maturity-report
    path: ${{ steps.maturity.outputs.report-path }}
```

## Permissions

The workflow job must have the following permissions:

```yaml
permissions:
  contents: write       # to push the badge-update branch
  pull-requests: write  # to open the pull request
```

## How the badge update works

The action searches the target README for an existing DevOps Maturity
shields.io badge using a regular expression. If one is found it is replaced
in-place; if none exists the new badge is prepended before the first Markdown
heading (or at the very top of the file).

## Reproducibility

For reproducible CI results, pin both the action and the CLI to specific versions:

```yaml
- uses: devops-maturity/devops-maturity-action@v1
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    cli-version: '0.1.0'  # pin CLI version
```

## License

[Apache 2.0](LICENSE)