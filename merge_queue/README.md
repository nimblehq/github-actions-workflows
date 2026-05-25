# merge-queue

A simple "labeled" merge queue for GitHub Actions, alternative to [GitHub's native merge queue](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue).

No GitHub Enterprise required. No external services. Just a workflow.

## What it does

Label a PR with `mergeme` and merge-queue takes over:

1. **Rebases** the PR onto the latest base branch
2. **Waits** for all status checks to pass on the rebased code
3. **Squash merges** into the base branch and deletes the source branch
4. **Dequeues** with a PR comment explaining what went wrong if anything fails

Multiple PRs labeled at the same time? They all get processed. When a job starts, it scans for every open PR with the label and works through them in order — so even if several PRs are labeled while one is already being processed, they'll all be picked up by the time the queue drains.

## Quick start

Create `.github/workflows/merge-queue.yml` in your repo:

```yaml
name: Merge Queue

on:
  pull_request:
    types: [labeled]

concurrency:
  group: merge-queue
  cancel-in-progress: false

jobs:
  merge:
    if: github.event.label.name == 'mergeme'
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      checks: read
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: nimblehq/github-actions-workflows/merge_queue@0.2.0
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

That's it. Label a PR with `mergeme` to try it out.

## How the queue works

When triggered, the action scans for all open PRs carrying the label and processes them serially — oldest first. The `concurrency` block ensures only one job runs at a time, so a second trigger that arrives while the first job is busy will wait and then drain whatever remains in the queue when it starts.

```
PR #1 labeled  ─┐
PR #2 labeled  ─┤──>  job picks up #1, #2, #3 in order  ──>  all merged
PR #3 labeled  ─┘
```

Each PR rebases onto the base branch at the moment it's processed, which includes all previously merged PRs. This is the key property of a merge queue — no PR merges without being tested against the current state of the target branch.

## What happens on failure

merge-queue removes the `mergeme` label and leaves a comment on the PR explaining what went wrong:

| Failure                  | What merge-queue does                                |
| ------------------------ | ---------------------------------------------------- |
| Merge conflicts          | Removes from queue, comments to resolve conflicts    |
| Rebase fails             | Removes from queue, comments to rebase manually      |
| Status checks fail       | Removes from queue, comments to fix and re-label     |
| Timeout (default 30 min) | Removes from queue, comments to re-label             |
| Squash merge rejected    | Removes from queue, comments about branch protection |

To retry, fix the issue and add the `mergeme` label again.

## Inputs

| Input           | Description                                        | Default               |
| --------------- | -------------------------------------------------- | --------------------- |
| `github-token`  | GitHub token with write access to contents and PRs | `${{ github.token }}` |
| `label`         | Label that adds a PR to the queue                  | `mergeme`             |
| `poll-interval` | Seconds between status check polls                 | `30`                  |
| `timeout`       | Max seconds to wait for status checks              | `1800`                |

## Outputs

| Output   | Description                                                        |
| -------- | ------------------------------------------------------------------ |
| `result` | `merged` or `failed` — reflects the last PR processed in the queue |

## Custom label

```yaml
- uses: nimblehq/github-actions-workflows/merge_queue@0.2.0
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    label: "ready-to-merge"
```

Update the `if` condition in the workflow to match:

```yaml
if: github.event.label.name == 'ready-to-merge'
```

## Token permissions

The default `GITHUB_TOKEN` works for repos without branch protection. If you have branch protection rules that restrict who can push or merge, you'll need a [GitHub App token](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app) or [fine-grained PAT](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#fine-grained-personal-access-tokens) with:

- **Contents**: read and write (for rebase push)
- **Pull requests**: read and write (for merge, labels, comments)
- **Checks**: read (for polling CI status)

## Job timeout

GitHub Actions jobs have a default timeout of 6 hours. If your queue is deep and CI is slow, the job may be cancelled mid-way. Increase the job timeout in your workflow if needed:

```yaml
jobs:
  merge:
    timeout-minutes: 720 # 12 hours
```

PRs that weren't reached before the timeout will keep their label and be picked up by the next trigger.

## Why not GitHub's built-in merge queue?

GitHub has a [native merge queue](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue), but it requires a GitHub Enterprise plan or public repos on Team/Free plans with specific configurations. `merge-queue` gives you the same core behavior on any plan:

- Serialized merges tested against the latest base branch
- Automatic rebase before testing
- Clear feedback on failures

And since it's just a workflow file and a composite action, you can read and modify every line of it.

## License

> Inspired by [opper-ai/pr-gatory-action](https://github.com/opper-ai/pr-gatory-action)

MIT
