---
name: dependabot-automerge-skill
description: Set up or repair GitHub Dependabot auto-merge for a repository. Use when the user mentions dependabot, auto-merge, dependabot PR stuck, semver-major merging, GitHub Actions PRs not auto-merging, branch protection required checks, allow_auto_merge disabled, or wants to reduce manual PR churn. Triggers on phrases like "set up dependabot auto-merge", "PR stuck waiting for checks", "auto-merge not waiting for CI", "major version dependabot", "branch protection required status check", "PR stuck BEHIND", "PR stuck DIRTY after a sibling dependabot PR merged", "auto-merge returns 422", "oauth app cannot create workflow", "auto-merge workflow never runs", "app/dependabot vs dependabot[bot]", "I bumped the JDK matrix and now auto-merge is broken", "dependabot PRs stuck after I changed build.yml", "CI looks like it's running but isn't gating the PR", "all my dependabot PRs went BEHIND at once", "I have gh but I don't want to create a separate PAT", "rebase produced a real conflict not just BEHIND", "first batch of dependabot PRs are all major version bumps touching workflow files", "MYTOKEN secret is set but auto-merge still fails with empty GH_TOKEN", "two workflows produce the same check name and branch protection is ambiguous", "dependabot.yml produces double prefix like build(deps)(deps)", "dependabot PR title is build(deps-dev)(deps-dev) maven double prefix". Does NOT use when the user only wants to configure dependabot.yml update schedule, or only wants to enable dependabot security updates without auto-merge.
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
- "dependabot PRs are stuck after I changed build.yml"
- "branch protection required check name no longer matches"
- "CI looks like it's running but isn't gating the PR"
- "I bumped the JDK matrix and now auto-merge is broken"
- "dependabot PR went DIRTY (not BEHIND) after a sibling PR merged"
- "I have `gh` authenticated but I don't want to create a separate PAT"
- "the first dependabot PRs are all major version bumps touching workflow files"
- "MYTOKEN is set but `gh pr review` still errors with empty GH_TOKEN in the log"
- "two CI workflows produce the same `build (os, java)` check name"
- "dependabot PR titles look like `build(deps)(deps): ...` with duplicated scope"

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
5. **Inventory available tokens**:
   ```bash
   gh secret list
   gh auth token | cut -c1-10   # check the prefix — gho_ (OAuth) vs ghp_ (classic PAT) vs github_pat_ (fine-grained)
   ```

   **Two paths, pick the easier one**:

   - **If the user is the repo admin and `gh` is authenticated** — the existing user OAuth token (`gho_*` prefix) has implicit `workflow` scope for repos where the user is admin. You can use it directly as `MYTOKEN`. See Pitfall 5 for the smoke-test and the `gh auth refresh` anti-pattern.
   - **Otherwise** — you need a separate PAT (classic with `repo` + `workflow` scope, or fine-grained with `Contents: write` + `Pull requests: write` + `Actions: write`). Ask the user to create one and store it as a repo secret.

   The classic GITHUB_TOKEN supplied to a workflow is never sufficient — it cannot enable auto-merge on PRs that modify `.github/workflows/*.yml`. See Pitfall 5.

If the repo does not have `gh` auth, stop and ask the user to log in. Do not guess at branch protection.

6. **Sync local clone with `origin` before reading local files.** A local clone left over from a previous session can be many commits behind `origin/master` (especially on repos where dependabot has been auto-merging regularly). `git status` reports "nothing to commit, working tree clean" in that state because it is comparing against the stale local tracking ref — the staleness only shows up after `git fetch`. Always run `git fetch origin` (or `git pull --ff-only`) before reading `.github/workflows/*.yml` from disk. Otherwise you will optimize against files that are no longer what GitHub is running, push a "fix" that is actually a no-op, and wonder why nothing changed.

---

## Strategy

There are six moving parts. All are required.

| # | File / setting | Purpose |
| --- | --- | --- |
| 1 | `.github/workflows/auto-merge.yml` | Approve and enable auto-merge on qualifying Dependabot PRs |
| 2 | `.github/workflows/build.yml` | Run CI on `pull_request` so required checks appear on the PR |
| 3 | `allow_auto_merge` repo setting | Precondition for `gh pr merge --auto` — see Pitfall 6 |
| 4 | A token secret (`repo` + `workflow` scope) | A user OAuth token (admin) or a separate PAT. `GITHUB_TOKEN` cannot enable auto-merge on PRs that modify `.github/workflows/*.yml` — see Pitfall 5 |
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
- `required_linear_history: true` (recommended) — keeps master history clean, which is what `--rebase` auto-merge produces anyway. Without this, admins can sneak in merge commits that defeat the whole point.

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

### Step 7 — Self-improvement (mandatory, not optional)

After all twelve Verification checks pass, before reporting success:

1. **Write a per-project notes file** at `<project>/docs/dependabot-optimization-notes.md` (see the template in the Self-improvement loop section below). This is for the project owner — it documents what changed in *their* repo and why.
2. **Update this skill** (`SKILL.md` + `README.md`) with anything new you learned this run. See the Self-improvement loop section for what qualifies and how to do it.
3. **Commit the skill changes locally**. Do **not** push — the skill is loaded from a local path and has no remote.
4. **Do not commit the project notes file** unless the user asks. The notes are useful as a local artifact and a project record; committing is the user's call.

The user is paying for the optimized repo *and* an improved skill. Skipping this step means the next repo you optimize will hit the same traps.

---

## Pitfalls (read these before declaring success)

These are the ten things that have actually broken in production. Re-check each one before finishing.

### Pitfall 1 — Major version updates never merge

**Symptom**: `actions/checkout 6→7` style PRs sit open forever.

**Cause**: Auto-merge workflow only matches `semver-patch` / `semver-minor`.

**Fix**: Add `semver-major` to the `if` chain, gated by head ref prefix (see policy table). The head ref for Dependabot is always `dependabot/<ecosystem>/<branch>-<dep>-<version>`, so the ecosystem name is a reliable discriminator.

### Pitfall 2 — Auto-merge skips CI

**Symptom**: PR is merged before any CI runs at all.

**Cause**: `build.yml` only triggers on `push`, not on `pull_request`. No required check exists on the PR, so `gh pr merge --auto` merges immediately.

**Fix**: Add `pull_request:` to `on:` in the CI workflow AND set up branch protection with the resulting check marked required. Both are needed; one alone does not help.

**Subtle variant — "it kind of works, but only by accident"**: a `build.yml` of just `on: [push]` may *appear* to gate PRs because Dependabot pushes to its PR branch on every rebase, which fires the `push` event, which runs CI, which then shows up in the PR's Checks tab via the branch-ref annotation. The PR can look green and auto-merge cleanly. But:
- It depends on Dependabot's push-on-rebase behavior. If Dependabot ever switches to a cherry-pick or merge strategy, the gate silently disappears.
- `gh run list --workflow="Java CI"` will show runs triggered by `push` events, never by `pull_request`. That's the smoking gun.
- Any push to a non-master branch (e.g. a feature branch) also triggers CI, wasting runner minutes (Pitfall 4).
- Branch protection's "required check" is technically satisfied by a check produced from a `push` event, not the canonical `pull_request` event. Some integrations (e.g. merge queues) care about this distinction.

**Diagnostic**:

```bash
gh run list --workflow="<your build workflow>" --limit 20 \
  --json event,headBranch,conclusion \
  --jq '.[] | "\(.event)  \(.headBranch)  \(.conclusion)"'
```

If you see zero `pull_request` rows among the last 20 runs, your CI is not running on PR events. Fix the `on:` block (Step 2).

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

**Empty-secret detection in the workflow log**: when the secret exists in the repo's secret list but holds no value, the workflow log shows `GH_TOKEN: ` (truly empty — colon followed by nothing), not `GH_TOKEN: ***` (masked). The trailing colon with no asterisks is the giveaway. The `gh pr review` step then errors with `gh: To use GitHub CLI in a GitHub Actions workflow, set the GH_TOKEN environment variable`. Fix: `gh secret set MYTOKEN --repo <owner>/<repo> --body "$TOKEN"`. To inspect, run `gh run view <id> --log-failed` and look at the env block of the failing step.

**Shortcut — user OAuth token, no separate PAT needed**: If the user invoking this skill is the repo admin and already has `gh` authenticated, you do not need a separate PAT. A user OAuth token (`gho_*` prefix) has implicit `workflow` scope for repos where the user is admin — it can approve and enable auto-merge on PRs that modify workflow files. Use it directly:

```bash
TOKEN=$(gh auth token)
gh secret set MYTOKEN --repo <owner>/<repo> --body "$TOKEN"
```

Verify the scope with a smoke test on any open workflow-touching Dependabot PR:

```bash
gh pr merge <N> --auto --rebase
gh pr view <N> --json autoMergeRequest --jq '.autoMergeRequest.enabledBy.login'
```

If the second command prints a real login (e.g. `XenoAmess`), the token has sufficient scope. The PR's `autoMergeRequest` is now set; CI will run and the merge will fire when green. If the command fails with the `enablePullRequestAutoMerge` GraphQL error, the token is a classic PAT or app token without `workflow` — fall back to the separate-PAT path.

**Smoke-test variant — no open dependabot PRs available.** If the repo has no open dependabot PRs (e.g. all are merged or stuck-closed), you can still verify scope by running `gh pr merge <N> --auto --rebase` against a closed dependabot PR. The response distinguishes between the two failure modes:

- `Pull request Pull request is closed` (GraphQL error) → token has the right scope; the PR's `state: closed` is what blocked the operation. This is the success case for the smoke test.
- `OAuth App ... without 'workflows' permission (enablePullRequestAutoMerge)` → token lacks `workflow` scope; fall back to the separate-PAT path.

This is useful when you want to verify `MYTOKEN` scope without waiting for the next dependabot cycle to produce a fresh PR.

**Anti-pattern — `gh auth refresh -s workflow`**: do not try to add the `workflow` scope to an existing OAuth token from a non-interactive CLI session. The command will hang waiting for a browser confirmation. Either use the shortcut above (if the user is admin), or ask the user to add the scope via the GitHub web UI (`Settings → Developer settings → Personal access tokens → [token] → Edit scopes → Workflow ✓ → Update token`) and then continue.

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

**Adjacent-line variant — the race produces DIRTY, not BEHIND**: when two parallel Dependabot PRs both touch adjacent lines in the same file (e.g. both bump a different action version in `build.yml`), the second to rebase after the first's merge can hit `mergeStateStatus: DIRTY`, a real merge conflict, not a stale-base issue. Cause: PR A's merge removed a context line that PR B's diff was anchored to (e.g. PR B's diff had `actions/checkout@v6` as a context line, and PR A's merge changed it to `@v7`). The rebase patch no longer applies cleanly. The good news: dependabot's next hourly auto-rebase regenerates the diff with the new context and usually succeeds. The bad news: that means another hour of waiting, with the PR showing DIRTY the whole time. **Do not push a manual fix — see Pitfall 10.** Either wait, or `@dependabot rebase` to force the next attempt sooner.

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

**Normalization quirk — read this if you are debugging**: even when `gh pr list --json author` shows `app/dependabot` for a PR opened by the new GitHub App, the value `github.event.pull_request.user.login` inside the workflow run is normalized back to `dependabot[bot]`. So a workflow gated on the legacy form may still "work" today, even on a fully migrated repo. The match-both form is still the right defense — the normalization is undocumented behavior and could change without notice.

### Pitfall 9 — Matrix change silently invalidates branch protection required checks

**Symptom**: After bumping a CI workflow matrix value (e.g. JDK `11` → `17`, or adding a new OS image to `matrix.os`), every new PR shows `mergeStateStatus: BLOCKED` even though all checks are green. The auto-merge workflow does not fire because the required checks never match anything.

**Cause**: When the `matrix` axes change, the actual check name changes too — `build (ubuntu-latest, 11, false)` becomes `build (ubuntu-latest, 17, false)`. The old name is still listed as required in branch protection, but no future PR will ever produce a check by that name. New PRs produce checks under the new name, which is not required, so they are gated by zero required checks and GitHub treats them as "not satisfied".

**Diagnostic**:

```bash
gh api graphql -F query='
query {
  repository(owner: "<owner>", name: "<repo>") {
    pullRequest(number: <N>) {
      statusCheckRollup {
        contexts(first: 20) {
          nodes {
            ... on CheckRun { name isRequired(pullRequestNumber: <N>) }
          }
        }
      }
    }
  }
}' | jq '.data.repository.pullRequest.statusCheckRollup.contexts.nodes'
```

If the actual check names have `isRequired: false` (or no entry exists under that name), branch protection has gone stale relative to the CI matrix.

**Fix**: After any matrix change, run Step 4 again with the new actual names from GraphQL. Treat the matrix dimensions as part of the check contract — a refactor that touches `build.yml` matrix values should always re-run Step 4 in the same change.

**Bonus**: this also bites if you rename the workflow file or the job name. Anything that changes the canonical check name must trigger a branch protection review.

### Pitfall 10 — Manually fixing DIRTY rebase conflicts desyncs the PR

**Symptom**: A dependabot PR sits at `mergeStateStatus: DIRTY` (real conflict, see Pitfall 7 adjacent-line variant). The natural response is to fetch the PR branch, resolve the conflict locally, and push. The PR shows CLEAN after the push, the auto-merge workflow re-runs, the merge happens. Problem solved — right?

**Cause**: There is no bug to fix. This is a trap.

**Why the manual push is wrong**:

- The push is **not** signed by dependabot. The PR's commit author changes from `dependabot[bot]` to your login, while the PR's `user.login` (dependabot) stays the same. This dual-identity state can confuse integrations that key off commit author (CODEOWNERS, security alerts, some branch policies).
- Subsequent dependabot activity on the same PR — for example, a second rebase triggered by the user commenting `@dependabot rebase` — will collide with your non-dependabot commits. The branch's history becomes a mix of signed and unsigned commits, and you may find yourself in a "fixup commit" loop.
- The real resolution is already on its way. Dependabot re-rebases on its hourly schedule; the second attempt usually succeeds because the conflict is now between a fresh diff and the latest master.

**Fix**: Do nothing. Wait for the next hourly dependabot rebase cycle. If you need it faster, `@dependabot rebase` on the PR — but do **not** push a manual fix.

**Diagnostic when you are tempted to fix**: `git fetch origin <head-ref>` + `git checkout <head-ref>`. If the file already shows the correct final state (e.g. `actions/checkout@v7` and `actions/cache@v6`), dependabot has already re-rebased and your push is redundant. If the file shows a partial state, wait one more cycle and re-check; do not push.

**Why this deserves its own pitfall**: the "DIRTY" state in GitHub's merge status is *meant* to signal "needs human resolution", so the manual-fix reflex is correct in the general case. It is wrong specifically for dependabot PRs because the bot is on a timer that will retry automatically. Forgetting that distinction costs you the PR's signed-commit guarantee.

### Pitfall 11 — Third-party auto-merge action silently succeeds while doing nothing

**Symptom**: An open dependabot PR has `mergeStateStatus: CLEAN` and all required checks pass, but the PR is not merging. `gh pr view <N> --json autoMergeRequest` returns `null`. The `auto-merge` workflow run shows every step with `conclusion: success` (including any "Merge dependabot PR" or "Enable auto-merge" step). Looking at the actions log for that step, you see no error output — it just appears to do nothing.

**Cause**: The popular `ahmadnassri/action-dependabot-auto-merge@v2` action (and any other wrapper that hides `gh pr merge --auto` behind a try/catch with a swallowed error) masks the most common failure: `gh pr merge --auto` returning 422 because the repo's `allow_auto_merge` setting is off (Pitfall 6). Some versions also swallow 422s from token scope failures (Pitfall 5). The action's exit code is 0 in either case, so the workflow run is green, but the PR's `autoMergeRequest` never gets set.

**Why the symptom is misleading**: a wrapper action that reports "success" but didn't actually do the thing it's named after is the worst kind of failure — there is no error to read. The diagnostic only works if you compare the workflow run's success to the PR's actual state. If they disagree (run is success, PR is not auto-merging), the wrapper is hiding something.

**Diagnostic**:

```bash
# Compare workflow conclusion to PR's actual auto-merge state
for n in $(gh pr list --state open --json number --jq '.[].number'); do
  run=$(gh run list --workflow="<your auto-merge workflow>" --limit 50 --json conclusion,headBranch --jq ".[] | select(.headBranch | startswith(\"dependabot/\")) | .conclusion" | head -1)
  amr=$(gh pr view $n --json autoMergeRequest --jq '.autoMergeRequest // "null"')
  echo "PR #$n  run=$run  autoMergeRequest=$amr"
done
```

If you see `run=success  autoMergeRequest=null` for any PR, the wrapper is hiding a 422. Check `gh api repos/<owner>/<repo> --jq '.allow_auto_merge'` — if it's `false`, that's the cause. If it's `true`, the issue is most likely token scope (Pitfall 5).

**Fix**: Replace the wrapper with explicit steps so each step's exit code is the success criterion and any 422 surfaces as a visible failure:

```yaml
- name: Approve dependabot PR
  if: steps.check.outputs.should_merge == 'true'
  run: gh pr review --approve "$PR_URL"
  env:
    PR_URL: ${{ github.event.pull_request.html_url }}
    GH_TOKEN: ${{ secrets.MYTOKEN }}

- name: Enable auto-merge
  if: steps.check.outputs.should_merge == 'true'
  run: gh pr merge --auto --rebase "$PR_URL"
  env:
    PR_URL: ${{ github.event.pull_request.html_url }}
    GH_TOKEN: ${{ secrets.MYTOKEN }}
```

Both steps have a single command; if either returns non-zero (including 422), the step's `conclusion` becomes `failure` and shows up red in the Actions tab. The PR's `autoMergeRequest` either gets set or it does not — you can read it directly.

**Why this deserves its own pitfall**: the wrapper action is the most-clicked result when you search "github action dependabot auto-merge", and it has a fundamental design flaw (swallowed errors with green exit code) that hides the most common failure mode. Anyone inheriting an existing `auto-merge.yml` that uses this action is likely in the "green run, no merge" state right now and does not know it.

### Pitfall 12 — Two workflows with the same job name + matrix produce the same check name

**Symptom**: A repo has two CI workflows (e.g. a fast `build.yml` and a more thorough `build_idea_plugin.yml` for the IDE plugins). The PR's Checks tab shows the matrix checks appearing **twice each** — `build (ubuntu-latest, 21, false)` once from each workflow, `build (windows-latest, 21, false)` once from each. Branch protection is set to require `build (ubuntu-latest, 21, false)`, and GitHub accepts the requirement, but you cannot tell which workflow's check satisfied it. If one workflow passes and the other fails, the required-check logic is ambiguous.

**Cause**: The GitHub Actions check name format is `<job-name> (<matrix-axes>)` — **the workflow name is NOT part of it.** Two workflows with the same `name:` field, a job called `build` with the same matrix axes, will produce identical check names. Renaming the workflow's `name:` does not help; the job name has to differ.

**Diagnostic**: query the check rollup on any open PR. If you see the same `name` listed twice with different workflow IDs, this pitfall applies:

```bash
gh api graphql -F query='
query {
  repository(owner: "<owner>", name: "<repo>") {
    pullRequest(number: <N>) {
      statusCheckRollup {
        contexts(first: 20) {
          nodes { ... on CheckRun { name databaseId } }
        }
      }
    }
  }
}'
```

**Fix**: rename the job in one of the workflows so the check names become distinct. Example: keep the main workflow's job as `build` and rename the plugin workflow's job to `plugins-build`. The check name will then be `plugins-build (ubuntu-latest, 21, false)`, and branch protection can require both `build (ubuntu-latest, 21, false)` and `plugins-build (ubuntu-latest, 21, false)` independently. Both must pass for the PR to merge; if either fails, the merge blocks with a clear attribution.

**Why this deserves its own pitfall**: this is a real hazard for any repo that has split its CI into "fast main build" and "thorough extra build" workflows — a very common shape, especially for monorepos with sub-projects. Branch protection silently accepts the duplicate name, so the misconfiguration is invisible until you actually need to debug a CI failure and find the wrong check satisfied the gate.

---

## Verification (mandatory before reporting done)

Do not tell the user "done" until all twelve checks pass. After that, do not report "done" without also completing the Self-improvement loop below.

1. **`allow_auto_merge` is on**: `gh api repos/<owner>/<repo> --jq '.allow_auto_merge'` returns `true`.
2. **CI runs on the PR**: open any dependabot PR, confirm `build (..., ..., ...)` checks appear under the PR's Checks tab.
3. **Required check is actually required**: GraphQL query in Step 4 returns `isRequired: true` for the CI check(s).
4. **Patch auto-merges**: trigger a patch bump (e.g. dependabot creates one overnight), confirm it gets approved and merged within a few minutes.
5. **GitHub Actions major auto-merges**: trigger a github-actions major bump. Confirm same as above.
6. **Maven major does NOT auto-merge**: trigger a maven major bump. Confirm the PR is approved by the bot but stays open with `autoMerge: false`.
7. **No double-runs**: in the Actions tab, the latest dependabot PR should have exactly `N` CI runs (where `N` is the size of the matrix), not `2N`.
8. **Auto-merge workflow uses the right token**: open the auto-merge workflow run for a workflow-touching PR (e.g. an `actions/checkout` bump). The Approve and Enable auto-merge steps should NOT show the `workflows` permission error.
9. **Auto-merge workflow actually runs on Dependabot PRs**: in the Actions tab, filter by `auto-merge.yml` and confirm there is at least one run per recent open Dependabot PR (not zero). If the count is zero across all open PRs, Pitfall 8 (login mismatch) is the cause.
10. **CI is triggered by `pull_request` events, not just `push`**: `gh run list --workflow="<build workflow>" --limit 20 --json event --jq '.[].event'` should show a healthy mix including `pull_request`. If every row says `push`, the `on:` block is wrong (Pitfall 2 subtle variant — CI "looks like it works" because Dependabot pushes on rebase).
11. **`MYTOKEN` has the required scope (no separate PAT needed if you took the user-OAuth-token path)**: `gh pr merge <N> --auto --rebase` on any open workflow-touching Dependabot PR sets `autoMergeRequest.enabledBy` to a real login (not null, not `web-flow`). This proves the secret has the implicit `workflow` scope. If you took the separate-PAT path, this is covered by check 8.
12. **No duplicate check names across multiple CI workflows**: GraphQL query on the check rollup of any open PR should show each `<job> (<matrix>)` exactly once. If the same name appears twice (different `databaseId`), Pitfall 12 applies — branch protection's "required" gate is ambiguous, and a failure in one workflow can be masked by a success in the other. Also: when running `gh secret set MYTOKEN`, verify the secret actually has a value (not an empty string); an empty secret shows as `GH_TOKEN: ` (colon with no value) in the workflow log, not as `GH_TOKEN: ***` (Pitfall 5 empty-secret diagnostic).

If any check fails, do not report success. Go back to Pitfalls and diagnose.

---

## Self-improvement loop (run after a successful optimization)

This skill improves with every repo you apply it to. After Verification
(all twelve checks pass), audit what you actually learned during
**this** run and update this file before finishing the conversation.
This is not optional polish — it is part of the deliverable. **Step 7**
of Implementation makes this a required step, not a recommended one.

### Per-project notes file (companion to the skill update)

For every project you apply this skill to, write a per-project notes
file at `<project>/docs/dependabot-optimization-notes.md`. This is a
project-local companion to the skill — the project owner reads it to
understand what changed and why, and you reference it when updating
the skill.

Use this template (adapt the sections; do not omit any):

```markdown
# Dependabot Optimization Notes — <project-name>

## Context
- Project: <owner>/<repo> (<lang>/<ecosystem>)
- Default branch: <name>
- Dependabot ecosystems: <list>
- CI matrix: <axes and dimensions>
- Pre-optimization state: <open PRs, branch protection, allow_auto_merge>

## What was done
<numbered list of changes>

## What worked first time
<list>

## What was tricky / required iteration
<list>

## Key findings (also fed back to the skill)
<numbered list, each linking to the relevant Pitfall / Snag / Verification change>

## Verification results
<table of 12 checks, with actual results>
```

After writing the per-project notes, perform the skill update as
described below. Both are required before reporting "done".

### When to update the skill

### When to update the skill

Update SKILL.md / README.md when **any** of these happened during this optimization:

1. **New failure mode** — a symptom → cause → fix pattern that does not
   match any existing Pitfall. Promote it to the next Pitfall number,
   with the exact symptom string a future user is likely to paste at you.
2. **Missing diagnostic** — a faster way to recognize a failure
   (e.g. reading the workflow run log step conclusions, checking
   `author.login` on open PRs). Add it to the existing Pitfall or as a
   new Verification check.
3. **Strategy gap** — a moving part that turned out to be required beyond
   the current six (e.g. `dependabot.yml` grouping to suppress noise).
   Add it to the Strategy table and update the count elsewhere.
4. **Sharp edge already in scope** — a snag you hit but is not yet in
   the Snags section (e.g. commit-prefix duplication, OAuth scope during
   `git push`). Add it if it is general enough to recur.
5. **New trigger phrase** — if the user described their problem with
   words the description field would not have matched, extend the
   description and the When-to-use list.
6. **Stale example** — a code snippet or matrix axis that no longer
   reflects modern practice (e.g. JDK 11 in a matrix when 11 is EOL).

Do **not** update the skill when:

- You only followed the existing procedure without learning anything new.
- The lesson is project-specific and does not generalize.
- The fix would be a one-off workaround, not a repeatable pattern.

### How to update

1. Make the **minimum surgical edit** that captures the lesson. Do not
   rewrite sections that still apply; append or insert.
2. Update affected counts (e.g. "eleven pitfalls" → "twelve pitfalls",
   "twelve verification checks" → "thirteen"). Grep for the old count first
   to catch every reference.
3. Update README.md to match if the count or trigger list changed.
4. Sanity-check cross-references still resolve — a new Pitfall N+1
   referenced from Pre-flight and Verification must actually exist.
5. Commit locally. **Do not push** — this skill has no remote; opencode
   loads it from a local path.
   ```bash
   cd /home/xenoamess/workspace/dependabot-automerge-skill
   git add SKILL.md README.md
   git commit -m "feat: <one-line description of the lesson learned>"
   ```

### Worked example

After optimizing `XenoAmess/docker-image-rebecca` (the run that produced
this revision):

- New symptom: workflow never runs because the `if:` line checks
  `dependabot[bot]` but the repo's PRs are authored by `app/dependabot`
  (GitHub migrated Dependabot to a GitHub App in 2024). Promoted to
  Pitfall 8 with the login-era table and the `||` fix.
- Diagnostic gap: `allow_auto_merge: false` was visible in a run log as
  a `skipped` step conclusion, not as a `failure`. Added a run-log
  diagnostic one-liner to Pitfall 6.
- Strategy gap: the original "dependabot.yml is optional" underplayed
  how much grouping + weekly schedule + sensible PR limit matters for
  user-visible noise. Promoted to Step 6 with a concrete recipe.
- Batch rebase script in Quick Reference iterated **all** open PRs
  instead of filtering by author — would rebase human PRs. Fixed.
- Stale example: JDK 11 matrix axis when 11 is past EOL free support.
  Updated to JDK 17.
- Counts bumped: seven → eight pitfalls, eight → nine verification
  checks. New trigger phrases ("workflow never runs", "app/dependabot
  vs dependabot[bot]") added to description.

After optimizing `XenoAmess/x8l_idea_plugin` (the run that produced
the previous revision):

- **Pitfall 9** added: changing a CI matrix value (e.g. JDK 11 → 17)
  silently invalidates branch protection's required check names.
  Symptom is `mergeStateStatus: BLOCKED` with green checks, which
  looks like a different problem. Diagnostic: GraphQL `isRequired`
  query on the new check names.
- **Pitfall 8 enriched** with the login normalization quirk:
  `gh pr list` shows `app/dependabot` but the workflow context
  returns `dependabot[bot]`. A defensive `||` is still required
  because the normalization is undocumented.
- **Pitfall 2 enriched** with the subtle-variant diagnostic: a
  `build.yml` of just `on: [push]` can "accidentally work" because
  Dependabot pushes on rebase, firing the `push` event. The CI
  shows up on the PR via the branch-ref annotation, but it's not
  a real `pull_request` gate. Diagnostic: `gh run list --json event`
  should show `pull_request` rows.
- **Verification check #10 added**: CI must be triggered by
  `pull_request` events, not just `push` (Pitfall 2 subtle variant).
- **Step 4 enriched**: `required_linear_history: true` recommended
  alongside `--rebase` auto-merge.
- **Three new snags** added: migration push flips all open PRs to
  `BEHIND` (plan to batch rebase after), grouping in dependabot.yml
  closes individual PRs (expected, replacement PR has highest-severity
  type), AdoptOpenJDK → Temurin distribution migration.
- Counts bumped: eight → nine pitfalls, nine → ten verification
  checks. New trigger phrases ("bumped JDK matrix", "build.yml change
  broke auto-merge", "PRs went BEHIND after I pushed the workflow")
  added to description.

After optimizing `cyanpotion/cyan_zip` (the run that produced the
latest revision):

- **Pitfall 10 added**: manually pushing a fix to a `mergeStateStatus:
  DIRTY` dependabot PR looks correct but desyncs the PR's signed-commit
  author and creates a fixup-commit loop. Symptom is the urge to "just
  fix the conflict", which is the right reflex for human-authored PRs
  but wrong for dependabot. Fix: wait, or `@dependabot rebase`. See
  Pitfall 7 adjacent-line variant for the trigger.
- **Pitfall 7 enriched** with the adjacent-line DIRTY variant. When two
  parallel dependabot PRs both touch adjacent lines in the same file
  (e.g. both bump a different action version in `build.yml`), the
  second to rebase can hit a real merge conflict, not just BEHIND.
- **Pitfall 5 enriched** with the user-OAuth-token shortcut: if the
  user is admin and `gh` is authenticated, `gh auth token` works as
  `MYTOKEN` directly. Also added the `gh auth refresh -s workflow`
  anti-pattern — it hangs in non-interactive sessions.
- **Verification check #11 added**: smoke-test `MYTOKEN` scope with
  `gh pr merge --auto` on a workflow-touching PR.
- **Pre-flight 5 enriched** with the user-OAuth-token path.
- **Two new snags**: the first dependabot batch tests the hardest path
  (major version bumps touching workflow files); the GitHub App login
  normalization quirk is undocumented behavior that may change.
- **Counts bumped**: nine → ten pitfalls, ten → eleven verification
  checks. New trigger phrases ("I have `gh` but I don't want to
  create a separate PAT", "rebase produced a real conflict not just
  BEHIND", "first batch of dependabot PRs are all major version bumps
  touching workflow files") added to description.
- **Self-improvement loop elevated** from "section to read" to "Step 7
  of Implementation" with a per-project docs file convention
  (`<project>/docs/dependabot-optimization-notes.md`). Both writing
  the doc and updating the skill are required deliverables, not
  optional polish.

After optimizing `XenoAmess/jcpp-maven-plugin` (the run that produced
the latest revision):

- **Pitfall 11 added**: the popular `ahmadnassri/action-dependabot-auto-merge@v2`
  action silently succeeds while doing nothing. Symptom: open dependabot
  PR with `mergeStateStatus: CLEAN` and `autoMergeRequest: null`,
  workflow run is green. Cause: the wrapper swallows 422s from
  `gh pr merge --auto` (most often `allow_auto_merge=false` per Pitfall 6,
  sometimes token scope per Pitfall 5). Diagnostic: compare `gh run list`
  conclusions to `gh pr view --json autoMergeRequest` — disagreement
  means a wrapper is hiding something. Fix: replace with explicit
  `gh pr review --approve` + `gh pr merge --auto --rebase` steps so
  each step's exit code surfaces. This is the most-clicked
  "github action dependabot auto-merge" search result; anyone inheriting
  an `auto-merge.yml` that uses it is likely in the green-but-no-merge
  state right now.
- **Diagnostic one-liner added** to Pitfall 11 — looping over open PRs
  to compare run conclusion vs `autoMergeRequest` is the fast triage.
- **Counts bumped**: ten → eleven pitfalls (verification count unchanged
  at eleven). New trigger phrases ("auto-merge workflow runs but does
  nothing / always skipped", "auto-merge merged without running CI")
  added to description — though these were already in the When-to-use
  list, they were treated as separate symptoms before; now they have a
  shared root cause (Pitfall 11).

One commit, ~140 insertions, no rewrite. This is the expected shape of
an update: targeted edits, counts grepped, README synced, single commit.

After optimizing `XenoAmess/evosuite` (the run that produced this
revision):

- **Pitfall 12 added**: two workflows with the same job name and matrix
  produce the same check name (`<job-name> (<matrix-axes>)`, not
  `<workflow> / <job>`). Symptom is a PR Checks tab showing the matrix
  rows duplicated, and branch protection's "required" gate is
  ambiguous — one workflow's failure can be masked by the other's
  success. Fix: rename the job in one workflow. Common in monorepos
  with split fast/thorough CI; the misconfiguration is invisible until
  you actually need to debug a CI failure.
- **Diagnostic added to Pitfall 5**: when the repo secret (`MYTOKEN`)
  is registered but holds no value, the workflow log shows
  `GH_TOKEN: ` (colon with nothing after) instead of `GH_TOKEN: ***`
  (masked). The error message is `gh: To use GitHub CLI in a GitHub
  Actions workflow, set the GH_TOKEN environment variable`. Fix:
  `gh secret set MYTOKEN --repo <owner>/<repo> --body "$TOKEN"`. The
  empty value is a one-line root cause but very easy to miss — `gh secret
  list` shows the secret, so the existence check passes; only the
  log-failed inspection catches the empty value.
- **Three new snags**: `update-branch` API returns 422 when master is
  unchanged (use close+reopen as the fallback); `include: "scope"` +
  `prefix: "build(deps)"` collides into `build(deps)(deps): ...`
  (drop the include); check name format reminder (`<job> (<matrix>)`,
  not `<workflow> / <job>`).
- **Counts bumped**: eleven → twelve pitfalls, eleven → twelve
  verification checks. New trigger phrases ("MYTOKEN is set but auto-merge
  still fails with empty GH_TOKEN", "two workflows produce the same
  check name", "dependabot titles look like build(deps)(deps)") added
  to description and When-to-use.

After optimizing `cyanpotion/multi_language`
this revision):

- **All three of Pitfall 6, 11, and the missing-`pull_request`-event form of Pitfall 2 fired at once in the same repo.** Pre-flight found 7 open dependabot PRs all stuck at `mergeStateStatus: CLEAN` with `autoMergeRequest: null`, every `auto-merge` workflow run green, `allow_auto_merge: false`, and `build.yml` had only `on: [push]`. The wrapper action (Pitfall 11) was hiding the `allow_auto_merge: false` 422 (Pitfall 6), and the `on: [push]`-only CI was "accidentally" running because dependabot pushes on rebase (Pitfall 2 subtle variant). Replacing the wrapper with explicit `gh pr review` + `gh pr merge --auto --rebase` steps exposed all three failures as red `conclusion: failure` rows on the very first run. Lesson: when troubleshooting a "stuck" dependabot repo, the three pitfalls compound so often that the diagnostic is the same in all three cases — `gh run list` conclusion vs `gh pr view --json autoMergeRequest` — and the fix is the same in all three cases too (replace the wrapper, enable `allow_auto_merge`, add `pull_request` to `on:`).

After optimizing `XenoAmess/add-from-repo-maven-plugin` (the run that produced
this revision):

- **Pre-flight check 6 added**: sync local clone with `origin` before reading
  local files. A stale local tracking ref makes `git status` report
  "working tree clean" while the working copy is actually 92 commits
  behind; you optimize against files that are no longer what GitHub is
  running and push a no-op "fix". Always `git fetch origin` first.
- **Pitfall 5 enriched** with a smoke-test variant for when there are no
  open dependabot PRs available. Calling `gh pr merge <N> --auto --rebase`
  against a closed dependabot PR returns `"Pull request is closed"` (not
  a workflow-permission GraphQL error), which still proves the token has
  the required scope. Useful when you want to verify `MYTOKEN` without
  waiting for the next dependabot cycle.
- **New snag added**: adding `groups:` to `dependabot.yml` immediately
  closes the original per-dependency PRs as superseded. This is a free
  way to drain a backlog, but if the new grouped PR is then closed
  without merging, the individual PRs do not reopen — you wait for the
  next weekly cycle. Caveat noted in Snags section.
- **Pre-flight trigger phrase refined**: the smoke-test bullet no longer
  requires an "open" PR (it works on closed PRs too). README trigger
  list gained three new phrases covering the new diagnostics.
- Counts unchanged: still twelve pitfalls, twelve verification checks.
  The changes are diagnostic / snag enrichment, not new failure modes.
- **New snag added**: `prefix-development` + `include: "scope"` produces `build(deps-dev)(deps-dev): ...` for Maven. Maven deps are `dependency-type: direct:development` by default, so both fields apply to the same package; the development prefix and the development scope collide. Fix: use a single `prefix:` and skip `prefix-development` for Maven projects.
- **User OAuth shortcut for `MYTOKEN` verified end-to-end on a workflow-touching PR.** `gh auth token` → `gh secret set MYTOKEN` → first auto-merge workflow run on a PR that bumps `actions/checkout` succeeded with `autoMergeRequest.enabledBy.login: "XenoAmess"`. The PR auto-merged within ~1 minute of the rebase. No separate PAT was needed. The shortcut now has a concrete worked example to point to.

One commit, ~12 insertions (one snag + a worked-example entry), no rewrite.

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
  | `DIRTY` | has a real merge conflict (not just "behind") | **for dependabot PRs: see Pitfall 10 — do NOT push a manual fix**, wait or `@dependabot rebase`. For human PRs: resolve the conflict. |
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

- **`@dependabot rebase` vs `close+reopen` vs `update-branch` API: pick by intent.** All three re-trigger the `pull_request` workflow. They differ in what happens to the PR branch and how fast they act:
  - `gh api -X PUT repos/<owner>/<repo>/pulls/<N>/update-branch` — **rebases the branch onto the current base** and **preserves `autoMergeRequest`** (does not reset it). Requires `repo` scope; admins and many OAuth tokens have it. Fastest of the three: bypasses dependabot's hourly auto-rebase schedule entirely, no comment round-trip. Use this when a dependabot PR is `BEHIND` and you want it unblocked right now without losing its auto-merge setting.
  - `@dependabot rebase` (comment) — **force-pushes** the PR branch onto the latest base, **resets `autoMergeRequest` to null**, then the auto-merge workflow re-runs and re-enables it. Subject to dependabot's hourly rebase schedule; if you comment while one is already queued, the second request may be coalesced. Use when you also want dependabot to regenerate the patch (e.g. you suspect a DIRTY state from a sibling-line conflict that won't be fixed by a clean rebase).
  - `gh pr close <N> --delete-branch=false && gh pr reopen <N>` — **does not touch the branch**. Just re-fires the `pull_request` event. Use when you only want to force the auto-merge workflow to re-evaluate (e.g. you just changed `auto-merge.yml`'s policy and want existing PRs to be judged under the new rules without rebase churn).
  - For draining a `BEHIND` backlog after the migration push, the **`update-branch` API is the right tool** — it keeps `autoMergeRequest` set, so as soon as CI re-runs and passes, the merge fires without a second auto-merge-workflow run. With `@dependabot rebase` you eat the rebase → re-enable → re-run-CI → merge sequence on every PR.

- **Changing `commit-message.prefix` on already-open PRs produces duplicated prefixes**. If a PR's title is already `build(deps): bump foo` and you push a new `dependabot.yml` with `commit-message.prefix: build(deps)`, the next rebase will produce `build(deps)(deps): bump foo`. Cosmetic only — does not break auto-merge. To clean up, `close+reopen` the affected PR; the next dependabot push will regenerate the title under the new prefix.

- **`open-pull-requests-limit: 100` is a foot-gun**. The Dependabot default is 5. Daily schedule + limit 100 = a backlog that never drains and a noisy PR tab. Recommended defaults: `weekly` schedule, limit 5–10 for github-actions, 10–20 for maven. Grouped updates (`groups:` in dependabot.yml) reduce the multiplier — one grouped PR replaces N individual ones.

- **Migration push makes every open Dependabot PR go `BEHIND` at once.** When you push the auto-merge / dependabot.yml / build.yml changes that this skill recommends, master advances by one or more commits. Every open Dependabot PR that was previously `CLEAN` instantly becomes `BEHIND`. With `strict: true` branch protection, auto-merge cannot fire until each one rebases — and Dependabot's auto-rebase is hourly, not instant. Two options: (a) comment `@dependabot rebase` on every open PR in one batch (the Quick Reference batch script filters by author so it skips human PRs), or (b) accept that the queue will drain within ~1h. Plan for this in the migration order: push workflow changes → immediately batch-rebase all open Dependabot PRs → wait for them to settle → run the verification checklist.

- **Adding `groups:` to `dependabot.yml` immediately closes the original per-dependency PRs as superseded.** This is a free way to drain a backlog of N individual PRs into 1 (or a few) grouped PRs. The new grouped PR(s) then proceed through the normal auto-merge path; the original PRs close themselves. Caveat: the close happens the moment the new grouped PR is opened, which is *before* the grouped PR is merged. If for any reason the grouped PR is then closed without merging (e.g. a CI failure or a maintainer close), the individual PRs do not reopen — you have to wait for the next weekly cycle to regenerate them. For low-risk projects this is a clean win; for repos where individual PRs are deliberately being reviewed out-of-band, do not add `groups:` without first merging or closing the existing ones.

- **Grouping in dependabot.yml closes the old individual PRs.** When you add a `groups:` block to an ecosystem that already has individual PRs open, Dependabot on the next cycle closes those individual PRs and opens a single grouped PR with a title like `chore(deps): bump the major group with 2 updates`. This is expected — the old PRs show up as `CLOSED` (not merged), the new grouped PR inherits their changes, and `update-type` for the grouped PR reports the highest-severity bump in the group. Don't panic if your "verified working" PRs vanish from the open list; the replacement is the grouped PR.

- **`AdoptOpenJDK` distribution is being phased out.** `actions/setup-java` with `distribution: adopt` now emits an annotation warning that AdoptOpenJDK has moved to Eclipse Temurin. Recommended: change to `distribution: temurin` in the CI workflow. Non-blocking but visible in the Actions tab.

- **The first dependabot batch tests the hardest path.** If the repo has not been kept up to date, the very first dependabot cycle after enabling this skill will open N major version bumps, most of which touch `.github/workflows/*.yml`. That means the first PRs to be tested are exactly the ones that need the `workflow` scope on `MYTOKEN` and the `gh pr merge --auto` API call. If anything is misconfigured with the token, it shows up immediately. Plan for this: do the Pre-flight 5 scope check *before* the first batch lands, not after.

- **`gh auth refresh -s workflow` hangs in non-interactive CLI sessions.** If you try to add the `workflow` scope to an existing OAuth token from a non-interactive session, the command blocks waiting for a browser confirmation. It will not time out on its own; you have to Ctrl-C it. Either take the user-OAuth-token shortcut (Pitfall 5) and use the token as-is, or ask the user to add the scope via the web UI at `Settings → Developer settings → Personal access tokens → [token] → Edit scopes → Workflow ✓`. Do not retry the refresh in a loop.

- **User OAuth token login normalization is undocumented and may change.** Even when `gh pr list --json author` shows `app/dependabot` for a PR opened by the new Dependabot GitHub App, the value of `github.event.pull_request.user.login` inside the workflow run is normalized to `dependabot[bot]`. This means a workflow gated on the legacy form will silently work on a fully migrated repo. The match-both `||` form (Pitfall 8) is still the right defense — the normalization is undocumented behavior and could be removed in a future GitHub update.

- **`update-branch` API returns 422 when master hasn't moved.** `gh api -X PUT repos/<owner>/<repo>/pulls/<N>/update-branch` only rebases if the base branch has a new commit. If the PR is in the "old auto-merge workflow" state and you want to force a re-evaluation without a rebase, use `gh pr close <N> --delete-branch=false && gh pr reopen <N>` instead — that re-fires the `pull_request: reopened` event with no base-branch motion required. Useful right after you set `MYTOKEN` to a valid value and need the workflow to retry the approve+merge steps that previously failed for the empty-secret reason.

- **`include: "scope"` + `prefix:` collides into `build(deps)(deps): ...`.** When the conventional-commits-style prefix already contains a scope (`build(deps)`, `chore(deps)`, `feat(api)`) and `dependabot.yml` also has `include: "scope"`, the package-manager scope (`deps` for maven, `github-actions` for the github-actions ecosystem) is appended to the prefix, producing a doubled scope. Drop `include: "scope"` from the dependabot config; the package name in the commit body is enough disambiguation.

- **`prefix-development` + `include: "scope"` produces `build(deps-dev)(deps-dev): ...` for Maven.** Maven dependencies are reported as `dependency-type: direct:development` by dependabot, so `prefix-development` and `include: "scope"` both apply to the same package. Setting `prefix-development: "build(deps-dev)"` plus `include: "scope"` yields `build(deps-dev)(deps-dev):` — the development prefix and the development scope collide. For Maven projects, just use a single `prefix:` (e.g. `build(deps)`) and skip `prefix-development`; the per-PR title will be `build(deps)(deps): ...` at worst, which is the same pattern the existing snag above describes. Alternatively keep `prefix-development` but set it to the same value as `prefix` so the collision is invisible.

- **Check name format is `<job-name> (<matrix-axes>)`, not `<workflow> / <job>`.** A common source of duplicate check names when two workflows (`build.yml` and `build_extra.yml`, say) both define a job called `build` with the same matrix. Renaming `name:` on the workflow does not help; the job ID has to differ. See Pitfall 12 for the full pattern and fix.

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

# Faster: force-rebase via API, preserves autoMergeRequest
gh api -X PUT repos/<owner>/<repo>/pulls/<N>/update-branch

# Batch-rebase all open Dependabot PRs (filter by author — never rebase human PRs)
gh pr list --state open --json number,author \
  --jq '.[] | select(.author.login == "app/dependabot" or .author.login == "dependabot[bot]") | .number' | \
  xargs -I {} sh -c 'gh pr comment {} --body "@dependabot rebase"'

# Faster batch rebase via the update-branch API (one HTTP call per PR, no comment queue)
gh pr list --state open --json number,author \
  --jq '.[] | select(.author.login == "app/dependabot" or .author.login == "dependabot[bot]") | .number' | \
  xargs -I {} gh api -X PUT repos/<owner>/<repo>/pulls/{}/update-branch

# Set git remote to SSH (when gh OAuth lacks workflow scope)
git remote set-url origin git@github.com:<owner>/<repo>.git

# Use the existing gh OAuth token as MYTOKEN (when user is admin — Pitfall 5 shortcut)
TOKEN=$(gh auth token)
gh secret set MYTOKEN --repo <owner>/<repo> --body "$TOKEN"

# Smoke-test MYTOKEN scope on a workflow-touching Dependabot PR
gh pr merge <N> --auto --rebase
gh pr view <N> --json autoMergeRequest --jq '.autoMergeRequest.enabledBy.login'
# Expect: a real login (e.g. "XenoAmess"). If null, the token lacks the scope.
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
- "I have a PAT but it's still using `GITHUB_TOKEN`" → check the secret name case (`secrets.mytoken` ≠ `secrets.MYTOKEN`). `gh secret list` shows the actual names. If the user is admin, the simpler fix is to use `gh auth token` directly (Pitfall 5 shortcut).
- "The auto-merge workflow shows `skipped` for the merge step but no error" → Pitfall 6 (`allow_auto_merge` off). Run the run-log diagnostic command in Pitfall 6 to confirm.
- "Auto-merge workflow never runs at all on Dependabot PRs" → Pitfall 8 (login mismatch). Check `author.login` on the open PRs and update the workflow's `if:` line.
- "I bumped the JDK / OS in build.yml matrix and now PRs are BLOCKED" → Pitfall 9. Re-run Step 4 with new check names from GraphQL, then PUT branch protection again.
- "All my Dependabot PRs went BEHIND at once after I pushed the workflow changes" → migration push snag. Batch `@dependabot rebase` on all open Dependabot PRs (Quick Reference).
- "My old individual PRs disappeared when I added groups" → expected. They were closed; the grouped PR replaces them.
- "CI shows green on the PR but the auto-merge workflow isn't being gated by it" → Pitfall 2 subtle variant. The CI is probably being triggered by `push` only, not `pull_request`. Confirm with `gh run list --workflow=<build> --json event` and fix the `on:` block.
- "I have `gh` authenticated but I don't want to create a separate PAT" → Pitfall 5 user-OAuth-token shortcut. Use `gh auth token` directly as `MYTOKEN`; verify with the smoke test in Quick Reference.
- "A dependabot PR is stuck at `DIRTY` (not BEHIND) after a sibling PR merged" → Pitfall 7 adjacent-line variant + Pitfall 10. Wait or `@dependabot rebase`; do NOT push a manual fix.
- "I tried `gh auth refresh -s workflow` and it just hangs" → see the anti-pattern note in Pitfall 5. Use the user-OAuth-token shortcut, or ask the user to add the scope via the web UI.
- "The first dependabot PRs are all major version bumps touching workflow files and the first one failed" → this is the worst-case test path. Re-check Pre-flight 5 (token scope), Verification check #8/#11, and Pitfall 5. The "first batch is hardest" snag explains why.
