# Agent pipeline: issues to pull requests

HappyCynic repositories can hand feature work to Claude through GitHub issues. The process
lives here, in the composite action [`actions/agent-sdlc`](../actions/agent-sdlc/action.yml);
each repository opts in with a small workflow and keeps its own project rules in `CLAUDE.md`
and `openspec/`.

## How it works

1. A maintainer opens an issue describing the feature (problem, acceptance criteria, out of
   scope).
2. The maintainer adds the label **`agent:propose`**. Claude, acting as `claude[bot]`, writes an
   [OpenSpec](https://github.com/Fission-AI/OpenSpec) change for the issue (proposal, tasks,
   spec deltas) on a branch `agent/issue-<n>-<slug>`, opens a **draft** pull request, comments
   on the issue, and stops.
3. The maintainer reviews the proposal. To change it, comment on the PR and add
   `agent:propose` again, or edit it directly.
4. The maintainer adds **`agent:implement`**. Claude implements the tasks in the same PR (tests
   first), runs the repository's checks, archives the change so `openspec/specs/` is updated,
   and marks the PR ready for review.
5. The maintainer reviews, approves and squash-merges. Releases stay with the maintainer.

The workflow removes the trigger label when a stage ends, so adding it again re-runs that
stage.

## Guard rails

| Risk | Control |
|---|---|
| Strangers steering the agent through issues on public repositories | Only people with write access can add labels, and the action refuses to run for actors without write access. Issue text is never interpolated into the prompt; Claude reads it with `gh issue view` and is told to treat it as data. |
| Agent merging, approving, releasing or relabelling | Not in its allowed tools (`gh pr merge`, `gh pr review`, `gh api`, `gh release`, `gh label`, `git tag` are disallowed); repository rulesets block pushes to the default branch and `v*` tags; required approval from code owners. |
| The Claude token leaking through a workflow on another branch | The token is an **environment** secret in environment `agent`, which only the default branch may use. The caller's job selects that environment and passes the token to the action; a workflow edited on a branch cannot read it. (A reusable workflow cannot read a caller repository's environment secrets, which is why this is a composite action.) |
| Changes to the pipeline itself | This repository is protected like the others; callers pin the action by commit SHA, so a change here takes effect in a repository only when its pin is bumped (Dependabot does that as a reviewed PR). |

## One-time setup for the organization

1. Install the [Claude GitHub App](https://github.com/apps/claude) on HappyCynic with **Only
   select repositories**.
2. Create a Claude subscription token on a trusted machine: `claude setup-token`. Store it in
   your password manager; you will add it to each repository's `agent` environment. It uses
   the subscription's usage and expires after about a year; rotate it before then.

## Adding a repository

1. **Claude app:** add the repository to the app's selected repositories.
2. **Environment and secret** (Settings → Environments → New environment):
   - Name `agent`. Deployment branches and tags: **Selected branches**, add the default branch
     (e.g. `main`).
   - Environment secret `CLAUDE_CODE_OAUTH_TOKEN`: the token from setup step 2.

   With the GitHub CLI:

   ```sh
   repo=HappyCynic/<repo>
   gh api -X PUT "repos/$repo/environments/agent" \
     -F 'deployment_branch_policy[protected_branches]=false' \
     -F 'deployment_branch_policy[custom_branch_policies]=true'
   gh api -X POST "repos/$repo/environments/agent/deployment-branch-policies" -f name=main -f type=branch
   gh secret set CLAUDE_CODE_OAUTH_TOKEN --env agent --repo "$repo"
   ```
3. **Labels:**

   ```sh
   gh label create agent:propose --repo "$repo" --color 5319E7 --description "Agent writes a proposal, then stops for review"
   gh label create agent:implement --repo "$repo" --color 0E8A16 --description "Agent implements the approved proposal"
   ```
4. **Project rules:** a `CLAUDE.md` with the project's hard rules, setup and checks, and an
   "Issue workflow" section; OpenSpec initialised with `openspec init --tools claude` at the
   version the caller pins, with project context in `openspec/config.yaml`; baseline specs in
   `openspec/specs/`. `CODEOWNERS` should require the maintainer's review for `CLAUDE.md`,
   `.claude/`, `openspec/config.yaml` and `.github/`.
5. **Rulesets:** default branch with no bypass, pull request with one approval from code
   owners, required CI checks, signed commits, squash merging only; a tag ruleset for `v*`.
6. **Caller workflow** at `.github/workflows/agent.yml`:

   ```yaml
   name: agent
   on:
     issues:
       types: [labeled]
   permissions: {}
   jobs:
     stage:
       if: github.event.label.name == 'agent:propose' || github.event.label.name == 'agent:implement'
       runs-on: ubuntu-latest
       environment: agent # holds CLAUDE_CODE_OAUTH_TOKEN; only the default branch may use it
       timeout-minutes: 60
       concurrency: # one stage per issue at a time
         group: agent-${{ github.event.issue.number }}
         cancel-in-progress: false
       permissions:
         contents: read # the agent pushes with the Claude app's token, not this one
         issues: write # clear the trigger label afterwards
         id-token: write # Claude GitHub App token exchange
       steps:
         - uses: HappyCynic/.github/actions/agent-sdlc@<commit-sha> # pin; Dependabot bumps it
           with:
             claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
             openspec_version: 1.13.2 # must match CLAUDE.md
             setup_uv: true # Python + uv projects
             setup: uv python install 3.14 && uv sync --frozen --all-groups
             allowed_tools: Bash(uv:*)
   ```
7. **First run:** open a small issue, add `agent:propose`, and check that the draft PR, the
   issue comment and the commits come from `claude[bot]`. The first workflow run by
   `claude[bot]` on a pull request may need approval under Actions settings for first-time
   contributors.

## Rotating the Claude token

The token lives in each repository's `agent` environment (an organization secret would be
readable by workflows on any branch). Create a new one with `claude setup-token`, then:

```sh
read -rs TOKEN   # paste the new token
for repo in pv-icloud; do   # every repository that uses the pipeline
  printf %s "$TOKEN" | gh secret set CLAUDE_CODE_OAUTH_TOKEN --env agent --repo "HappyCynic/$repo"
done
unset TOKEN
```

## Changing the pipeline

Edit `actions/agent-sdlc/action.yml` here through a pull request, then bump the pinned SHA in each repository
(Dependabot's `github-actions` updates do this). Keep `anthropics/claude-code-action` and the
other actions pinned to commit SHAs, and prefer releases at least two weeks old.
