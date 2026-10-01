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

Claude reads; the workflow writes.

| Step | Runs Claude | Holds |
| --- | --- | --- |
| Prepare: fetch the PR head, write the diff, the PR data, the checks and the members' comments | No | A read-only GitHub token |
| Review | Yes | The Claude token. No git, no gh, no GitHub write token. It reads the review folders, writes only to `/tmp/review/out`, and runs only the skill's two scripts, installed read-only. Pipes, redirects and command substitution are denied. |
| Post: check the body, scan it for credentials, post it | No | The app token, minted after Claude finished |

- The job never runs the pull request's code. The gate comes from the required CI checks, which run without secrets.
- It runs only inside PT-Perkasa-Pilar-Utama, and never for a bot's PR.
- A PR from a fork, or by an author who is not an owner, member or collaborator, waits for a maintainer's approval in the `reviewer-chan-outside` environment before it uses any secret or Claude usage. The job refuses if that environment has no required reviewers.
- Comments reach Claude only from owners, members and collaborators. On a public repo, anyone else's comment is dropped.
- The skill's scripts refuse files outside the review folders, by real path, so `..` and symlinks cannot reach secrets.
- The app's private key reaches only the two steps that mint a token, and neither runs Claude.
- `main` takes changes only through a reviewed pull request, and each calling repo pins a tagged commit.

What this does not cover: a crafted pull request can still try to steer the review itself, for example toward approving bad code. The human approval that branch protection requires is the guard against that. Never remove it because a bot reviews.
