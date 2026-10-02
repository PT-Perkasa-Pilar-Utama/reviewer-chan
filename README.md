# Reviewer-chan

The shared job behind Reviewer-chan, PT Perkasa Pilar Utama's review bot.
Comment `/reviewer-chan` on a pull request, and the
Reviewer-chan app posts a tech-lead review made by the `lead-review` skill from
[snowfluke/tech-lead-skills](https://github.com/snowfluke/tech-lead-skills).

This repo is public because GitHub lets a public repo call a reusable workflow
only from a public repo. It holds no secrets. The job reads the tech lead's
Claude token (optional), the app's private key and an optional OpenCode key from the calling repo's `reviewer-chan`
environment.

## Call it

Add `.github/workflows/reviewer-chan.yml` to your repo:

```yaml
name: reviewer-chan

on:
  issue_comment:
    types: [created]
  # To review every new pull request too, remove the # from the next two lines.
  # pull_request_target:
  #   types: [opened, ready_for_review]

permissions:
  contents: read
  pull-requests: read
  checks: read
  actions: read

jobs:
  review:
    # A cheap filter, so most comments start no job; the shared job checks the exact command.
    if: >-
      github.event_name == 'pull_request_target' ||
      (github.event.issue.pull_request && startsWith(github.event.comment.body, '/reviewer-chan'))
    uses: PT-Perkasa-Pilar-Utama/reviewer-chan/.github/workflows/review.yml@<the tag's commit SHA> # v2.6.0
    with:
      daily_cap: 10
      tech_leads: your-login # logins separated by spaces or commas
      claude_model: ${{ vars.REVIEWER_CHAN_CLAUDE_MODEL }}
      claude_effort: ${{ vars.REVIEWER_CHAN_CLAUDE_EFFORT }}
      opencode_model: ${{ vars.REVIEWER_CHAN_OPENCODE_MODEL }}
      opencode_effort: ${{ vars.REVIEWER_CHAN_OPENCODE_EFFORT }}
    # A called workflow sees a secret only when it is passed or inherited; the job's
    # reviewer-chan environment then supplies the value.
    secrets: inherit
```

A comment whose first line is `/reviewer-chan` or `/reviewer-chan review` starts a review. Case and
extra spaces do not matter. The PR author with write access, a tech lead in
`tech_leads`, or a repo admin may comment it. Keep the file name
`reviewer-chan.yml`: the daily cap counts runs of it.

The calling repo needs an environment named `reviewer-chan`, limited to the
`main` branch, with these secrets: `REVIEWER_CHAN_PRIVATE_KEY` (required),
`CLAUDE_CODE_OAUTH_TOKEN` (optional) and `OPENCODE_API_KEY` (optional). The job refuses to run if the environment is open
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

The engine (Claude or OpenCode) reads; the workflow writes.

| Step | Runs Claude | Holds |
| --- | --- | --- |
| Prepare: fetch the PR head, write the diff, the PR data, the checks, the comments by writers, and the skeletons | No | A read-only GitHub token |
| Review | Yes | With Claude: the Claude token, and no GitHub token at all. It reads the review folders, writes only to `/tmp/review/out`, and runs only the skill's two scripts, installed read-only. Shell syntax that chains, redirects, expands or substitutes is denied. Its shell commands run with no credentials in their environment. With OpenCode: no secret except the optional `OPENCODE_API_KEY`, and none with a free model. |
| Verify and gate | The verifier: yes, with no tools, in a new session | The Claude token, or the OpenCode key if any. The gates (format, verifier, CI line, scans, no finding dropped) run without Claude. A body fault goes back to the reviewing session to fix, at most twice. Claude never writes a verdict. |
| Post | No | The app token, minted after Claude finished |

- The secrets live in the calling repo's `reviewer-chan` environment, which only `main` can use. A workflow on any other branch cannot read them.
- The job never runs the pull request's code. The gate comes from the required CI checks, which run without secrets.
- It runs only inside PT-Perkasa-Pilar-Utama, and never for a bot's comment. On a bot's or an outsider's PR, only a tech lead or a repo admin can start it.
- Only the PR author with write access, a tech lead in `tech_leads`, or a repo admin can start a review. Anyone else's comment is ignored, and the run summary says why. On a PR by an outsider (a fork, or an author without write access), only a tech lead or an admin can start it, and that comment is the approval. The gate resolves the head commit when the comment arrives, and the run summary names it. If the PR changes before the review starts, nothing is posted.
- The caller's trigger is `issue_comment` (type `created`), which runs the calling file from the default branch, so a PR cannot change it.
- Every comment that starts with `/reviewer-chan` starts a run of the calling workflow, and the gate stops the ones that do not match. The daily cap counts only runs whose review job ran.
- OpenCode: the job downloads OpenCode's release file and checks it against a pinned SHA-256 digest. Its config comes only from the job, so the repo's own OpenCode config, plugins and `.claude` files are ignored. It has no web tools. It has no env scrub like Claude's.
- Comments reach the engine only from people with write access.
- The rules come from the base branch. The workflow picks the checklist, not Claude.
- The skill's scripts refuse files outside the review folders and inside any `.git` folder, by real path.
- Each request gets an eyes reaction and one status comment, which changes to the outcome: the review's link, why nothing was posted, or that a newer request replaced it.
- The review posts as a comment or a request for changes. It never approves. A round with nothing open dismisses Reviewer-chan's earlier requests for changes, so it does not keep the PR blocked.
- Every run uploads its drafts, verdicts and gate results as an artifact for 7 days. A file holding a credential-shaped or long encoded string is withheld from it.
- The posted body has no images, raw HTML, mentions, or links other than the repo's own files at a pinned commit. A failed check posts a fixed sentence, never text from the PR or from Claude.
- `main` takes changes only through a reviewed pull request, and each calling repo pins a tagged commit.

What this does not cover: a crafted pull request can still try to steer the
review itself, for example toward fewer findings. The human approval that
branch protection requires is the guard against that. Never remove it because
a bot reviews.
