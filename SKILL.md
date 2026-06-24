---
name: dependabot-automerge-skill
description: Set up or repair GitHub Dependabot auto-merge for a repository. Use when the user mentions dependabot, auto-merge, dependabot PR stuck, semver-major merging, GitHub Actions PRs not auto-merging, branch protection required checks, allow_auto_merge disabled, or wants to reduce manual PR churn. Triggers on phrases like "set up dependabot auto-merge", "PR stuck waiting for checks", "auto-merge not waiting for CI", "major version dependabot", "branch protection required status check", "PR stuck BEHIND", "auto-merge returns 422", "oauth app cannot create workflow", "auto-merge workflow never runs", "app/dependabot vs dependabot[bot]". Does NOT use when the user only wants to configure dependabot.yml update schedule, or only wants to enable dependabot security updates without auto-merge.
---

# Dependabot Auto-Merge Skill

This skill configures a GitHub repository so Dependabot PRs are merged automatically after CI passes, while still leaving breaking-change PRs (e.g. maven major bumps) for human review.

It is built from a real incident post-mortem. The strategy, the gotchas, and the verification steps below are all things that have been observed to fail in production. Do not skip the verification section.

---

## When to use

Use this skill when the user says any of:

- "set up dependabot auto-merge"
- "dependabot PR is stuck / not merging"
- "auto-merge merged without running CI"
- "the required check name doesn't match"
- "I want major version updates for github-actions to auto-merge"
- "branch protection / required status check"
- "I keep getting dependabot PRs to click through manually"
- "PR is stuck at `BEHIND`"
- "`gh pr merge --auto` returns 422 / "Auto merge is not allowed""
- "auto-merge says 'OAuth App cannot create or update workflow'"
- "I changed auto-merge.yml and nothing happened"
- "dependabot is open but not being merged"
- "auto-merge workflow runs but does nothing / always skipped"
- "Dependabot PRs are bot authored but my workflow never fires"

Do **not** use this skill for:

- Just editing `.github/dependabot.yml` schedule/ecosystem config (no auto-merge involved).
- Dependabot security updates without auto-merge.
- Non-Dependabot PR auto-merge (e.g. `bors`, merge queues, custom bots). Different problem.

---

## Pre-flight checks

Before changing anything, gather context. Do not skip these.

1. **Read the current state**:
   - `.github/dependabot.yml` — which ecosystems are configured? schedule? PR limit? groups? labels?
   - `.github/workflows/` — is there already an `auto-merge.yml`? what is its `if:` line?
   - If `build.yml` exists, what is the `on:` block? Does it include `pull_request`?
2. **Check the GitHub repo** (use `gh` CLI; the user must be authenticated):
   ```bash
   gh auth status
   gh api repos/<owner>/<repo>/branches/master/protection
   gh api repos/<owner>/<repo> --jq '.allow_auto_merge'   # see Pitfall 6
   ```
3. **Check if there are stuck PRs** that motivated the request:
   ```bash
   gh pr list --state open --json number,title,headRefName,mergeStateStatus,author
   ```
   **Also note the `author.login`** — GitHub migrated Dependabot in 2024. New PRs are authored by `app/dependabot`; legacy PRs (and any in repos that haven't migrated) are authored by `dependabot[bot]`. Your auto-merge workflow's `if:` line must match the one the repo actually uses — see Pitfall 8.
4. **Read the actual check names** that GitHub is generating, not what you assume. See Pitfall 3.
5. **Inventory available PAT secrets**:
   ```bash
   gh secret list
   ```
   You need at least one PAT with `repo` + `workflow` scope for auto-merge to work on PRs that modify `.github/workflows/*.yml`. `GITHUB_TOKEN` is not sufficient — see Pitfall 5. If none exists, ask the user to create one.

If the repo does not have `gh` auth, stop and ask the user to log in. Do not guess at branch protection.

---

## Strategy

There are six moving parts. All are required.

| # | File / setting | Purpose |
| --- | --- | --- |
| 1 | `.github/workflows/auto-merge.yml` | Approve and enable auto-merge on qualifying Dependabot PRs |
| 2 | `.github/workflows/build.yml` | Run CI on `pull_request` so required checks appear on the PR |
| 3 | `allow_auto_merge` repo setting | Precondition for `gh pr merge --auto` — see Pitfall 6 |
| 4 | A PAT secret (`repo` + `workflow` scope) | Needed because `GITHUB_TOKEN` cannot enable auto-merge on PRs that modify `.github/workflows/*.yml` — see Pitfall 5 |
| 5 | Branch protection on `master` | Mark the CI check as required, so `gh pr merge --auto` actually waits |
| 6 | `dependabot.yml` (recommended) | Grouping + weekly schedule + reasonable PR limit + labels are what turn "auto-merge works" into "PR noise goes to zero". See Step 6. |

### Decision table for what to merge

The auto-merge workflow decides per-PR. Default policy:

| `update-type` | head ref prefix | Action |
| --- | --- | --- |
| `semver-patch` | any | auto-merge |
| `semver-minor` | any | auto-merge |
| `semver-major` | `dependabot/github_actions/*` | auto-merge |
| `semver-major` | other (e.g. `dependabot/maven/*`) | leave for human review |
| anything else (digest, indirect, etc.) | any | leave for human review |

Rationale: GitHub Actions major versions are usually safe (just Node runtime bumps); Java/Maven majors can break the build. Adapt this table to the user's repo — e.g. for a docs-only repo, merging all majors is fine.

---

## Implementation

Apply the changes in this order. Do not push or run any GitHub API call until each step is reviewed.

### Step 1 — `auto-merge.yml`

Create `.github/workflows/auto-merge.yml`. **Use a PAT secret** (`MYTOKEN` below) for the merge and approve steps — `GITHUB_TOKEN` is not sufficient for PRs that modify `.github/workflows/*.yml` (see Pitfall 5). `fetch-metadata` is read-only and can use `GITHUB_TOKEN`.

```yaml
name: Dependabot auto-merge

on:
  pull_request:

permissions:
  pull-requests: write
  contents: write

jobs:
  dependabot:
    if: |
      github.event.pull_request.user.login == 'dependabot[bot]' ||
      github.event.pull_request.user.login == 'app/dependabot'
    runs-on: ubuntu-latest
    steps:
      - name: Dependabot metadata
        id: meta
        uses: dependabot/fetch-metadata@v2
        with:
          github-token: "${{ secrets.GITHUB_TOKEN }}"

      - name: Check if auto-merge applicable
        id: check
        run: |
          TYPE="${{ steps.meta.outputs.update-type }}"
          REF="${{ github.event.pull_request.head.ref }}"
          if [[ "$TYPE" == "version-update:semver-patch" ]] || \
             [[ "$TYPE" == "version-update:semver-minor" ]] || \
             { [[ "$TYPE" == "version-update:semver-major" ]] && [[ "$REF" == dependabot/github_actions/* ]]; }; then
            echo "should_merge=true" >> "$GITHUB_OUTPUT"
          else
            echo "should_merge=false" >> "$GITHUB_OUTPUT"
          fi

      - name: Approve dependabot PR
        if: steps.check.outputs.should_merge == 'true'
        run: gh pr review --approve "$PR_URL"
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GH_TOKEN: ${{ secrets.MYTOKEN }}   # PAT with repo + workflow scope

      - name: Enable auto-merge
        if: steps.check.outputs.should_merge == 'true'
        run: gh pr merge --auto --rebase "$PR_URL"
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GH_TOKEN: ${{ secrets.MYTOKEN }}   # PAT with repo + workflow scope
```

Adapt the third `||` branch to match the policy table above. For a Maven-only repo, drop the major-merge clause entirely.

**Secret name is case-sensitive** — `${{ secrets.mytoken }}` (lowercase) and `${{ secrets.MYTOKEN }}` (uppercase) are different secrets. Verify with `gh secret list`.

### Step 2 — `build.yml` (or whatever the CI workflow is named)

Make sure it runs on `pull_request` **and** scope `push` to `master` to avoid double runs:

```yaml
on:
  push:
    branches: [ master ]
  pull_request:
```

Do **not** write:

```yaml
on:
  push:
  pull_request:    # WRONG: every PR commit triggers both events
```

If the build uses a `matrix`, the actual check name will include matrix dimensions (e.g. `build (ubuntu-latest, 17, false)`). You will need this exact string in Step 4.

### Step 3 — `allow_auto_merge` (repo setting)

`gh pr merge --auto` requires the repo-level `allow_auto_merge` flag. This is a separate setting from branch protection and is **off by default** on many repos. Enable it:

```bash
gh api -X PATCH repos/<owner>/<repo> \
  -H "Accept: application/vnd.github+json" \
  -f allow_auto_merge=true
```

Verify:

```bash
gh api repos/<owner>/<repo> --jq '.allow_auto_merge'
# should print: true
```

If this is off, every `gh pr merge --auto` will silently 422. See Pitfall 6.

### Step 4 — Branch protection

Get the actual check names from a recent PR, then set required checks. Never guess the name.

```bash
# Find the real check names
gh api graphql -F query='
query {
  repository(owner: "<owner>", name: "<repo>") {
    pullRequest(number: <N>) {
      statusCheckRollup {
        contexts(first: 20) {
          nodes {
            ... on CheckRun {
              name
              isRequired(pullRequestNumber: <N>)
            }
          }
        }
      }
    }
  }
}'
```

Then set the protection (example for the matrix above):

```bash
gh api -X PUT repos/<owner>/<repo>/branches/master/protection \
  -H "Accept: application/vnd.github+json" \
  --input - <<'EOF'
{
  "required_status_checks": {
    "strict": true,
    "checks": [
      { "context": "build (ubuntu-latest, 17, false)" },
      { "context": "build (windows-latest, 17, false)" }
    ]
  },
  "enforce_admins": false,
  "required_pull_request_reviews": null,
  "restrictions": null
}
EOF
```

- `strict: true` — PR must be rebased onto master before merge.
- `enforce_admins: false` — let admins bypass (useful for hot-fixes).
- `required_pull_request_reviews: null` — auto-merge should not be gated by human review.

### Step 5 — Commit and push

```bash
git add .github/workflows/auto-merge.yml .github/workflows/build.yml
git commit -m "ci: dependabot auto-merge with required CI gate"
```

`git push` may be rejected because the remote master has moved while you were editing. Rebase, then push. The `Bypassed rule violations` message during push is expected when `enforce_admins: false`.

**If `git push` is rejected with "OAuth App to create or update workflow ... without `workflow` scope"**, your gh CLI token lacks the `workflow` scope. The simplest fix is to switch the remote URL from HTTPS to SSH:

```bash
git remote set-url origin git@github.com:<owner>/<repo>.git
git push
```

SSH uses your local SSH key and is not subject to the `workflow` OAuth scope restriction.

### Step 6 — `dependabot.yml` (do this, not optional)

Auto-merge works without touching `dependabot.yml`, but the user will still drown in noise unless you also do these four things. Each one collapses PR count or clarifies intent:

1. **Weekly schedule, not daily.** `interval: daily` produces a PR per bump per day; `weekly` produces one batched bump on a predictable day.
2. **`open-pull-requests-limit` of 5–20.** Default is 5. `100` is a foot-gun — see Snags.
3. **Groups** so N updates → 1 PR instead of N PRs. This also avoids the Pitfall 7 BEHIND race almost entirely.
4. **`commit-message.prefix` and `labels`** so the auto-merge workflow (and humans) can identify dependabot PRs at a glance.

Concrete recipe (adapt `directory`, `target-branch`, and ecosystem-specific bits):

```yaml
version: 2
updates:
  - package-ecosystem: "maven"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "04:00"
      timezone: "Asia/Shanghai"   # or your team TZ
    target-branch: "master"
    open-pull-requests-limit: 10
    commit-message:
      prefix: "build(deps)"
      prefix-development: "build(deps-dev)"
      include: "scope"
    labels:
      - "dependencies"
      - "java"
    groups:
      maven-minor-and-patch:
        applies-to: version-updates
        patterns: ["*"]
        update-types: ["minor", "patch"]
      maven-major:
        applies-to: version-updates
        patterns: ["*"]
        update-types: ["major"]

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "04:00"
      timezone: "Asia/Shanghai"
    target-branch: "master"
    open-pull-requests-limit: 5
    commit-message:
      prefix: "build(deps)"
      include: "scope"
    labels:
      - "dependencies"
      - "github-actions"
    groups:
      github-actions:
        applies-to: version-updates
        patterns: ["*"]
```

Why this works:

- **Two maven groups** let you keep your auto-merge policy table simple — one PR per group per cycle, and your policy can match the group name instead of the per-PR `update-type` if you want.
- **`applies-to: version-updates`** scopes groups to version bumps only — security updates stay in their own PRs (you usually want those reviewed individually).
- **Same `day: monday` for all ecosystems** means the user sees one batch of dependabot activity per week, not a constant drip.

---

## Pitfalls (read these before declaring success)

These are the eight things that have actually broken in production. Re-check each one before finishing.

### Pitfall 1 — Major version updates never merge

**Symptom**: `actions/checkout 6→7` style PRs sit open forever.

**Cause**: Auto-merge workflow only matches `semver-patch` / `semver-minor`.

**Fix**: Add `semver-major` to the `if` chain, gated by head ref prefix (see policy table). The head ref for Dependabot is always `dependabot/<ecosystem>/<branch>-<dep>-<version>`, so the ecosystem name is a reliable discriminator.

### Pitfall 2 — Auto-merge skips CI

**Symptom**: PR is merged before any CI runs at all.

**Cause**: `build.yml` only triggers on `push`, not on `pull_request`. No required check exists on the PR, so `gh pr merge --auto` merges immediately.

**Fix**: Add `pull_request:` to `on:` in the CI workflow AND set up branch protection with the resulting check marked required. Both are needed; one alone does not help.

### Pitfall 3 — Required check name does not match

**Symptom**: CI is green, auto-merge is enabled, but `mergeStateStatus: "BLOCKED"`. PR never merges.

**Cause**: Branch protection has `Java CI / build` (workflow name + job name), but the actual GitHub Actions check name is the job name with matrix dimensions, like `build (ubuntu-latest, 17, false)`. The two do not match.

**Diagnostic**: Run the GraphQL query in Step 4 and look for `isRequired: false` on a check you expected to be required. That is the smoking gun.

**Fix**: Use the actual name(s) shown by GraphQL. For matrix jobs you need one entry per matrix axis value — there is no wildcard.

### Pitfall 4 — CI runs twice per PR

**Symptom**: A single dependabot PR shows four CI runs (push + pull_request × matrix axes). Runner minutes wasted.

**Cause**: `on: { push:, pull_request: }` triggers both events when dependabot pushes to its PR branch.

**Fix**: Scope `push` to the base branch: `push: { branches: [ master ] }`. PR commits only fire `pull_request`; direct pushes to master only fire `push`. No overlap.

### Pitfall 5 — `GITHUB_TOKEN` cannot enable auto-merge on workflow PRs

**Symptom**: Auto-merge workflow logs `GraphQL: Pull request refusing to allow a GitHub App to create or update workflow '.github/workflows/<file>.yml' without 'workflows' permission (enablePullRequestAutoMerge)`. Exit code 1, the merge step fails. The Approve step usually succeeds first.

**Cause**: GitHub restricts `GITHUB_TOKEN` on `pull_request` events from writing to `.github/workflows/**`. Any PR that bumps an action version (i.e. virtually all `dependabot/github_actions/*` PRs) hits this.

**Fix**: Use a repo PAT with `repo` + `workflow` scope for the `gh pr review` and `gh pr merge` steps. `fetch-metadata` is read-only and can keep using `GITHUB_TOKEN`. Configure via `gh secret set MYTOKEN` (use whatever name you want; remember the case).

**Diagnostic**: If you're not sure which token the action is using, check the secret name. `${{ secrets.mytoken }}` (lowercase) and `${{ secrets.MYTOKEN }}` (uppercase) are different secrets — names are case-sensitive. An undefined secret silently falls back to `GITHUB_TOKEN` and produces this exact error.

### Pitfall 6 — `allow_auto_merge` is off at the repo level

**Symptom**: `gh pr merge --auto` returns 422 / "Auto merge is not allowed for this repository". The auto-merge workflow succeeds (no error in the step), but the PR's `autoMergeRequest` stays null.

**Cause**: The `allow_auto_merge` repo setting is independent of branch protection and is **off by default**. Some repos turn it off explicitly.

**Fix**: Enable it before doing anything else:

```bash
gh api -X PATCH repos/<owner>/<repo> -f allow_auto_merge=true
```

**Diagnostic**: `gh api repos/<owner>/<repo> --jq '.allow_auto_merge'` — should print `true`. If it prints `false` or `null`, that's your problem.

**Diagnostic from a workflow run log**: open any past run of `auto-merge.yml` and look at the step list. If `Enable auto-merge` (or whatever you named it) shows `conclusion: skipped`, this is almost always the reason — the `gh pr merge --auto` call short-circuited because the repo flag was off. A successful-but-`skipped` step is the smoking gun; a `failure` step is usually Pitfall 5 (token scope). To inspect step conclusions:

```bash
gh run list --workflow=auto-merge.yml --limit 1 --json databaseId --jq '.[0].databaseId' | \
  xargs -I {} gh api repos/<owner>/<repo>/actions/runs/{}/jobs | \
  python3 -c "import sys,json;[print(s['name'], s['conclusion']) for j in json.load(sys.stdin)['jobs'] for s in j['steps']]"
```

### Pitfall 7 — `strict: true` + rapid dependabot rebase race

**Symptom**: After setting everything up correctly, one or more Dependabot PRs sit at `mergeStateStatus: BEHIND` indefinitely. The auto-merge workflow ran and `should_merge` was `true`, but the merge never happens.

**Cause**: When many Dependabot PRs are open at once, they all want to be the "first" to merge. Each one's auto-merge cycle looks like:

1. dependabot rebases PR onto master at SHA X
2. CI passes, auto-merge waits
3. Another PR merges first, master moves to X+1
4. The first PR is now "1 commit behind" and `strict: true` blocks it
5. dependabot's auto-rebase runs **once an hour**, not instantly
6. In the meantime, more PRs merge, the gap widens

Eventually one PR wins the race (lowest rebase count + no contention), but the rest can sit for hours.

**Fix options** (pick one, do not combine):

- **Trigger rebase manually for all open Dependabot PRs** in one batch, with the `gh` script in the Quick Reference. Each round collapses the backlog. Repeat until all PRs are CLEAN.
- **Switch to `strict: false`** — auto-merge handles rebase as part of the merge. Tradeoff: stale PRs merge through, but you lose protection against out-of-date merges.
- **Sequential processing** — disable auto-merge for the lower-priority PRs, merge the most important one first, then re-enable auto-merge. Slow but predictable.
- **Group updates in dependabot.yml** — a single grouped PR (e.g. "maven-minor-and-patch" with N updates) collapses N PRs into 1, eliminating the race almost entirely.

**Do not** keep refreshing the PR page hoping it will clear. The state is genuinely stuck until something pushes master or dependabot re-rebases.

### Pitfall 8 — Dependabot login mismatch: `app/dependabot` vs `dependabot[bot]`

**Symptom**: Dependabot PRs are open, the workflow file exists, but `auto-merge` never runs on any of them. The Actions tab shows zero runs of `auto-merge.yml` for the open Dependabot PRs (or runs only on PRs that pre-date the login migration). The `if:` condition evaluates to false silently.

**Cause**: GitHub migrated Dependabot from a legacy bot to a GitHub App in 2024. The login on the PR's author field changed:

| Bot era | `pull_request.user.login` |
|---|---|
| Pre-2024 (legacy) | `dependabot[bot]` |
| 2024+ (GitHub App) | `app/dependabot` |

A workflow that gates on `if: github.event.pull_request.user.login == 'dependabot[bot]'` will **never fire** on PRs opened by the new Dependabot, and vice versa. The repo's PR list is the ground truth — check `author.login` in step 3 of pre-flight and pick the matching string.

**Fix**: Match both forms with `||`:

```yaml
if: |
  github.event.pull_request.user.login == 'dependabot[bot]' ||
  github.event.pull_request.user.login == 'app/dependabot'
```

**Diagnostic**: when this is the bug, the auto-merge workflow shows up in the Actions tab for **none** of the open Dependabot PRs. Compare to a working repo where you should see one auto-merge run per `synchronize` event. If the count is zero, the `if:` line is wrong.

**Same trap applies to** `github.actor == 'dependabot[bot]'` and any `dependabot` GitHub Action filter. Always use `||` with both logins.

---

## Verification (mandatory before reporting done)

Do not tell the user "done" until all nine checks pass.

1. **`allow_auto_merge` is on**: `gh api repos/<owner>/<repo> --jq '.allow_auto_merge'` returns `true`.
2. **CI runs on the PR**: open any dependabot PR, confirm `build (..., ..., ...)` checks appear under the PR's Checks tab.
3. **Required check is actually required**: GraphQL query in Step 4 returns `isRequired: true` for the CI check(s).
4. **Patch auto-merges**: trigger a patch bump (e.g. dependabot creates one overnight), confirm it gets approved and merged within a few minutes.
5. **GitHub Actions major auto-merges**: trigger a github-actions major bump. Confirm same as above.
6. **Maven major does NOT auto-merge**: trigger a maven major bump. Confirm the PR is approved by the bot but stays open with `auto-merge: false`.
7. **No double-runs**: in the Actions tab, the latest dependabot PR should have exactly `N` CI runs (where `N` is the size of the matrix), not `2N`.
8. **Auto-merge workflow uses the right token**: open the auto-merge workflow run for a workflow-touching PR (e.g. an `actions/checkout` bump). The Approve and Enable auto-merge steps should NOT show the `workflows` permission error.
9. **Auto-merge workflow actually runs on Dependabot PRs**: in the Actions tab, filter by `auto-merge.yml` and confirm there is at least one run per recent open Dependabot PR (not zero). If the count is zero across all open PRs, Pitfall 8 (login mismatch) is the cause.

If any check fails, do not report success. Go back to Pitfalls and diagnose.

---

## Snags to watch for

- **`enforce_admins: false` does not apply to Dependabot**: Dependabot opens PRs as itself, so the admin bypass does not affect it. Good — the protection will still gate Dependabot.
- **`@dependabot recreate` resets the PR**: if you find a PR that has been force-pushed mid-flight, give it a minute. The auto-merge workflow re-runs on `pull_request` events.
- **First run after enabling**: the first time branch protection is set up, GitHub can take 30–60 seconds to register the required checks. Re-querying GraphQL immediately may still show `isRequired: false`. Wait, then re-check.
- **Admins can push even with branch protection** when `enforce_admins: false`. So you can hot-fix without disabling protection.
- **`mergeStateStatus` cheat sheet** — the state is more nuanced than "blocked" / "not blocked":

  | State | Meaning | Action |
  | --- | --- | --- |
  | `CLEAN` | ready to merge, all checks pass | wait for auto-merge to fire |
  | `BEHIND` | base branch has new commits the PR doesn't have | see Pitfall 7 |
  | `BLOCKED` | a required review or check is missing | inspect the PR |
  | `DIRTY` | has a real merge conflict (not just "behind") | manual resolution required |
  | `UNSTABLE` | checks are still running or one just failed | wait or look at the failed check |
  | `UNKNOWN` | GitHub is still computing the state | refresh in a few seconds |

- **Old PRs don't pick up new auto-merge logic automatically**. If you change `auto-merge.yml` (e.g. switch from `target: minor` to native `gh pr merge --auto`), existing open Dependabot PRs will keep using the old workflow. To force them through the new logic, close and reopen:

  ```bash
  gh pr close <N> --delete-branch=false
  gh pr reopen <N>
  ```

  The `pull_request` event (action: `reopened`) re-runs the workflow. This is the only way short of waiting for dependabot to push a new commit.

- **`@dependabot rebase` is a force-push**. When you post `@dependabot rebase` on a PR with auto-merge enabled, dependabot force-pushes its branch, which **resets `autoMergeRequest` to null**. The auto-merge workflow then re-runs and re-enables it. This is fine, but means the rebase → re-enable sequence takes ~30–60 seconds and the merge itself happens on a third workflow run. Don't be surprised by the delay.

- **`gh push` from the CLI to a workflow file needs `workflow` scope**. If your local git remote is HTTPS and your `gh` token doesn't have the `workflow` OAuth scope, you get: "refusing to allow an OAuth App to create or update workflow `.github/workflows/...` without `workflow` scope". Two fixes:
  1. Switch the remote to SSH: `git remote set-url origin git@github.com:<owner>/<repo>.git`
  2. Re-authenticate `gh` with the `workflow` scope (note: classic OAuth app tokens cannot add this scope; you need a fine-grained PAT or a different auth method). SSH is simpler.

- **`@dependabot rebase` vs `close+reopen`: pick by intent.** Both re-trigger the `pull_request` workflow, but they differ in what happens to the PR branch:
  - `@dependabot rebase` — **force-pushes** the PR branch onto the latest base, **resets `autoMergeRequest` to null**, then the auto-merge workflow re-runs and re-enables it. Use when you want the PR to also pick up the latest base (resolves BEHIND, lets you skip ahead of a slow rebase).
  - `gh pr close <N> --delete-branch=false && gh pr reopen <N>` — **does not touch the branch**. Just re-fires the `pull_request` event. Use when you only want to force the auto-merge workflow to re-evaluate (e.g. you just changed `auto-merge.yml`'s policy and want existing PRs to be judged under the new rules without rebase churn).
  - Neither is "wrong", but pick deliberately. Default to `close+reopen`; reach for rebase only when you also need to update the base.

- **Changing `commit-message.prefix` on already-open PRs produces duplicated prefixes**. If a PR's title is already `build(deps): bump foo` and you push a new `dependabot.yml` with `commit-message.prefix: build(deps)`, the next rebase will produce `build(deps)(deps): bump foo`. Cosmetic only — does not break auto-merge. To clean up, `close+reopen` the affected PR; the next dependabot push will regenerate the title under the new prefix.

- **`open-pull-requests-limit: 100` is a foot-gun**. The Dependabot default is 5. Daily schedule + limit 100 = a backlog that never drains and a noisy PR tab. Recommended defaults: `weekly` schedule, limit 5–10 for github-actions, 10–20 for maven. Grouped updates (`groups:` in dependabot.yml) reduce the multiplier — one grouped PR replaces N individual ones.

---

## Quick reference — all `gh` commands used in this skill

```bash
# Auth check
gh auth status

# allow_auto_merge (Step 3)
gh api repos/<owner>/<repo> --jq '.allow_auto_merge'
gh api -X PATCH repos/<owner>/<repo> -f allow_auto_merge=true

# Secret inventory (Pre-flight 5)
gh secret list

# Current branch protection
gh api repos/<owner>/<repo>/branches/master/protection

# PR list
gh pr list --state open --json number,title,headRefName,mergeStateStatus

# PR status + auto-merge flag
gh pr view <N> --json mergeStateStatus,autoMergeRequest,statusCheckRollup

# Real check names + isRequired
gh api graphql -F query='...'   # see Step 4

# Set branch protection
gh api -X PUT repos/<owner>/<repo>/branches/master/protection \
  -H "Accept: application/vnd.github+json" --input - <<'EOF'
{ ... }
EOF

# Re-trigger auto-merge workflow on a stuck PR
gh pr close <N> --delete-branch=false
gh pr reopen <N>

# Force dependabot to rebase a PR (resolves BEHIND)
gh pr comment <N> --body "@dependabot rebase"

# Batch-rebase all open Dependabot PRs (filter by author — never rebase human PRs)
gh pr list --state open --json number,author \
  --jq '.[] | select(.author.login == "app/dependabot" or .author.login == "dependabot[bot]") | .number' | \
  xargs -I {} sh -c 'gh pr comment {} --body "@dependabot rebase"'

# Set git remote to SSH (when gh OAuth lacks workflow scope)
git remote set-url origin git@github.com:<owner>/<repo>.git
```

---

## When the user pushes back

- "I don't want CI to run twice" → Step 2 + Pitfall 4.
- "Major versions should not auto-merge at all" → drop the third `||` branch in Step 1.
- "Maven major should also auto-merge" → remove the `dependabot/github_actions/*` prefix gate in Step 1.
- "I want strict review" → set `required_pull_request_reviews` in Step 4. Note: this will likely conflict with `enforce_admins: false`; pick one.
- "I want a different merge method" → change `--rebase` to `--merge` or `--squash` in `auto-merge.yml`.
- "I changed the workflow but old PRs aren't auto-merging with the new logic" → close + reopen each open PR. See Snags.
- "My PRs are stuck at `BEHIND` even though CI passes" → Pitfall 7. Easiest fix: run the batch rebase script in Quick Reference on all open Dependabot PRs.
- "The auto-merge workflow says 'OAuth App cannot create or update workflow'" → your `gh` token lacks the `workflow` scope. Either switch the git remote to SSH (Quick Reference) or use a fine-grained PAT.
- "I have a PAT but it's still using `GITHUB_TOKEN`" → check the secret name case (`secrets.mytoken` ≠ `secrets.MYTOKEN`). `gh secret list` shows the actual names.
- "The auto-merge workflow shows `skipped` for the merge step but no error" → Pitfall 6 (`allow_auto_merge` off). Run the run-log diagnostic command in Pitfall 6 to confirm.
- "Auto-merge workflow never runs at all on Dependabot PRs" → Pitfall 8 (login mismatch). Check `author.login` on the open PRs and update the workflow's `if:` line.
