# Reviewer-chan

The shared job behind Reviewer-chan, PT Perkasa Pilar Utama's review bot.
Request a review from the `reviewer-chan` team on a pull request, and the
Reviewer-chan app posts a tech-lead review made by the `lead-review` skill from
[snowfluke/tech-lead-skills](https://github.com/snowfluke/tech-lead-skills).

This repo is public because GitHub lets a public repo call a reusable workflow
only from a public repo. It holds no secrets. The job reads the tech lead's
Claude token and the app's private key from the calling repo's `reviewer-chan`
environment.

## Call it

Add `.github/workflows/reviewer-chan.yml` to your repo:

```yaml
name: reviewer-chan

on:
  pull_request_target:
    types: [review_requested]

permissions:
  contents: read
  pull-requests: read
  checks: read
  actions: read

jobs:
  review:
    uses: PT-Perkasa-Pilar-Utama/reviewer-chan/.github/workflows/review.yml@<the tag's commit SHA> # v1.3.1
    with:
      daily_cap: 10
      also_on_reviewer: your-login # optional: a review request to you also starts Reviewer-chan
    # A called workflow sees a secret only when it is passed or inherited; the job's
    # reviewer-chan environment then supplies the value.
    secrets: inherit
```

The calling repo needs an environment named `reviewer-chan`, limited to the
`main` branch, with two secrets: `CLAUDE_CODE_OAUTH_TOKEN` and
`REVIEWER_CHAN_PRIVATE_KEY`. The job refuses to run if the environment is open
to every branch. GitHub Free offers environment secrets on public repos only.

Pin the job by commit, as above. Your repo's Dependabot (the `github-actions`
ecosystem) then proposes each new tag as a pull request, so a change here reaches
your repo only after someone reviews it.

Org members: the full setup guide, with the admin steps, is in
PT-Perkasa-Pilar-Utama/reviewer-chan-setup.

## Release

Merge the change to `main` through a pull request, then tag it: `vMAJOR.MINOR.PATCH`.
Dependabot in each calling repo picks up the tag.

## Safety

Claude reads; the workflow writes.

| Step | Runs Claude | Holds |
| --- | --- | --- |
| Prepare: fetch the PR head, write the diff, the PR data, the checks, the comments by writers, and the skeletons | No | A read-only GitHub token |
| Review | Yes | The Claude token. No git, no gh, no GitHub write token. It reads the review folders, writes only to `/tmp/review/out`, and runs only the skill's two scripts, installed read-only. Shell syntax that chains, redirects, expands or substitutes is denied. Its shell commands run with no credentials in their environment. |
| Verify | Yes, with no tools, in a new session | The Claude token. The reviewing session never sees its answer. |
| Post: check the body, scan it, post it | No | The app token, minted after Claude finished |

- Both secrets live in the calling repo's `reviewer-chan` environment, which only `main` can use. A workflow on any other branch cannot read them.
- The job never runs the pull request's code. The gate comes from the required CI checks, which run without secrets.
- It runs only inside PT-Perkasa-Pilar-Utama, and never for a bot's PR.
- A PR from a fork, or by an author without write access, waits for a maintainer's approval in the `reviewer-chan-outside` environment. The job refuses if that environment has no required reviewers. The approval covers the head commit the run names; if the PR changes before the review starts, nothing is posted.
- Comments reach Claude only from people with write access.
- The rules come from the base branch. The workflow picks the checklist, not Claude.
- The skill's scripts refuse files outside the review folders and inside any `.git` folder, by real path.
- The review posts as a comment or a request for changes. It never approves.
- The posted body has no images, raw HTML, mentions, or links other than the repo's own files at a pinned commit. A failed check posts a fixed sentence, never text from the PR or from Claude.
- `main` takes changes only through a reviewed pull request, and each calling repo pins a tagged commit.

What this does not cover: a crafted pull request can still try to steer the
review itself, for example toward fewer findings. The human approval that
branch protection requires is the guard against that. Never remove it because
a bot reviews.
