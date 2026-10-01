# Reviewer-chan

The shared job behind Reviewer-chan, PT Perkasa Pilar Utama's review bot.
Request a review from the `reviewer-chan` team on a pull request, and the
Reviewer-chan app posts a tech-lead review made by the `lead-review` skill from
[snowfluke/tech-lead-skills](https://github.com/snowfluke/tech-lead-skills).

This repo is public because GitHub lets a public repo call a reusable workflow
only from a public repo. It holds no secrets: each calling repo passes its tech
lead's Claude token and the app's private key in.

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
    uses: PT-Perkasa-Pilar-Utama/reviewer-chan/.github/workflows/review.yml@3a8ccc595456afa9638b65dfe781d8ad49e4d29f # v1.0.0
    with:
      daily_cap: 10
    secrets:
      claude_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN_CHANGEME }}
      reviewer_chan_key: ${{ secrets.REVIEWER_CHAN_PRIVATE_KEY }}
```

Pin the job by commit, as above. Your repo's Dependabot (the `github-actions`
ecosystem) then proposes each new tag as a pull request, so a change here reaches
your repo only after someone reviews it.

Org members: the full setup guide, with the admin steps, is in
PT-Perkasa-Pilar-Utama/review-chan-setup.

## Release

Merge the change to `main` through a pull request, then tag it: `vMAJOR.MINOR.PATCH`.
Dependabot in each calling repo picks up the tag.

## Safety

The job never runs the pull request's code. It checks out the base commit,
reads the PR's files as text, takes the gate from the required CI checks, and
lets Claude run only the read-only commands in its `--allowedTools`. `main`
takes changes only through a reviewed pull request, and each calling repo pins
a tagged commit.
