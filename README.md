# mulzatech-infra-reviewer

An AI reviewer for infrastructure pull requests. On a PR it reads the diff
against a short rubric of real mistakes (mutable image tags, privilege
grants made by automation, leaked secrets, widened exposure, and so on)
and posts one advisory comment. Built for the
[MulzaTech DOOM on Azure](https://github.com/marcelomusza/mulzatech-doom-azure)
project; see its
[Phase 8 doc](https://github.com/marcelomusza/mulzatech-doom-azure/blob/main/docs/phases/phase-8-ai-infra-reviewer.md)
and [ADR 0007](https://github.com/marcelomusza/mulzatech-doom-azure/blob/main/docs/adr/0007-ai-infra-reviewer-design.md)
for the reasoning.

## How it works

- `rubric.md` is the review criteria. Each item comes from a mistake that
  actually happened, with the reason next to it.
- `review.py` sends a diff plus the rubric to the Anthropic API and prints
  Markdown. Default model is `claude-haiku-5-5` (about a tenth of a cent
  per review); set `REVIEW_MODEL` to change it.
- `.github/workflows/review.yml` is a reusable workflow: it builds the PR
  diff (excluding vendored files), runs the script, and posts the result,
  updating its own earlier comment instead of adding a new one.

## Using it from another repo

Add a workflow like this, and an `ANTHROPIC_API_KEY` repository secret
(create the key with a spend limit):

```yaml
name: Infra review

on:
  pull_request:
    paths:
      - 'infra/**'

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    uses: marcelomusza/mulzatech-infra-reviewer/.github/workflows/review.yml@main
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

The rubric is specific to the MulzaTech project; fork this repo and edit
`rubric.md` for yours.

## Running it locally

```bash
pip install -r requirements.txt
python review.py tests/sample_bad.diff
```

Set `ANTHROPIC_API_KEY` in your environment first (use a masked prompt,
not a command-line argument). `tests/sample_bad.diff` contains seeded
problems plus a prompt-injection attempt; `tests/sample_clean.diff` should
come back as "No issues found".

## Safety properties

- **Comment only.** Permissions are `contents: read` and
  `pull-requests: write`. The PR's code is diffed, never executed.
- **The diff is treated as untrusted data**, wrapped in per-run random
  tags, with text that tries to steer the review reported as a finding.
- **Runs on `pull_request`**, so fork PRs get no secrets (and are
  skipped), rather than `pull_request_target`.
- **No silent truncation.** An oversized diff gets an explicit "a human
  should review this" comment.

## Limitations

It is advisory: it can miss things and is not a required check. It sees
only the diff, not the rest of the repo, and does not run
`terraform plan`. Treat it as a second pair of eyes.
