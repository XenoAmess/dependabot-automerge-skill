---
name: dependabot-automerge-skill
description: Set up or repair GitHub Dependabot auto-merge for a repository. Use when the user mentions dependabot, auto-merge, dependabot PR stuck, semver-major merging, GitHub Actions PRs not auto-merging, branch protection required checks, allow_auto_merge disabled, wants to reduce manual PR churn, or wants to optimize dependabot. Triggers on phrases like "set up dependabot auto-merge", "optimize dependabot", "PR stuck waiting for checks", "auto-merge not waiting for CI", "major version dependabot", "branch protection required status check", "PR stuck BEHIND", "PR stuck DIRTY after a sibling dependabot PR merged", "PR stuck DIRTY after I rewrote auto-merge.yml", "auto-merge returns 422", "oauth app cannot create workflow", "auto-merge workflow never runs", "app/dependabot vs dependabot[bot]", "I bumped the JDK matrix and now auto-merge is broken", "dependabot PRs stuck after I changed build.yml", "CI looks like it's running but isn't gating the PR", "all my dependabot PRs went BEHIND at once", "I have gh but I don't want to create a separate PAT", "rebase produced a real conflict not just BEHIND", "first batch of dependabot PRs are all major version bumps touching workflow files", "MYTOKEN secret is set but auto-merge still fails with empty GH_TOKEN", "I set MYTOKEN but it's empty inside the auto-merge workflow even though gh secret list shows it", "dependabot PR workflow can't read any custom secrets only GITHUB_TOKEN", "actions secret vs dependabot secret I used gh secret set without --app dependabot", "two workflows produce the same check name and branch protection is ambiguous", "dependabot.yml produces double prefix like build(deps)(deps)", "dependabot PR title is build(deps-dev)(deps-dev) maven double prefix", "dependabot grouped my major upgrades into one huge PR that broke three things at once", "patterns: [\"*\"] in my dependabot.yml groups everything into a single PR", "drop the groups: block from dependabot.yml", "one PR per dependency please, not one PR per ecosystem", "auto-merge workflow is green but the PR never merges", "auto-merge wrapper does nothing / swallowed error", "dependabot did not auto-merge", "fetch-metadata refused the commit signature", "dependabot labels could not be found", "create missing labels for dependabot", "my dependabot is configured but no PRs appear", "dependabot is silent / no PRs are opening". Does NOT use when the user only wants to configure dependabot.yml update schedule, or only wants to enable dependabot security updates without auto-merge.
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
- "PR is stuck at `DIRTY` after I rewrote auto-merge.yml / dependabot.yml"
- "`gh pr merge --auto` returns 422 / "Auto merge is not allowed""
- "auto-merge says 'OAuth App cannot create or update workflow'"
- "I changed auto-merge.yml and nothing happened"
- "dependabot is open but not being merged"
- "auto-merge workflow runs but does nothing / always skipped"
- "Dependabot PRs are bot authored but my workflow never fires"
- "dependabot PRs are stuck after I changed build.yml"
- "branch protection required status check name no longer matches"
- "CI looks like it's running but isn't gating the PR"
- "I bumped the JDK matrix and now auto-merge is broken"
- "dependabot PR went DIRTY (not BEHIND) after a sibling PR merged"
- "I have `gh` authenticated but I don't want to create a separate PAT"
- "the first dependabot PRs are all major version bumps touching workflow files"
- "MYTOKEN is set but `gh pr review` still errors with empty GH_TOKEN in the log"
- "I set MYTOKEN but it's empty inside the auto-merge workflow even though `gh secret list` shows it"
- "dependabot PR workflow can't read any custom secrets, only GITHUB_TOKEN"
- "actions secret vs dependabot secret — I used `gh secret set` without `--app dependabot`"
- "two CI workflows produce the same `build (os, java)` check name"
- "dependabot PR titles look like `build(deps)(deps): ...` with duplicated scope"
- "dependabot grouped my major upgrades into one huge PR that broke three things at once"
- "patterns: `[\"*\"]` in my dependabot.yml groups everything into a single PR"
- "drop the `groups:` block from dependabot.yml"
- "one PR per dependency please, not one PR per ecosystem"
- "dependabot labels could not be found / create missing labels for dependabot"
- "my dependabot is configured but no PRs appear"
- "dependabot is silent / no PRs are opening"

Do **not** use this skill for:

- Just editing `.github/dependabot.yml` schedule/ecosystem config (no auto-merge involved).
- Dependabot security updates without auto-merge.
- Non-Dependabot PR auto-merge (e.g. `bors`, merge queues, custom bots). Different problem.

---

## Pre-flight checks

Before changing anything, gather context. Do not skip these.

> **Default branch**: this skill uses `master` as a shorthand example. Discover the repo's actual default branch (`develop`, `main`, etc.) and substitute it for every `master` reference below.
>
> ```bash
> gh api repos/<owner>/<repo> --jq '.default_branch'
> ```

1. **Read the current state**:
   - `.github/dependabot.yml` — which ecosystems are configured? schedule? PR limit? groups? labels?
   - `.github/workflows/` — is there already an `auto-merge.yml`? what is its `if:` line?
   - If `build.yml` exists, what is the `on:` block? Does it include `pull_request`?
2. **Check that every label referenced in `dependabot.yml` exists in the repo, create the missing ones.** If `dependabot.yml` lists labels under `updates[].labels:` (e.g. `dependencies`, `gradle`, `github-actions`) that don't yet exist in the repo, Dependabot will refuse to open PRs and surface the error `The following labels could not be found: <name>. Please create it before Dependabot can add it to a pull request.` — at which point every PR for that ecosystem is silently dropped. Diff the in-config labels against the repo's actual label list and create the missing ones in one batch before pushing any dependabot.yml change:

   ```bash
   # Extract the labels block from each ecosystem entry in dependabot.yml
   CONFIG_LABELS=$(awk '/^[[:space:]]*-/{flag=1} flag && /^[[:space:]]*labels:/{getline; while ($0 ~ /^[[:space:]]*-/){gsub(/^[[:space:]]*-[[:space:]]*"|"$/,""); print; if (getline <=0 || $0 !~ /^[[:space:]]*-/) break}}' \
     .github/dependabot.yml | sort -u)

   EXISTING=$(gh label list --limit 200 --json name --jq '.[].name' | sort -u)

   missing=()
   while read -r l; do
     [[ -z "$l" ]] && continue
     grep -qxF "$l" <<<"$EXISTING" || missing+=("$l")
   done <<<"$CONFIG_LABELS"

   if (( ${#missing[@]} > 0 )); then
     echo "Creating missing labels: ${missing[*]}"
     for l in "${missing[@]}"; do
       # Sensible defaults — adapt color/description as you see fit
       gh label create "$l" --color "0366d6" --description "Dependabot updates" \
         || { echo "failed to create $l" >&2; exit 1; }
     done
   fi
   ```

   **This must run *before* you push any dependabot.yml change**, or the first dependabot cycle after the push will drop every PR for the ecosystems whose labels were missing. See Pitfall 17.
3. **Check the GitHub repo** (use `gh` CLI; the user must be authenticated):
   ```bash
   gh auth status
   gh api repos/<owner>/<repo> --jq '.default_branch'
   gh api repos/<owner>/<repo>/branches/master/protection
   gh api repos/<owner>/<repo> --jq '.allow_auto_merge'   # see Pitfall 6
   ```
4. **Check if there are stuck PRs** that motivated the request:
   ```bash
   gh pr list --state open --json number,title,headRefName,mergeStateStatus,author
   ```
    **Also note the `author.login`** — GitHub migrated Dependabot in 2024. New PRs are authored by `app/dependabot`; legacy PRs (and any in repos that haven't migrated) are authored by `dependabot[bot]`. Your auto-merge workflow's `if:` line must match the one the repo actually uses — see Pitfall 8.
5. **Read the actual check names** that GitHub is generating, not what you assume. See Pitfall 3.
6. **Inventory available tokens and verify the correct secret namespace**:
   ```bash
   gh secret list                  # actions secrets only
   gh secret list --app dependabot # dependabot secrets (often the real problem)
   gh auth token | cut -c1-10      # check the prefix — gho_ (OAuth) vs ghp_ (classic PAT) vs github_pat_ (fine-grained)
   ```

   **Two paths, pick the easier one**:

   - **If the user is the repo admin and `gh` is authenticated** — the existing user OAuth token (`gho_*` prefix) has implicit `workflow` scope for repos where the user is admin. You can use it directly as `MYTOKEN`. See Pitfall 5 for the smoke-test and the `gh auth refresh` anti-pattern.
   - **Otherwise** — you need a separate PAT (classic with `repo` + `workflow` scope, or fine-grained with `Contents: write` + `Pull requests: write` + `Actions: write`). Ask the user to create one and store it as a repo secret.

   The classic GITHUB_TOKEN supplied to a workflow is never sufficient — it cannot enable auto-merge on PRs that modify `.github/workflows/*.yml`. See Pitfall 5.

If the repo does not have `gh` auth, stop and ask the user to log in. Do not guess at branch protection.

7. **Sync local clone with `origin` before reading local files.** A local clone left over from a previous session can be many commits behind `origin/master` (especially on repos where dependabot has been auto-merging regularly). `git status` reports "nothing to commit, working tree clean" in that state because it is comparing against the stale local tracking ref — the staleness only shows up after `git fetch`. Always run `git fetch origin` (or `git pull --ff-only`) before reading `.github/workflows/*.yml` from disk. Otherwise you will optimize against files that are no longer what GitHub is running, push a "fix" that is actually a no-op, and wonder why nothing changed.

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

Auto-merge works without touching `dependabot.yml`, but the user will still drown in noise unless you also do these three things. Each one collapses PR *churn* or clarifies intent:

1. **Weekly schedule, not daily.** `interval: daily` produces a PR per bump per day; `weekly` produces one batched bump on a predictable day.
2. **`open-pull-requests-limit` of 5–20.** Default is 5. `100` is a foot-gun — it lets Dependabot open far more PRs than the auto-merge pipeline can drain, which increases noise and the `BEHIND` rebase race (Pitfall 7). Pick a number your CI matrix can actually clear in a weekly cycle: `10` for the main maven root, `5` for smaller ecosystems. See Snags.
3. **`commit-message.prefix` and `labels`** so the auto-merge workflow (and humans) can identify dependabot PRs at a glance.

**Do not** add a `groups:` block. Grouping — particularly `patterns: ["*"]` — collapses N unrelated dependency bumps into a single multi-hundred-line PR. When that PR fails CI, you cannot tell which bump caused it; when it passes, the diff is too large to review in one sitting. Major-version PRs in particular become unreviewable — the auto-merge policy in Step 1 says "do not auto-merge major", so the giant PR sits open until a human can pick it apart. The default behaviour (one PR per dependency per cycle) keeps each diff small enough to read, bisect and revert in isolation, at the cost of a higher PR count. That trade-off is the right one. See Pitfall 13.

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
    labels:
      - "dependencies"
      - "java"

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
      prefix: "ci"
    labels:
      - "dependencies"
      - "github-actions"
```

Why this works:

- **No `groups:` block** means one PR per dependency per cycle. Each PR is one bump, one diff, one review. The `prefix:` + `labels` keep the stream scannable so the higher PR count is not a navigation problem.
- **`include: "scope"` is omitted** — it collides with `prefix:` to produce `build(deps)(deps): ...` (see Snags below).
- **Same `day: monday` for all ecosystems** means the user sees one batch of dependabot activity per week, not a constant drip.
- **For per-PR auto-merge policy**, the auto-merge workflow's existing `update-type` check (Step 1) reads the bump's `semver-*` classification directly from dependabot's metadata — that already works correctly without `groups:`.

### Step 7 — Skill update + per-project notes (required deliverable, not polish)

After all sixteen Verification checks pass, before reporting success. This step is a **deliverable**, not meta-cleanup: the user is paying for the optimized repo *and* an improved skill. Skipping this step means the next repo you optimize will hit the same traps, and the user has historically had to prompt for it (see the Worked-Example entry for `XenoAmess/XenoAmessBlog` and the "Final checklist before reporting done" section below).

1. **Write a per-project notes file** at `<project>/docs/dependabot-optimization-notes.md` (see the template in the Self-improvement loop section below). This is for the project owner — it documents what changed in *their* repo and why.
2. **Update this skill** (`SKILL.md` + `README.md`) with anything new you learned this run. See the Self-improvement loop section below for what qualifies and how to do it.
3. **Commit the skill changes locally**. Do **not** push — the skill is loaded from a local path and has no remote.
4. **Do not commit the project notes file** unless the user asks. The notes are useful as a local artifact and a project record; committing is the user's call.

---

## Pitfalls (read these before declaring success)

These are the things that have actually broken in production. Re-check each one before finishing.

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

**Fix**: Use a repo PAT with `repo` + `workflow` scope for the `gh pr review` and `gh pr merge` steps. `fetch-metadata` is read-only and can keep using `GITHUB_TOKEN`. Configure via `gh secret set MYTOKEN --repo <owner>/<repo> --app dependabot --body "$TOKEN"` (use whatever name you want; remember the case and the `--app dependabot` namespace).

**Diagnostic**: If you're not sure which token the action is using, check the secret name. `${{ secrets.mytoken }}` (lowercase) and `${{ secrets.MYTOKEN }}` (uppercase) are different secrets — names are case-sensitive. An undefined secret silently falls back to `GITHUB_TOKEN` and produces this exact error.

**The `--app dependabot` requirement**: Even if you set a secret with the right value and the right scope, a dependabot-triggered workflow run can only read it when the secret is in the **dependabot** namespace. `gh secret set MYTOKEN` (without `--app`) writes to the actions namespace; the runner on a dependabot PR will see an empty value. See Pitfall 16 for the exact symptoms and fix.

**Empty-secret detection in the workflow log**: when the secret exists in the repo's secret list but holds no value, the workflow log shows `GH_TOKEN: ` (truly empty — colon followed by nothing), not `GH_TOKEN: ***` (masked). The trailing colon with no asterisks is the giveaway. The `gh pr review` step then errors with `gh: To use GitHub CLI in a GitHub Actions workflow, set the GH_TOKEN environment variable`. Fix: `gh secret set MYTOKEN --repo <owner>/<repo> --app dependabot --body "$TOKEN"`. To inspect, run `gh run view <id> --log-failed` and look at the env block of the failing step.

**Shortcut — user OAuth token, no separate PAT needed**: If the user invoking this skill is the repo admin and already has `gh` authenticated, you do not need a separate PAT. A user OAuth token (`gho_*` prefix) has implicit `workflow` scope for repos where the user is admin — it can approve and enable auto-merge on PRs that modify workflow files. Use it directly, and store it in the **dependabot** namespace so dependabot-triggered workflow runs can read it:

```bash
TOKEN=$(gh auth token)
gh secret set MYTOKEN --repo <owner>/<repo> --app dependabot --body "$TOKEN"
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

**Device-flow variant (works if the user is present)**: if the smoke test fails and the user can open a browser, run `gh auth refresh -h github.com -s workflow`. The CLI prints a one-time code and a `https://github.com/login/device` URL. After the user completes the device-flow authorization in the browser, the same `gh auth token` value will have the `workflow` scope for workflow-touching PRs. Note: `gh auth status` may continue to list the original scopes (`admin:public_key`, `gist`, `read:org`, `repo`) and omit `workflow` even though the token now works. Trust the smoke test (`gh pr merge <N> --auto --rebase` on an open workflow-touching Dependabot PR), not the scopes list.

**Dependabot-secret gotcha — `--app dependabot` is required, not optional**: This is the single most common reason the above setup "looks right but does nothing". `gh secret set MYTOKEN` (without `--app`) creates an **actions** secret. The actions secret is available to ordinary `on: push` / `on: workflow_dispatch` workflows, but is **NOT** available to the runner when the workflow is triggered by a Dependabot PR — GitHub treats Dependabot PRs as semi-trusted (similar to forks) and refuses to inject any custom secrets other than `GITHUB_TOKEN`. A separate secret namespace exists for Dependabot, set with `--app dependabot`. Symptom: the auto-merge workflow log shows `GH_TOKEN: ` (truly empty — no asterisks, no value) in the env block, the run fails with `gh: To use GitHub CLI in a GitHub Actions workflow, set the GH_TOKEN environment variable`, and the debug `echo "${#GH_TOKEN}"` prints `0`. The smoke-test command (`gh pr merge <N> --auto --rebase` from your shell) succeeds because *your* token has the scope — only the workflow run sees the empty value. Fix:

```bash
TOKEN=$(gh auth token)
gh secret set MYTOKEN --repo <owner>/<repo> --app dependabot --body "$TOKEN"
```

Verify it landed in the dependabot namespace (default `gh secret list` only lists *actions* secrets and will silently miss it):

```bash
gh secret list --repo <owner>/<repo> --app dependabot
```

Then trigger a fresh workflow run (e.g. `gh pr comment <N> --body "@dependabot recreate"` on an open Dependabot PR). The env block in the next run should show `GH_TOKEN: ***` instead of empty, and the auto-merge step will succeed.

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

**Stale-base variant — `@dependabot rebase` says "The base commit has not changed" while GitHub still shows BEHIND**: a subtler case observed after a migration push. You post `@dependabot rebase`, and dependabot replies with "The base commit for this pull request has not changed." But `gh pr view <N> --json mergeStateStatus` clearly says `BEHIND`. Cause: dependabot's *internal* view of the base is whatever it last synced from master — typically the SHA when the PR was opened. If master has since moved (e.g. you pushed the auto-merge/dependabot.yml changes from this skill), dependabot's view of the base is stale, and the rebase command short-circuits with that message. GitHub's merge status, by contrast, compares against the live `master` HEAD, so it correctly reports BEHIND. The fix is `@dependabot recreate` (not `rebase`) — recreate forces a fresh commit generated against the current master, which then sets the base to the live HEAD and lets the auto-merge workflow re-enable auto-merge on a CLEAN diff. Useful one-liner:

```bash
# For a PR stuck BEHIND where @dependabot rebase says "no change":
gh pr comment <N> --body "@dependabot recreate"
```

After recreate, expect: (1) dependabot pushes a new commit, (2) CI re-runs, (3) auto-merge workflow re-runs and re-enables auto-merge, (4) merge fires. This is the same wait as the rebase variant above, but it bypasses the stale-base refusal.

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

**Protocol-level reinforcement — `dependabot/fetch-metadata@v3` verifies the git signature, not just the commit author**: even `--amend --no-edit` (which preserves the `Author:` and `Signed-off-by:` trailer and looks correct in `git log`) breaks the commit's *git signature*, because amend produces a new commit object with a new hash. `dependabot/fetch-metadata` calls GitHub's signature-verification API on the commit; when the verification fails, the step emits:

```
Dependabot's commit signature is not verified, refusing to proceed.
PR is not from Dependabot, nothing to do.
```

…and exits 0. The auto-merge workflow's overall run reports `success` (because the step didn't *fail* — it just declined), but no `should_merge` decision is ever made, no approval is recorded, and no auto-merge is enabled. If you ever reach for "let me just amend a trailing whitespace fix and force-push to unstick the rebase", expect the auto-merge workflow to silently no-op afterwards. Diagnostic: `gh run view <id> --log-failed | grep -F 'commit signature'`. Fix: do not amend. Wait or use `@dependabot recreate`.

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

### Pitfall 13 — `groups:` with `patterns: ["*"]` bundles every upgrade in that ecosystem into one PR

**Symptom**: The next weekly dependabot cycle opens a single PR titled something like `build(deps): bump the maven-major group with 3 updates`, and that PR contains three completely unrelated dependency upgrades (e.g. `maven-release-plugin 2.5.3→3.3.1`, `maven-surefire-plugin 2.22.2→3.5.6`, `jaxb2-maven-plugin 1.6→2.5.0`). When the CI fails, the failure log is huge and you cannot tell which of the three bumps caused it; when CI passes, the diff is too large to review in one sitting. Major-version PRs in particular are unreviewable: the policy in Step 1 says "do not auto-merge major", so the PR is left for a human, and the human is asked to review a diff that spans three plugins and their configs.

**Cause**: The `groups:` block in `dependabot.yml` uses `patterns: ["*"]` to mean "match every dependency in this ecosystem". Combined with `update-types: ["major"]` (or `["minor", "patch"]`), it tells dependabot to open **one PR per group, per cycle**, containing every available upgrade. The previous version of this skill (Step 6) explicitly recommended this pattern, on the rationale that "N updates → 1 PR" reduces noise. In practice it just moves the noise from "many small PRs" to "one giant PR", and makes failures un-attributable.

**Fix**: drop the `groups:` block entirely from every ecosystem entry. With no `groups:`, dependabot falls back to its default behaviour: one PR per dependency per cycle. The per-ecosystem `commit-message.prefix` and `labels` still make the PR stream scannable; the PR *count* goes up but each PR's *diff* stays small enough to read, bisect and revert in isolation. See the corrected Step 6 recipe for the exact shape.

**Why this deserves its own pitfall**: the previous recipe in this skill is what produced PRs of this shape. Anyone who followed it is sitting on a backlog of giant grouped PRs that are hard to review and hard to debug. The recipe looked like a free win (collapse N PRs into 1), but the failure mode (a 3-bump PR that breaks three things at once) is the exact opposite of what dependabot is supposed to provide. If you already have `groups:` with `patterns: ["*"]` in your config, drop them on the next push — the next cycle will reopen the individual PRs, each with the right `update-type` so the auto-merge policy picks them up correctly.

### Pitfall 14 — Pushing a fix to `auto-merge.yml` does not re-evaluate existing stuck PRs

**Symptom**: You push a fix to `.github/workflows/auto-merge.yml` to address Pitfall 1 (add `semver-major`), Pitfall 5 (switch to `MYTOKEN`), Pitfall 8 (add `app/dependabot` login), or any other workflow bug. The new workflow is now active on master. But the **existing** dependabot PRs that were stuck under the old workflow are still stuck — they show `autoMergeRequest: null` and `mergeStateStatus: CLEAN` (green CI, no auto-merge). Hours pass. Nothing changes. The fix appears to have done nothing.

**Cause**: The auto-merge workflow has `on: pull_request` — it runs when a pull request is `opened`, `synchronize`d, or `reopened`. Editing the workflow file in master does **not** synchronize existing PRs; it only changes the workflow definition for *future* events on those PRs. An existing stuck PR with no new commits on its head branch will not fire the new workflow, ever, until something pushes to that branch.

**Fix**: Comment `@dependabot rebase` on each open dependabot PR you want re-evaluated. This forces dependabot to push a new commit (or `@dependabot recreate` if rebase cannot apply cleanly because of a DIRTY conflict with a sibling PR that just merged — see Pitfall 7 adjacent-line variant). Either action fires a `synchronize` event on the PR, which runs the new workflow.

```bash
for n in $(gh pr list --state open --author app/dependabot --json number --jq '.[].number'); do
  gh pr comment "$n" --body "@dependabot rebase"
done
```

**Diagnostic to confirm the new workflow fired**:

```bash
gh run list --workflow="Dependabot auto-merge" --limit 10 --json headBranch,conclusion,createdAt
```

You should see a fresh run for each rebased PR's `headRefName` with a `createdAt` within the last few minutes. The per-step conclusions of that new run should show `Approve dependabot PR` and `Enable auto-merge` as `success` (not `skipped`) — `skipped` means the new workflow still classified the PR as ineligible (which is a different problem; re-check Pitfall 1's policy table).

**Why this deserves its own pitfall**: any time you change `auto-merge.yml` — whether to fix a login mismatch, add a `semver-major` branch, or switch tokens — the fix is invisible to existing PRs until they get a new commit. The "push a fix and watch it work" mental model applies to the next *new* dependabot PR, not to the ones already on the PR list. Forgetting this is the difference between "the optimization worked" and "the optimization worked for next week but my PRs from last week are still stuck". This pitfall is what makes the other pitfalls tractable in a single optimization session instead of "wait for the next cycle".

**Adjacent side-effect — dependabot.yml changes retitle existing PRs**: when you change `commit-message.prefix`, `labels`, or other dependabot.yml fields in the same change as the auto-merge workflow, the `@dependabot rebase` cycle may retitle existing open PRs to match the new prefix. Observed in a maven + github-actions project: a PR originally titled `build(deps): bump actions/cache from 5 to 6` became `ci: bump actions/cache from 5 to 6` after the new prefix was applied, because dependabot regenerated the title during rebase. This is expected dependabot behavior (the title is computed from current config), but it means:

- Notification rules that filter by title (`build(deps):`) will stop firing for the retitled PRs.
- The PR list briefly looks inconsistent — some old-title, some new-title — until every open PR gets touched.
- Labels added in dependabot.yml only get applied to PRs that are recreated or rebased after the change; PRs already merged are unaffected.

Side-effect mitigation: create the new labels in the repo *before* pushing the dependabot.yml change, so the retitled PRs pick them up cleanly on the first rebase. Notify the team that open PR titles will change. Old PRs that are already merged keep their original title forever.

### Pitfall 15 — Rewriting `auto-merge.yml` (or any workflow it uses) creates `DIRTY` conflicts on open Dependabot PRs that bump that workflow

**Symptom**: A Dependabot PR that bumps `actions/checkout` (or `actions/setup-node`, or any action referenced in `.github/workflows/auto-merge.yml`) is open and `BEHIND`. You rewrite `auto-merge.yml` on master (e.g. drop the `Checkout code` step, swap `dependabot/fetch-metadata@v2` for `@v3`, switch from `GITHUB_TOKEN` to `secrets.MYTOKEN`). You comment `@dependabot rebase` on the open PR. Within a minute, the PR's `mergeStateStatus` flips from `BEHIND` to `DIRTY` and `mergeable: CONFLICTING`. The Dependabot comment confirms the rebase failed. `@dependabot rebase` again → still DIRTY.

**Cause**: A `DIRTY` state is a real merge conflict, not just a stale base. The Dependabot PR's diff tries to change a line that no longer exists in the post-rewrite file (or tries to insert a line where the surrounding context has shifted). This is structurally similar to Pitfall 7's adjacent-line variant, but the trigger is different: there, two parallel PRs race on adjacent lines; here, your own master-side rewrite deletes or moves the target line. `git rebase` regenerates the diff against the new base; the regenerated patch's `@@` context no longer matches the rewritten file, so it cannot apply.

**Fix**: `@dependabot recreate` (not `rebase`). `recreate` regenerates the *entire* commit from current dependabot metadata against current master — it does not try to replay the old patch. The new commit's diff is computed against the live file, so a removed step simply doesn't appear in the diff at all. For a `actions/checkout` 6→7 bump across three workflow files where one of the workflows no longer has the line, recreate produces a two-file diff instead of a three-file conflict.

```bash
gh pr comment <N> --body "@dependabot recreate"
```

Wait for the `synchronize` event, confirm CI re-runs against the new commit, and let auto-merge proceed normally.

**Avoidance — order of operations**: if you are about to rewrite `auto-merge.yml` *and* there is an open Dependabot PR that bumps any action used in `auto-merge.yml`, two safe orderings:

1. **Let the bump merge first**: keep the rewrite of `auto-merge.yml` on a local branch until the Dependabot bump auto-merges into master. Then rebase your local branch on top of the post-bump master and push. The rewrite lands on top of the bumped versions; no conflict.
2. **Push the rewrite first, then `@dependabot recreate` each affected PR**: this is what the standard optimization flow ends up doing (you cannot avoid pushing the new `auto-merge.yml` to fix the auto-merge bug that the bump itself exposed). Plan to comment `@dependabot recreate` on each open workflow-touching Dependabot PR as part of the migration push, not as a follow-up.

**Diagnostic — distinguishing this from Pitfall 7's stale-base variant**:

```bash
# 1. Is the PR in DIRTY?
gh pr view <N> --json mergeStateStatus,mergeable
# mergeable: CONFLICTING + mergeStateStatus: DIRTY → real conflict, recreate is the right tool

# 2. Did the rebase happen at all?
gh pr view <N> --json commits --jq '.commits[-1].committedDate'
# If the committedDate is older than your workflow push, dependabot hasn't tried yet — wait.

# 3. Is the conflict on a line your rewrite changed?
gh pr diff <N>
# Look for `@@` hunks that reference content you removed in your rewrite.
```

**Why this deserves its own pitfall**: the rewrite is often *the same change* that the optimization skill is making (replace `GITHUB_TOKEN` with `MYTOKEN`, add a `semver-major` branch, drop `actions/checkout@v6`). The bump PR being "stuck at DIRTY" after a successful-looking migration push feels like a separate failure ("rebase is broken"), but it is a direct consequence of the migration push itself. Without this pitfall, the next agent will debug it as a rebase problem and waste cycles on `@dependabot rebase` loops. With it, they recognize the shape immediately and reach for `@dependabot recreate`.

### Pitfall 16 — `gh secret set MYTOKEN` creates an *actions* secret, not a *dependabot* secret

**Symptom**: Auto-merge workflow fails with empty `GH_TOKEN` (and any other custom secret). The env block in the workflow log shows `GH_TOKEN: ` (truly empty, no asterisks, no value), not `GH_TOKEN: ***` (masked). `gh: To use GitHub CLI in a GitHub Actions workflow, set the GH_TOKEN environment variable`. Adding debug `echo "${#GH_TOKEN}"` to the step prints `0`. The token is set correctly per `gh secret list` and the smoke test from your local shell (`gh pr merge <N> --auto --rebase`) works fine.

**Cause**: GitHub has two separate secret namespaces per repo: **actions** secrets (default) and **dependabot** secrets. `gh secret set <NAME>` (without `--app`) writes to the *actions* namespace. The runner triggered by a Dependabot PR only receives `GITHUB_TOKEN` — it does **not** receive any *actions* secrets. A *dependabot* secret is required to inject custom tokens into dependabot PR workflows. This was added to the skill because it is the single most common reason a "correctly set up" repo still fails on the very first workflow run, and it is invisible until you actually look at the env block in the run log (which shows empty rather than masked, masking the difference from "secret doesn't exist").

**Fix**: Re-set the secret with `--app dependabot`:

```bash
TOKEN=$(gh auth token)  # or your PAT
gh secret set MYTOKEN --repo <owner>/<repo> --app dependabot --body "$TOKEN"

# Verify it landed in the dependabot namespace (default list omits it!)
gh secret list --repo <owner>/<repo> --app dependabot
```

Then trigger a fresh workflow run on an open dependabot PR:

```bash
gh pr comment <N> --body "@dependabot recreate"
```

In the next run's env block, `GH_TOKEN: ***` should now appear (masked = non-empty). The auto-merge step will succeed.

**Diagnostic that distinguishes this from Pitfall 5 (token scope)**:

| Symptom | Cause |
|---|---|
| `GH_TOKEN: ***` (masked) in env, step fails with `enablePullRequestAutoMerge GraphQL` | Pitfall 5 — token scope wrong |
| `GH_TOKEN: ` (truly empty) in env, step fails with `set the GH_TOKEN environment variable` | Pitfall 16 — wrong secret namespace |

The masking indicator (`***` vs empty) is the smoking gun. Look at the env block in `gh run view --log` for the failing step.

**Why this deserves its own pitfall**: the default-`gh secret set` path is so natural that it took the optimization flow all the way through to the smoke test passing before it became apparent that the workflow run saw something different from the local CLI. Without this pitfall, the next agent will re-run `gh secret set MYTOKEN --body "$TOKEN"` three or four times, conclude it must be a value-length or whitespace issue, and burn an hour before checking `--app`. With it, the fix is the first thing to try when the env block shows empty rather than masked. The remediation is one flag; the diagnostic is two characters in the run log (`***` vs ``).

**Adjacent trap — `gh secret list` only shows actions secrets by default**: this is what makes the bug invisible. `gh secret list` returns `[{"name":"DEBUG_KEYSTORE_BASE64",...}, {"name":"MYTOKEN",...}]` whether the secret is in the actions namespace (invisible to dependabot) or the dependabot namespace (visible to dependabot). Always pass `--app dependabot` when verifying dependabot-secret setup.

### Pitfall 17 — Labels referenced in `dependabot.yml` must already exist in the repo, or Dependabot silently drops every PR for that ecosystem

**Symptom**: Dependabot's "Pull requests" tab (under Insights → Dependency graph → Dependabot, or visible in the `gh` CLI output) shows the error:

```
The following labels could not be found: <name>. Please create it
before Dependabot can add it to a pull request. Please fix the above
issues or remove invalid values from dependabot.yml.
```

No PRs are opened for the affected ecosystem until the label exists. Existing open PRs in that ecosystem are unaffected — only *future* PRs from the ecosystem whose `labels:` list references a non-existent name are dropped. The error is surfaced in the Dependabot logs (and as a check-run error on the last PR dependabot tried to open) but does not fail any workflow, so it is silent unless someone looks at the Dependabot tab.

**Cause**: `dependabot.yml` lets each ecosystem entry declare `labels: [<name>, ...]`. Dependabot applies those labels to every PR it opens in that ecosystem. If a label doesn't exist at the moment dependabot tries to apply it, the whole PR is rejected — silently, since dependabot just doesn't open it. Common sources of the problem:

- A repo is being onboarded to dependabot and `dependabot.yml` references labels that the repo admin never created.
- The skill's Step 6 recipe (or some other config copy-paste) lists `dependencies`, `<ecosystem-name>`, and any per-ecosystem labels that the receiving repo doesn't already have.
- Labels were renamed/archived in the repo's label list while `dependabot.yml` still points at the old name.
- The repo was transferred to a new org and the labels didn't migrate with it.

**Fix**: every label listed under any `updates[].labels:` block must exist in the repo *before* the dependabot cycle runs. Diff the in-config labels against `gh label list`, then create the missing ones:

```bash
# Extract in-config labels and diff against the repo's actual label set.
# The awk fallback below works without yq.
yq '.updates[].labels[]' .github/dependabot.yml 2>/dev/null \
  | sort -u > /tmp/want.txt
gh label list --limit 200 --json name --jq '.[].name' | sort -u > /tmp/have.txt
comm -23 /tmp/want.txt /tmp/have.txt > /tmp/missing.txt

while read -r label; do
  [[ -z "$label" ]] && continue
  gh label create "$label" --color "0366d6" \
    --description "Dependabot updates" || echo "create failed: $label"
done < /tmp/missing.txt
```

Pre-flight step 2 in this skill runs the same diff (using an `awk` extraction so it doesn't depend on `yq`) and creates any missing labels before pushing the new `dependabot.yml`. Adopt that step as part of the standard flow; do not skip it.

**Verification**: after creating the labels, the next dependabot cycle will successfully open PRs for the ecosystem. Two diagnostics to confirm the fix landed:

```bash
# 1. The Dependabot tab no longer shows the labels error
gh api repos/<owner>/<repo>/dependabot/alerts 2>/dev/null

# 2. The next cycle produces the expected PR count
gh pr list --state all --author app/dependabot --json number \
  | jq 'length'
```

**Why this deserves its own pitfall**: the error is invisible until a human happens to look at the Dependabot tab. It will not produce a workflow run, a failed CI check, an email, or anything else that pings the maintainer. The "PRs aren't being opened" symptom is easy to misdiagnose as "the schedule is wrong" or "the wrong branch is targeted" or even "dependabot is broken at GitHub". In practice it is the *very first* thing to check when dependabot is configured correctly but no PRs appear after a few days. The fix is one bash loop; the diagnostic is a 5-character grep (`comm -23`). Without it, the next agent burns half an hour investigating non-existent CI/schedule problems.

---

## Verification (mandatory before reporting done)

Do not tell the user "done" until all sixteen checks pass. After that, do not report "done" without also completing Step 7 and running the **Final checklist before reporting done** section below — both halves of the deliverable must be visible in the same reply.

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
13. **No `groups:` with `patterns: ["*"]` in `dependabot.yml`**: `cat .github/dependabot.yml` should not contain `patterns: ["*"]` anywhere. If it does, Pitfall 13 applies — the next weekly cycle will open a single multi-dependency PR, and any failure in that PR is un-attributable to a specific bump. Drop the `groups:` block on the next push; the next cycle will reopen the individual PRs, each tagged with the right `update-type` so the auto-merge policy picks them up correctly. One PR per dependency per cycle is the right shape.
14. **Open dependabot PRs are re-evaluated after the auto-merge workflow changes**: after pushing changes to `.github/workflows/auto-merge.yml`, comment `@dependabot rebase` on each open dependabot PR (or use the one-liner in Pitfall 14). Within a few minutes, `gh run list --workflow="Dependabot auto-merge" --json headBranch,conclusion` should show a fresh `success` run on each rebased PR's branch — *not* zero runs (Pitfall 14 — workflow file change does not synchronize existing PRs). The per-step conclusions on that new run should be `success` (or `skipped` for steps the policy intentionally bypasses), not `failure` with a `workflows` permission error (Pitfall 5) or a `Not Found` (Pitfall 6, repo flag off).
15. **No `DIRTY` regression on workflow-touching PRs after rewriting `auto-merge.yml`**: if any open dependabot PR bumps an action used in `auto-merge.yml` (or in any other file your rewrite modified), `gh pr view <N> --json mergeStateStatus,mergeable` should not show `DIRTY`/`CONFLICTING`. If it does, the PR's rebase failed because your rewrite removed the line the PR was trying to change — comment `@dependabot recreate` on it (Pitfall 15), wait for the new commit, and re-check. After recreate, `mergeable: MERGEABLE` and the auto-merge workflow re-runs against the new diff.
16. **Every label referenced in `dependabot.yml` exists in the repo (Pitfall 17)**: run the diff from Pre-flight step 2 against the *current* repo state right before reporting done. `comm -23 /tmp/want.txt /tmp/have.txt` should be empty. If the comparison surfaces any missing label, the next dependabot cycle will drop every PR for that ecosystem silently — fail this check and create the labels before pushing the final `dependabot.yml`. Verify by also inspecting the Dependabot tab (Insights → Dependency graph → Dependabot) for the `The following labels could not be found` error.

If any check fails, do not report success. Go back to Pitfalls and diagnose.

---

## Final checklist before reporting done

Run through this list *immediately before* sending the success message. Every box must be checked. The historical failure mode is "the optimization worked, the agent reported done, the user had to prompt 'did you update the skill?'" — this checklist exists to make that lapse impossible to repeat.

**Project-side deliverables**

- [ ] The optimization changes are committed and pushed to the project's default branch.
- [ ] `gh pr list --state open` returns either `[]` or only PRs the user has explicitly chosen to leave open. A non-empty list means at least one dependabot PR did not drain — diagnose why before reporting done.
- [ ] Master history is linear (`gh api repos/<owner>/<repo>/commits/master --jq '.parents | length'` for recent commits should consistently be `1`, not `2`). `--rebase` auto-merge produces linear history; a merge commit means something merged outside auto-merge.
- [ ] Per-project notes file written at `<project>/docs/dependabot-optimization-notes.md`. Template is in the Self-improvement loop section below; required sections are Context, What was done, What worked first time, What was tricky / required iteration, Key findings, Verification results. Do **not** commit this file unless the user asks.

**Skill-side deliverables (Step 7)**

- [ ] `SKILL.md` updated with anything genuinely new from this run: new pitfalls, snag enrichments, worked-example entry, trigger phrases, count bumps if applicable. Do not rewrite unaffected sections — the goal is a surgical diff that captures the lesson.
- [ ] `README.md` synced if `SKILL.md` changed: trigger-phrase list bumped, pitfall count bumped if it changed, Worked-example project list bumped if a new project was added.
- [ ] Skill changes committed locally (`git commit` in `/home/xenoamess/workspace/dependabot-automerge-skill`). **Do not push** — the skill has no remote and is loaded from a local path.
- [ ] The commit message follows the pattern used by prior runs (e.g. `skill: <project> worked example + <one-line summary of lesson>`).

**Reporting**

- [ ] The success message lists both halves of the deliverable: "project optimized" *and* "skill updated and committed locally as `<sha>`". A success message that only mentions the project is the historical lapse pattern — make both halves visible in the same reply.

---

## Self-improvement loop (run after a successful optimization)

This skill improves with every repo you apply it to. After Verification
(all sixteen checks pass), audit what you actually learned during
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
<table of 14 checks, with actual results>
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
2. Update affected counts (e.g. "fifteen pitfalls" → "sixteen pitfalls",
   "fifteen verification checks" → "sixteen"). Grep for the old count first
   to catch every reference. The current target is sixteen pitfalls and
   sixteen verification checks.
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

After the follow-up that produced this revision (re-applying the
skill to `XenoAmess/evosuite` after a `dependabot/maven-major`
PR exposed a long chain of latent issues):

- **Pitfall 13 added**: `groups:` with `patterns: ["*"]` in
  `dependabot.yml` is an anti-pattern, not a feature. Symptom: the
  next weekly cycle opens a single PR containing every available
  major (or minor/patch) upgrade in the ecosystem, and any CI
  failure inside it is un-attributable to a specific bump. Cause:
  the `patterns: ["*"]` matches every dependency in the ecosystem,
  so the `groups:` block is equivalent to "one PR per ecosystem
  per cycle" rather than the implicit "one PR per dependency per
  cycle". The previous version of this skill's Step 6 explicitly
  recommended this pattern, on the rationale that "N updates → 1
  PR" reduces noise. In practice the noise just relocates from
  the PR list to the diff of one PR. Fix: drop the `groups:` block
  entirely; one PR per dependency per cycle is the right shape.
- **Step 6 rewritten** to drop the `groups:` recipe. The new
  recipe has the same weekly schedule, PR limit, prefix, labels,
  and per-ecosystem entries, just without the `groups:` block.
  Per-PR auto-merge policy still works because the auto-merge
  workflow's `update-type` check reads the bump's `semver-*`
  classification from dependabot's metadata directly — no group
  name required.
- **Verification check 13 added**: `cat .github/dependabot.yml`
  must not contain `patterns: ["*"]` anywhere. The grep is
  literal: that string is the unique signature of the
  anti-pattern. (Snag form for runtime: drop the `groups:`
  block on the next push; the cycle after that reopens the
  individual PRs.)
- **Snag added** restating Pitfall 13 in a runtime checklist
  form: "if you find a `groups:` block in an inherited config,
  drop it on the next push and let the next cycle reopen the
  individual PRs". Also clarifies the existing `Grouping in
  dependabot.yml closes the old individual PRs` snag — the
  closure is the *good* half (it forces the next cycle to
  re-evaluate), but any future cycle will repeat the bundling,
  so the new Snag recommends also dropping the `groups:`.
- **Description and When-to-use list extended** with three new
  trigger phrases: "dependabot grouped my major upgrades into
  one huge PR that broke three things at once",
  `patterns: ["*"]` in my dependabot.yml groups everything into
  a single PR`, and "drop the `groups:` block from
  dependabot.yml". The user-facing symptom of this pitfall
  ("one huge PR that broke three things at once") is the
  exact phrasing someone is most likely to type at you.
- **Counts bumped**: twelve → thirteen pitfalls, twelve →
  thirteen verification checks. The "twelve pitfalls" and
  "twelve-item verification checklist" references in README.md
  were also bumped to "thirteen".

One commit, ~85 insertions (one new pitfall, one verification
check, one snag, a Step-6 rewrite, one worked-example entry,
trigger-phrase bumps in two places), no rewrite of the
unaffected sections. The previous count was twelve pitfalls /
twelve verification; grepping the file for "twelve" before
editing caught the README references that needed bumping.

After optimizing `XenoAmess/to4096Words` (the run that produced
this revision):

- **Pitfall 7 enriched** with a "stale-base variant": posting
  `@dependabot rebase` on a PR stuck at `BEHIND` returns "The
  base commit for this pull request has not changed" when
  dependabot's *internal* view of the base is older than the
  live `master` HEAD (typical after a migration push). The fix
  is `@dependabot recreate`, which forces a fresh commit
  generated against the current master and lets auto-merge
  re-enable cleanly. This is a sub-case of the BEHIND state —
  not a new pitfall, but a useful escalation path when the
  primary rebase command refuses to act.
- No count change. The new content is an enrichment of the
  existing Pitfall 7's "Adjacent-line variant" section, plus a
  concrete one-liner in the worked example.

After optimizing `cyanpotion/x8l` (the run that produced
this revision):

- **Pitfall 5 enriched** with the device-flow path for adding the
  `workflow` scope to an existing `gh` OAuth token. Symptom: the
  user-OAuth-token shortcut smoke test fails with
  `without 'workflow' scope` even though the user is repo admin.
  Cause: the token's `workflow` scope is not active for this
  session. Fix: run `gh auth refresh -h github.com -s workflow`,
  which enters GitHub device flow; the user opens
  `https://github.com/login/device`, enters the one-time code, and
  authorizes. After authorization the same `gh auth token` value
  works for workflow-touching PRs. Important: `gh auth status` may
  continue to list only `admin:public_key`, `gist`, `read:org`,
  `repo` and omit `workflow` even though the token now passes the
  smoke test. Trust the smoke test result, not the scopes list.
- **Pitfall 5 user-OAuth-token shortcut verified on a repo under an
  organization**. `cyanpotion/x8l` is owned by the `cyanpotion`
  organization; the user is admin via org membership. The shortcut
  still worked after the device-flow refresh, confirming the path
  applies to org repos as well as personal repos.
- **Verification check #11 observed in the wild**: calling
  `gh pr merge --auto --rebase` on an open workflow-touching
  Dependabot PR (e.g. `actions/checkout` 6→7) set
  `autoMergeRequest.enabledBy.login: "XenoAmess"` and the PR
  auto-merged via rebase within seconds of CI passing.
- **Pitfall 14 rebase side-effect confirmed**: changing
  `commit-message.prefix` from `build(deps)` to `ci` for
  github-actions caused rebased open action-bump PRs to retitle
  from `build(deps): bump actions/cache ...` to
  `ci: bump actions/cache ...`. This is expected dependabot
  behavior; notification filters keyed to the old prefix will stop
  firing for those PRs.
- **Migration push produced `UNKNOWN` before `BEHIND`/`BLOCKED`**:
  immediately after setting branch protection and pushing the
  workflow changes, open Dependabot PRs briefly showed
  `mergeStateStatus: UNKNOWN` before settling to `BEHIND` or
  `BLOCKED`. This is normal; GitHub recomputes merge state after
  branch protection changes. A short wait plus a re-query confirms
  the real state.
- No count change. Pitfalls remain at fourteen; verification checks
  remain at fourteen.

After optimizing `XenoAmess/metasploit-java-external-module` (the run that
produced this revision):

- **Default branch gap closed.** The repo uses `develop`, not `master`. All
  `master` references in the recipe had to be adapted: branch protection,
  `dependabot.yml` `target-branch`, and `build.yml` `push: branches:`. Added
  a pre-flight note to discover `.default_branch` first and substitute it
  everywhere.
- **Maven submodule redundancy observed.** A separate `package-ecosystem:
  maven` entry for `/src/core` was added initially, then removed after
  discovering that the root `/` Maven aggregator already recurses into
  submodules and creates PRs like `dependabot/maven/src/core/...`. The
  duplicate entry produced redundant PRs and had to be cleaned up. Promoted
  to a Snag: "Maven aggregator projects: do not add separate ... entries
  for submodules."
- **Pitfall 11 confirmed in the wild:** the repo used
  `ahmadnassri/action-dependabot-auto-merge@v2` with a lowercase `mytoken`
  secret name while the actual secret was `MYTOKEN`. The wrapper reported
  green `success` while doing nothing. Replacing it with explicit `gh pr
  review --approve` + `gh pr merge --auto --rebase` steps immediately
  surfaced the real state and started merging eligible PRs.
- **Pitfall 2 subtle variant confirmed:** `build.yml` had only `on: [push]`.
  CI appeared on PRs because Dependabot pushes on rebase, but there was no
  real `pull_request` gate. Changing to `push: branches: [develop]` +
  `pull_request:` fixed the gating.
- **User OAuth token shortcut verified again.** `gh auth token` (prefix
  `gho_`) was stored as `MYTOKEN`; the first workflow-touching PR
  (`actions/checkout` 6→7) enabled auto-merge and merged via rebase once CI
  passed. No separate PAT was needed.
- Counts unchanged: still fourteen pitfalls, fourteen verification checks.
  The changes are pre-flight / snag enrichment, not new failure modes.

After optimizing `XenoAmess/XenoAmessBlogFramework` (the run that produced
this revision):

- **`dependabot.yml` `include: "scope"` collision observed on npm and
  github-actions ecosystems.** The existing config had
  `prefix: "build(deps)"` plus `include: "scope"` for both ecosystems,
  producing PR titles like
  `build(deps)(deps): bump the github-actions group with 5 updates`.
  Removing `include: "scope"` and switching github-actions to `prefix: "ci"`
  (with npm keeping `build(deps)` / `build(deps-dev)`) produced clean
  single-scope titles. This confirms the existing Snag that `prefix:` +
  `include: "scope"` collides; the concrete npm/github-actions example was
  added here for future reference.
- **`groups:` with `patterns: ["*"]` observed in an inherited npm config.**
  The repo had `npm-minor-and-patch` and `npm-major` groups as well as a
  `github-actions` group, all bundling every available upgrade into one PR
  per group. After dropping the `groups:` blocks, the next cycle will reopen
  individual PRs, each with the correct `update-type` for the auto-merge
  policy. This reinforces Pitfall 13 / the related Snag.
- **No workflow or branch-protection changes required.** The repo already
  had a working `auto-merge.yml` (using `MYTOKEN` for approve/merge),
  `build.yml` scoped to `push: branches: [master]` + `pull_request`,
  `allow_auto_merge=true`, and branch protection requiring
  `tester (ubuntu-latest, 26.x)`. Recent auto-merge workflow runs had
  already merged eligible PRs, including a workflow-touching grouped
  github-actions PR, confirming the OAuth-token-as-`MYTOKEN` path works.
- Counts unchanged: still fourteen pitfalls, fourteen verification checks.

After optimizing `XenoAmess/XenoAmessBlog` (the run that produced this
revision): a single repo simultaneously exhibited Pitfalls 1, 5, 6, and
11. The fix was straightforward (rewrite the wrapper, fix the secret
name, enable `allow_auto_merge`, set branch protection), but the
*diagnostic* is worth recording because the failure mode was the
hardest one to detect — every workflow run was green.

- **All-four-pitfalls-at-once observed in the wild.** The repo's
  `auto-merge.yml` was using `ahmadnassri/action-dependabot-auto-merge@v2`
  with `target: minor` and `github-token: ${{ secrets.mytoken }}` (lower-
  case). Combined with `allow_auto_merge: false` at the repo level, every
  auto-merge workflow run reported `success` (because the wrapper swallows
  errors) while doing nothing useful. PR #23 (`actions/checkout` 6→7) was
  the canary: a *major* bump, which `target: minor` would have skipped
  even if the wrapper worked. The empty-`mytoken` (case mismatch with
  `MYTOKEN`) made `gh pr merge --auto` 422 with `GH_TOKEN:` (empty) in
  the log. The `allow_auto_merge: false` made it 422 even if the token
  had been correct. None of these errors were visible in the Actions tab
  because the wrapper reports green exit code on swallowed failures.

- **Diagnostic sequence that finally identified all four layers**:
  1. `gh pr view <N> --json autoMergeRequest` → `null` after multiple
     workflow runs → something is failing silently.
  2. `gh run view <id> --json jobs --jq '.jobs[].steps[].conclusion'` →
     every step `success`, including the wrapper step → Pitfall 11.
  3. `gh api repos/<owner>/<repo> --jq '.allow_auto_merge'` → `false` →
     Pitfall 6, confirmed via the wrapper's swallowed 422.
  4. `gh secret list` → only `MYTOKEN` (uppercase) exists; workflow uses
     `secrets.mytoken` (lowercase) → Pitfall 5, the case mismatch makes
     `${{ secrets.mytoken }}` expand to empty.
  5. PR title is `Bump actions/checkout from 6 to 7` → semver-major →
     `target: minor` in the wrapper would have skipped it even with a
     working token → Pitfall 1, the policy gap.

- **`@dependabot rebase` vs the migration push when workflows changed in
  the same PR.** Rewriting `auto-merge.yml` did not re-evaluate the
  existing open Dependabot PR; it stayed at `autoMergeRequest: null`
  until something pushed to its branch. Commenting `@dependabot rebase`
  forced dependabot to push a new commit, which re-fired the
  `pull_request` event under the new workflow definition. Within ~90
  seconds, the new workflow approved the PR, set `autoMergeRequest` (via
  `gh pr merge --auto --rebase`), and the PR auto-merged once the
  required check passed. This is the same Pitfall 14 path used in
  earlier revisions, but here it was applied to a wrapper-action repo
  rather than a `gh pr merge` repo — the mechanics are identical.

- **Empty-secret diagnostic confirmed on `mytoken`**. The skill's
  Pitfall 5 already describes the empty-secret tell (workflow log shows
  `GH_TOKEN:` with a trailing colon and no masked asterisks). This run
  confirmed it: the workflow used `secrets.mytoken` (lowercase, undefined)
  while the repo had `MYTOKEN` (uppercase, set). The wrapper did not
  surface the error visibly because its exit code swallowed the 422.

- **Wrapper-action repos may need more than the "replace with explicit
  gh pr merge" recipe.** In this repo the wrapper was also configured
  with `target: minor`, which would have skipped the major-version
  `actions/checkout` PR even after the wrapper was replaced with the
  standard explicit-step workflow. The replacement recipe must also
  carry the same `semver-major` policy table (or the user's preferred
  policy) from Step 1 — don't assume the existing wrapper's policy
  matches what the new workflow should do.

- **Counts unchanged**: still fourteen pitfalls, fourteen verification
  checks. The new content is a worked example of multiple pitfalls
  compounding into a single hard-to-diagnose failure, plus a reminder
  that the wrapper-replacement recipe must also carry over the policy
  intent (`target:` → `update-type` checks).

After optimizing `XenoAmess-Auto/qr_code_simple` (the run that produced
this revision): an Android/Kotlin single-module project under the
`XenoAmess-Auto` org (not the personal `XenoAmess` namespace), with an
existing CI workflow (so the AGENTS.md "no CI" note was stale), existing
auto-merge workflow (with several latent bugs), and four open dependabot
PRs across two ecosystems.

- **Pitfall 15 added**: rewriting `auto-merge.yml` to drop a step that an
  open dependabot PR is trying to bump creates `DIRTY` (not just
  `BEHIND`) on the next rebase. Symptom: a `actions/checkout 6→7` PR
  goes `BEHIND → DIRTY` immediately after the rewrite pushes, because
  the `Checkout code` step that the PR was going to bump `@v6`→`@v7`
  on is no longer in the post-rewrite file. Cause: structurally
  similar to Pitfall 7's adjacent-line variant, but the trigger is
  master-side rewrites of files the PR targets, not parallel PRs.
  Fix: `@dependabot recreate` (not `rebase`), which regenerates the
  diff against current master instead of replaying the old patch.
  Avoidance: when rewriting a workflow that an open dependabot PR
  touches, push the rewrite first *only* if you plan to follow up
  with `@dependabot recreate`; otherwise let the bump merge first.
  See Pitfall 15 for the diagnostic and full avoidance recipe.

- **Pitfall 10 enriched with the signature-verification detail.** The
  existing Pitfall 10 said "the push is not signed by dependabot."
  In the session I tested an exception: `git commit --amend --no-edit`
  on a dependabot branch keeps the original `Author:` and
  `Signed-off-by:` trailers, but the amended commit has a new SHA and
  therefore a new (or absent) git signature. `dependabot/fetch-metadata@v3`
  calls GitHub's signature-verification API, sees the signature is
  invalid, and exits with `Dependabot's commit signature is not
  verified, refusing to proceed.` / `PR is not from Dependabot, nothing to do.`
  The workflow run reports `success` (the step declined but didn't
  fail), and the PR is silently ignored. The existing Pitfall 10
  recommendation (do not amend) is correct; the added detail is that
  the failure is at the *signature protocol level*, not just the
  author level — meaning any push that re-objects the commit (amend,
  rebase, filter-branch) will produce the same silent no-op, even if
  the diff is identical.

- **Snag added: pre-existing `autoMergeRequest` from a previous
  workflow version persists.** The repo had three open gradle-major
  PRs (`kotlin_version 2.4.0`, `androidx.core-core-ktx 1.19.0`,
  `androidx.activity-activity-ktx 1.13.0`) whose `autoMergeRequest`
  had been set by the *old* workflow (which approved all majors).
  The new policy correctly classifies them as `should_merge=false`
  (gradle + major = leave for human review), but the workflow does
  not unset a pre-existing `autoMergeRequest`. The PRs stay
  `BLOCKED` only because their CI is genuinely failing for unrelated
  reasons (the new kotlin/androidx versions break compilation). If
  those CI failures were ever fixed, the PRs would auto-merge under
  the *old* policy even though the new policy excludes them. The
  only ways to clear it are: push a new commit (which fires the new
  workflow, which can re-evaluate and re-set, but not clear, the
  flag) or `close + reopen` the PR (which resets
  `autoMergeRequest` to null per the Snag about Pitfall 14). There
  is no `gh pr merge --disable-auto` flag.

- **Snag added: reliable `gh secret set` pattern.** Several attempts
  at `gh secret set NAME --body "$VAR"` / `gh secret set NAME -b
  "$VAR"` / `echo "$VAR" | gh secret set NAME` from a single bash
  session, with intermediate `gh secret delete` and re-set attempts,
  produced one empty secret during testing. The workflow log
  surfaced it as `GH_TOKEN: ` (colon, no asterisks) per the existing
  Pitfall 5 empty-secret diagnostic. Fix: write to a temp file with
  `mktemp + chmod 600`, then `gh secret set NAME < "$TMPF"`. Verify
  after setting by dispatching a `workflow_dispatch` job that prints
  the first few chars of the secret.

- **Counts bumped**: fourteen → fifteen pitfalls, fourteen → fifteen
  verification checks. The new Verification check #15 inspects
  `mergeStateStatus` of any open dependabot PR that touches a
  workflow file after a rewrite push; if `DIRTY`, the failure mode
  is Pitfall 15 and the fix is `@dependabot recreate`. New trigger
  phrases ("PR stuck DIRTY after I rewrote auto-merge.yml",
  "fetch-metadata refused the commit signature") added to the
  description and When-to-use list. The README's "fourteen pitfalls"
  / "fourteen concrete checks" references bumped to "fifteen" in
  three places.

- **Self-improvement loop honored end-to-end.** The previous run
  (XenoAmess/XenoAmessBlog) elevated the loop from "section to
  read" to "Step 7 of Implementation". This run was the first to
  apply that framing: the per-project notes file was written first
  (`docs/dependabot-optimization-notes.md` in the project) and the
  skill update followed. The agent originally forgot the skill
  update and the user had to prompt for it; documenting that lapse
  here so future runs treat both halves as a single deliverable.
  The same lapse recurred in the very next run
  (`xenaomess-shade/shade_asm`); see the **Final checklist before
  reporting done** section, which was added in response.

After optimizing `xenaomess-shade/shade_asm` (the run that produced
this revision): a small Maven shade-jar repo under the
`xenaomess-shade` organization. The repo had a working dependabot.yml
but no auto-merge workflow; PR #7 (`asm.version` 9.9.1→9.10.1) was
already open and stuck at `CLEAN` (no `autoMergeRequest`).

- **`prefix: "deps"` + `include: "scope"` (no `prefix-development`)
  observed on a live repo.** The existing config produced PR titles
  `deps(deps): bump asm.version from 9.7 to 9.9.1` for production
  deps and `deps(deps-dev): bump org.apache.maven.plugins:maven-shade-plugin`
  for build plugins. Note the `-dev` on the scope — dependabot
  automatically changes the scope to `deps-dev` for development
  dependencies when `include: "scope"` is on, even without an
  explicit `prefix-development`. This is the *opposite* of the
  existing Snag 1513 pattern (`prefix-development: "build(deps-dev)"`
  + `include: "scope"` → `build(deps-dev)(deps-dev): ...`). Both
  end up producing the same doubled scope. The clean fix is the
  Step 6 recipe: drop `include: "scope"`, use `prefix: "build(deps)"`
  + `prefix-development: "build(deps-dev)"` (no `include`). The
  dev/prod distinction survives in the title, the doubling does
  not. Verified: PR #7 retitled itself from `deps(deps): ...` to
  `build(deps): ...` on the first rebase after the config change.

- **Burst-on-ecosystem-add observed: 4 PRs at once, all racing.** The
  repo previously had only `package-ecosystem: "maven"` configured.
  Adding `github-actions` in the same change caused dependabot to
  fire 4 PRs simultaneously on the next cycle — `actions/checkout
  4→7`, `actions/setup-java 4→5`, `actions/upload-artifact 4→7`, and
  `dependabot/fetch-metadata 2→3`. All 4 are `semver-major` for
  github-actions and all 4 touched `.github/workflows/*.yml`. With
  the Step 1 policy table this branch auto-merges, so they all
  cleared in ~2 minutes (CI re-ran, auto-merge fired). The pre-
  existing PR #7 (the maven minor) repeatedly went `BEHIND` as each
  github-actions PR landed in front of it; took 2 `@dependabot
  rebase` cycles to drain. Lesson: when adding a new ecosystem to
  `dependabot.yml`, expect a burst of N PRs, and use the
  `update-branch` API (Quick Reference) instead of `@dependabot
  rebase` for the pre-existing open PRs — it preserves
  `autoMergeRequest` and avoids the rebase → re-enable → re-CI
  → re-merge cycle on every iteration.

- **Migrating `actions/setup-java` from v4 to v5 was the workflow
  change that flipped branch protection's required check.** When
  dependabot opened the `setup-java 4→5` PR, the workflow file
  temporarily referenced a different action version. The required
  check name stayed `build` (no matrix dimension changes here, so
  Pitfall 9 didn't apply), but the run history visibly mixed v4 and
  v5 setups for ~2 minutes until the PR merged. No action needed,
  just noting that the "burst" includes the CI workflow itself
  upgrading, which can briefly confuse diagnostic output if you're
  watching closely.

- **The `Bypassed rule violations` push message is a success
  signal, not an error.** When pushing via SSH with
  `enforce_admins: false`, GitHub prints
  `remote: Bypassed rule violations for refs/heads/master: ... -
  Required status check "build" is expected.` and accepts the push.
  First-time users may panic at this; it is the expected success
  path when admins push workflow changes that have not yet been
  validated by CI.

- **Counts unchanged**: still sixteen pitfalls, sixteen verification
  checks. The new content is one Worked-Example entry (this one)
  and a Snag enrichment for the burst-on-ecosystem-add scenario.
  No new failure mode, just a more focused protocol for one already-
  documented edge case (Pitfall 7 race during a multi-PR drain).

After a follow-up on `XenoAmess/x8l_idea_plugin` (label-creation
trigger): the user hit the Dependabot error `The following labels
could not be found: gradle. Please create it before Dependabot can
add it to a pull request.` while reviewing the existing config that
this skill's Step 6 recipe had produced.

- **Pitfall 17 added**: a `dependabot.yml` `labels:` block referencing
  a label that doesn't exist in the repo causes Dependabot to drop
  every PR for that ecosystem silently — no workflow run, no CI check,
  no notification, just an error string in the Dependabot tab that no
  one reads. Symptom is "the config looks correct but no PRs have
  appeared in days". Cause: dependabot tries to apply the labels as
  part of opening the PR, and if any of them are missing the open
  is rejected. Fix: diff `gh label list` against the union of every
  `updates[].labels[]` in `dependabot.yml`, create the missing ones,
  push the config change only after the labels exist. The Pre-flight
  step 2 added in this revision enforces that diff with an `awk`
  fallback (no `yq` dependency). Verification check #16 confirms the
  diff is empty at report time. See Pitfall 17 for the full
  symptom/cause/fix/diagnostic.
- **The label-not-found error was the first visible signal that the
  optimization has actually completed.** Earlier runs of this skill
  never checked whether the recipe's labels existed in the receiving
  repo. For repos that already had `dependencies`, `gradle`,
  `maven`, etc. defined, the config worked silently on day one. For
  a fresh repo (or a repo where the maintainer prefers different
  label names) the first cycle dropped every PR for the affected
  ecosystem. Pre-flight step 2 closes that hole: every label the
  config needs is created in the same batch as the config is pushed.
- **Count bumps**: sixteen → seventeen pitfalls, sixteen → seventeen
  verification checks. "fourteen pitfalls" / "fourteen concrete
  checks" references in README.md bumped to "seventeen" in three
  places. Trigger phrases ("dependabot labels could not be found",
  "create missing labels for dependabot", "dependabot is configured
  but no PRs appear", "dependabot is silent / no PRs are opening")
  added to description and When-to-use list, plus the Dependabot
  quirks row in README.md.
- **Counts bumped total**: one commit, surgical diff. No Pitfalls
  1–16 / Snags 1–N were touched; the new content is the Pre-flight
  step 2 (label diff + create), the new Pitfall 17, the new
  Verification check #16, a new Snag, three new trigger phrases,
  and a Worked-Example entry (this one).

After optimizing `XenoAmess/http-service-spliter` (the run that produced
this revision): a Quarkus/Maven proxy service with an existing
`dependabot.yml` (targeted `groups:` blocks for Quarkus, test, Maven
plugins, GitHub Actions, and Docker) and an existing
`dependabot-auto-merge.yml` that used a check-waiting wrapper plus
title-grep for semver detection.

- **Another wrapper variant observed: `lewagon/wait-on-check-action` +
  title-grep policy detection.** The existing workflow hardcoded check
  names (`build-native-qemu, build-jvm`) and parsed the PR title with
  `grep -qiE` to skip "major" bumps. This is fragile in two ways: check
  names drift when jobs are renamed (Pitfall 9), and title regexes miss
  `semver-major` classifications that `dependabot/fetch-metadata`
  reports reliably. The wrapper is also unnecessary — branch protection
  + `gh pr merge --auto` already waits for required checks. Replaced
  with `dependabot/fetch-metadata` + explicit `gh pr review --approve`
  + `gh pr merge --auto --rebase` steps.
- **`include: "scope"` collision confirmed across three ecosystems in
  the same repo.** Maven, GitHub Actions, and Docker all used
  `prefix: "build(deps)"` with `include: "scope"`, which would have
  produced `build(deps)(deps): ...` titles. Dropped `include: "scope"`
  everywhere and switched GitHub Actions to `prefix: "ci"`.
- **Targeted `groups:` blocks are still `groups:`.** The config did not
  use `patterns: ["*"]`, but every ecosystem still had a `groups:`
  block bundling related updates (e.g. all Quarkus patch/minor bumps
  into one PR). Dropped all groups per Pitfall 13 so the next cycle
  opens one PR per dependency; the per-PR `update-type` metadata then
  drives the auto-merge policy correctly.
- **No open or closed Dependabot PRs available for end-to-end smoke
  test.** Verification relied on static checks: `allow_auto_merge=true`,
  labels created, `MYTOKEN` set in the Dependabot namespace, branch
  protection contexts match job names, YAML parses. The first real
  Dependabot PR will be the live smoke test for the OAuth-token-as-
  `MYTOKEN` scope.
- Counts unchanged: still seventeen pitfalls, seventeen verification
  checks.

---

## Snags to watch for

- **Maven aggregator projects: do not add separate `package-ecosystem: maven` entries for submodules.** If the root `pom.xml` declares the submodules in `<modules>`, a single Dependabot Maven entry at `/` already recurses into them and opens PRs like `dependabot/maven/src/core/...` for dependencies defined in submodule `pom.xml` files. Adding extra entries (e.g. `directory: "/src/core"`) produces duplicate PRs for the same bumps and makes the backlog harder to drain. Keep one root Maven entry and let Dependabot discover the submodule updates.

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
  | `UNKNOWN` | GitHub is still computing the state (common right after branch-protection changes or a migration push; wait a few seconds and re-query) | refresh in a few seconds |

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

- **Adding a new ecosystem to `dependabot.yml` triggers a burst of N PRs that race against each other.** If you add a `package-ecosystem` that was previously absent (e.g. introducing `github-actions` to a Maven-only repo, or `npm` to a Python-only repo), the very next cycle opens N PRs — one per outdated package in the new ecosystem. All N PRs hit Pitfall 7 simultaneously, and any pre-existing open PRs in the *original* ecosystem keep getting reblocked as the new ones merge in front of them. For the new burst, auto-merge (Step 1 policy) handles them cleanly; the bottleneck is the *pre-existing* PRs from the original ecosystem. For those, prefer the `update-branch` API (`gh api -X PUT repos/<owner>/<repo>/pulls/<N>/update-branch`) over `@dependabot rebase` — it preserves `autoMergeRequest` and avoids the rebase → re-enable → re-CI → re-merge cycle on every iteration. Observed: a Maven minor PR took 2 `@dependabot rebase` cycles to drain while 4 github-actions major PRs merged in front of it on each cycle; switching to `update-branch` would have cut this to one cycle per PR.

- **Adding `groups:` to `dependabot.yml` immediately closes the original per-dependency PRs as superseded.** This is a free way to drain a backlog of N individual PRs into 1 (or a few) grouped PRs. The new grouped PR(s) then proceed through the normal auto-merge path; the original PRs close themselves. Caveat: the close happens the moment the new grouped PR is opened, which is *before* the grouped PR is merged. If for any reason the grouped PR is then closed without merging (e.g. a CI failure or a maintainer close), the individual PRs do not reopen — you have to wait for the next weekly cycle to regenerate them. For low-risk projects this is a clean win; for repos where individual PRs are deliberately being reviewed out-of-band, do not add `groups:` without first merging or closing the existing ones.

- **Grouping in dependabot.yml closes the old individual PRs.** When you add a `groups:` block to an ecosystem that already has individual PRs open, Dependabot on the next cycle closes those individual PRs and opens a single grouped PR with a title like `chore(deps): bump the major group with 2 updates`. This is expected — the old PRs show up as `CLOSED` (not merged), the new grouped PR inherits their changes, and `update-type` for the grouped PR reports the highest-severity bump in the group. Don't panic if your "verified working" PRs vanish from the open list; the replacement is the grouped PR. **But see Pitfall 13: do not keep grouping in the config long-term.** The closure is the "good" half — it forces the next cycle to re-evaluate. The bad half is that any future cycle will repeat the bundling.

- **`patterns: ["*"]` in `dependabot.yml` `groups:` is anti-pattern, not a feature.** The `groups:` block is meant to *narrow* a PR set ("only these artifacts go in this group"). Using `patterns: ["*"]` as the pattern means "every artifact in this ecosystem", which is the same as having no `groups:` at all except for the side effect of merging N PRs into 1. The previous version of this skill (Step 6) recommended exactly this pattern, on the rationale that "N updates → 1 PR reduces noise". In practice the noise just relocates from the PR list to the diff of one PR, and CI failures become un-attributable. A 3-bump grouped PR where one bump is the culprit and the other two are innocent: you cannot tell which is which from the CI log. A 3-PR stream with one failing: the failing PR's title and CI log make the culprit obvious. Pitfall 13 is the formal version of this lesson; the Snag form is "if you find a `groups:` block in an inherited config, drop it on the next push and let the next cycle reopen the individual PRs".

- **`AdoptOpenJDK` distribution is being phased out.** `actions/setup-java` with `distribution: adopt` now emits an annotation warning that AdoptOpenJDK has moved to Eclipse Temurin. Recommended: change to `distribution: temurin` in the CI workflow. Non-blocking but visible in the Actions tab.

- **The first dependabot batch tests the hardest path.** If the repo has not been kept up to date, the very first dependabot cycle after enabling this skill will open N major version bumps, most of which touch `.github/workflows/*.yml`. That means the first PRs to be tested are exactly the ones that need the `workflow` scope on `MYTOKEN` and the `gh pr merge --auto` API call. If anything is misconfigured with the token, it shows up immediately. Plan for this: do the Pre-flight 5 scope check *before* the first batch lands, not after.

- **`gh auth refresh -s workflow` hangs in non-interactive CLI sessions.** If you try to add the `workflow` scope to an existing OAuth token from a non-interactive session, the command blocks waiting for a browser confirmation. It will not time out on its own; you have to Ctrl-C it. Either take the user-OAuth-token shortcut (Pitfall 5) and use the token as-is, or ask the user to add the scope via the web UI at `Settings → Developer settings → Personal access tokens → [token] → Edit scopes → Workflow ✓`. Do not retry the refresh in a loop.

- **User OAuth token login normalization is undocumented and may change.** Even when `gh pr list --json author` shows `app/dependabot` for a PR opened by the new Dependabot GitHub App, the value of `github.event.pull_request.user.login` inside the workflow run is normalized to `dependabot[bot]`. This means a workflow gated on the legacy form will silently work on a fully migrated repo. The match-both `||` form (Pitfall 8) is still the right defense — the normalization is undocumented behavior and could be removed in a future GitHub update.

- **`update-branch` API returns 422 when master hasn't moved.** `gh api -X PUT repos/<owner>/<repo>/pulls/<N>/update-branch` only rebases if the base branch has a new commit. If the PR is in the "old auto-merge workflow" state and you want to force a re-evaluation without a rebase, use `gh pr close <N> --delete-branch=false && gh pr reopen <N>` instead — that re-fires the `pull_request: reopened` event with no base-branch motion required. Useful right after you set `MYTOKEN` to a valid value and need the workflow to retry the approve+merge steps that previously failed for the empty-secret reason.

- **`include: "scope"` + `prefix:` collides into `build(deps)(deps): ...`.** When the conventional-commits-style prefix already contains a scope (`build(deps)`, `chore(deps)`, `feat(api)`) and `dependabot.yml` also has `include: "scope"`, the package-manager scope (`deps` for maven, `github-actions` for the github-actions ecosystem) is appended to the prefix, producing a doubled scope. Drop `include: "scope"` from the dependabot config; the package name in the commit body is enough disambiguation.

- **`prefix-development` + `include: "scope"` produces `build(deps-dev)(deps-dev): ...` for Maven.** Maven dependencies are reported as `dependency-type: direct:development` by dependabot, so `prefix-development` and `include: "scope"` both apply to the same package. Setting `prefix-development: "build(deps-dev)"` plus `include: "scope"` yields `build(deps-dev)(deps-dev):` — the development prefix and the development scope collide. For Maven projects, just use a single `prefix:` (e.g. `build(deps)`) and skip `prefix-development`; the per-PR title will be `build(deps)(deps): ...` at worst, which is the same pattern the existing snag above describes. Alternatively keep `prefix-development` but set it to the same value as `prefix` so the collision is invisible.

- **Check name format is `<job-name> (<matrix-axes>)`, not `<workflow> / <job>`.** A common source of duplicate check names when two workflows (`build.yml` and `build_extra.yml`, say) both define a job called `build` with the same matrix. Renaming `name:` on the workflow does not help; the job ID has to differ. See Pitfall 12 for the full pattern and fix.

- **Pre-existing `autoMergeRequest` from a previous workflow version persists.** When you tighten the auto-merge policy (e.g. add a `semver-major` exclusion for github-actions, switch from `target: minor` to a stricter rule, or replace a permissive wrapper with explicit steps), existing open dependabot PRs that the old workflow already armed with `autoMergeRequest` keep that flag. The new workflow runs on the next `synchronize` event but only adds or re-sets `autoMergeRequest`; it does not clear one set by the old version. If those PRs happen to pass CI (or already pass it), they will auto-merge under the *old* policy even after the new policy is in place. To unstick them: `close + reopen` the PR (Pitfall 14 pattern), or push any new commit to its branch so the new workflow re-evaluates and re-sets `autoMergeRequest` to whatever the new policy decides. There is no `gh pr merge --disable-auto` flag — clearing requires a PR-level event.

- **Setting `MYTOKEN` reliably: prefer the temp-file pattern over shell pipelines.** When scripting `gh secret set NAME`, multiple invocations from the same shell session are easy to mess up: `--body "$VAR"` with a shell variable, `echo "$VAR" | gh secret set NAME` via stdin, and `gh secret set NAME -b "$VAR"` all *can* work, but a typo or a partially-overwritten variable can silently produce an empty secret. The workflow log will show `GH_TOKEN: ` (colon, no asterisks) per the Pitfall 5 empty-secret diagnostic, but you may not catch it until the first workflow run fails. The most reliable pattern is to write the token to a temp file with `mktemp` + `chmod 600`, then pipe it in explicitly:

  ```bash
  TMP=$(mktemp) && chmod 600 "$TMP" && gh auth token > "$TMP" \
    && gh secret set MYTOKEN < "$TMP" && rm -f "$TMP"
  ```

  Verify after setting with a one-shot diagnostic workflow (Pitfall 5 enrichment, also see Verification #12): create a temporary `verify-mytoken.yml` with `on: workflow_dispatch`, print `${MYTOKEN:0:6}` (and `wc -c`), `workflow_dispatch` it, read the log, then delete the workflow file and commit the deletion.

- **`gh pr merge --auto --rebase` is idempotent for `autoMergeRequest` but not for the workflow-run history.** Calling it directly from the shell (or from a verification smoke test) sets `autoMergeRequest` once; subsequent calls do not produce a new entry. But each call to the auto-merge workflow's `gh pr merge --auto --rebase` step does produce a workflow run. If the user-OAuth-token smoke test in Pitfall 5 has already set auto-merge on the test PR, the workflow run that fires after the rebase will call `gh pr merge --auto --rebase` again — and GitHub will treat it as a no-op for the merge itself but as a fresh run for the audit log. This is harmless; just don't be surprised by the run count.

- **Missing labels break the very first dependabot cycle, not the second.** When you push `dependabot.yml` with labels that do not yet exist in the repo, the first cycle after the push drops *every* PR for the affected ecosystem. Not just some, not partial — zero. The Dependabot tab surfaces the error but no email / no CI failure / no GitHub notification reaches the maintainer. The symptom (no PRs opened) is easy to misread as a schedule/branch problem. The fix is to run Pre-flight step 2's label-diff before pushing the config (see Pitfall 17). If you inherit a config that references labels the repo doesn't have, create them in the same change before pushing the `dependabot.yml` edit — otherwise the config looks like it does nothing for a full cycle before the labels land and the next cycle recovers.

---

## Quick reference — all `gh` commands used in this skill

> Replace `master` below with the repo's actual default branch (`gh api repos/<owner>/<repo> --jq '.default_branch'`).

```bash
# Auth check
gh auth status

# Default branch discovery
gh api repos/<owner>/<repo> --jq '.default_branch'

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
- "A dependabot PR is stuck at `DIRTY` after I rewrote `auto-merge.yml` (not after a sibling PR merged)" → Pitfall 15. Your rewrite deleted or moved a line the PR was trying to change. Comment `@dependabot recreate` on the PR; the regenerated diff will skip the deleted step.
- "I amended a dependabot commit and the auto-merge workflow now silently no-ops" → Pitfall 10 protocol-level enforcement. `dependabot/fetch-metadata` verifies the commit's git signature, which amend always breaks. Use `@dependabot recreate` instead; do not amend.
- "MYTOKEN is set but the workflow sees an empty value (env shows `GH_TOKEN: `, not `***`)" → Pitfall 16. `gh secret set` without `--app dependabot` writes to the wrong namespace. Re-set with `--app dependabot` and verify with `gh secret list --app dependabot`.
- "I set MYTOKEN via `gh secret set` but `gh secret list` shows it yet the workflow sees empty" → Pitfall 16. Default `gh secret list` only shows the actions namespace; the secret landed there, where dependabot PR workflows cannot read it. Re-set with `--app dependabot`.
