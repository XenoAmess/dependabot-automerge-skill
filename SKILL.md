---
name: dependabot-automerge-skill
description: Set up or repair GitHub Dependabot auto-merge for a repository. Use when the user mentions dependabot, auto-merge, dependabot PR stuck, semver-major merging, GitHub Actions PRs not auto-merging, branch protection required checks, or wants to reduce manual PR churn. Triggers on phrases like "set up dependabot auto-merge", "PR stuck waiting for checks", "auto-merge not waiting for CI", "major version dependabot", "branch protection required status check". Does NOT use when the user only wants to configure dependabot.yml update schedule, or only wants to enable dependabot security updates without auto-merge.
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

Do **not** use this skill for:

- Just editing `.github/dependabot.yml` schedule/ecosystem config (no auto-merge involved).
- Dependabot security updates without auto-merge.
- Non-Dependabot PR auto-merge (e.g. `bors`, merge queues, custom bots). Different problem.

---

## Pre-flight checks

Before changing anything, gather context. Do not skip these.

1. **Read the current state**:
   - `.github/dependabot.yml` — which ecosystems are configured?
   - `.github/workflows/` — is there already an `auto-merge.yml`?
   - If `build.yml` exists, what is the `on:` block? Does it include `pull_request`?
2. **Check the GitHub repo** (use `gh` CLI; the user must be authenticated):
   ```bash
   gh auth status
   gh api repos/<owner>/<repo>/branches/master/protection
   ```
3. **Check if there are stuck PRs** that motivated the request:
   ```bash
   gh pr list --state open --json number,title,headRefName,mergeStateStatus
   ```
4. **Read the actual check names** that GitHub is generating, not what you assume. See §4.2.

If the repo does not have `gh` auth, stop and ask the user to log in. Do not guess at branch protection.

---

## Strategy

There are four moving parts. All four are required.

| # | File / setting | Purpose |
| --- | --- | --- |
| 1 | `.github/workflows/auto-merge.yml` | Approve and enable auto-merge on qualifying Dependabot PRs |
| 2 | `.github/workflows/build.yml` | Run CI on `pull_request` so required checks appear on the PR |
| 3 | Branch protection on `master` | Mark the CI check as required, so `gh pr merge --auto` actually waits |
| 4 | `dependabot.yml` (optional) | Only touch if the user wants grouping/ignores/schedule changes |

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

Create `.github/workflows/auto-merge.yml`:

```yaml
name: Dependabot auto-merge

on:
  pull_request:

permissions:
  pull-requests: write
  contents: write

jobs:
  dependabot:
    if: github.event.pull_request.user.login == 'dependabot[bot]'
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
            echo "should_merge=true" >> $GITHUB_OUTPUT
          else
            echo "should_merge=false" >> $GITHUB_OUTPUT
          fi

      - name: Approve dependabot PR
        if: steps.check.outputs.should_merge == 'true'
        run: gh pr review --approve "$PR_URL"
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Enable auto-merge
        if: steps.check.outputs.should_merge == 'true'
        run: gh pr merge --auto --rebase "$PR_URL"
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Adapt the third `||` branch to match the policy table above. For a Maven-only repo, drop the major-merge clause entirely.

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

If the build uses a `matrix`, the actual check name will include matrix dimensions (e.g. `build (ubuntu-latest, 11, false)`). You will need this exact string in step 3.

### Step 3 — Branch protection

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
      { "context": "build (ubuntu-latest, 11, false)" },
      { "context": "build (windows-latest, 11, false)" }
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

### Step 4 — Commit and push

```bash
git add .github/workflows/auto-merge.yml .github/workflows/build.yml
git commit -m "ci: dependabot auto-merge with required CI gate"
```

`git push` may be rejected because the remote master has moved while you were editing. Rebase, then push. The `Bypassed rule violations` message during push is expected when `enforce_admins: false`.

---

## Pitfalls (read these before declaring success)

These are the four things that have actually broken in production. Re-check each one before finishing.

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

**Cause**: Branch protection has `Java CI / build` (workflow name + job name), but the actual GitHub Actions check name is the job name with matrix dimensions, like `build (ubuntu-latest, 11, false)`. The two do not match.

**Diagnostic**: Run the GraphQL query in Step 3 and look for `isRequired: false` on a check you expected to be required. That is the smoking gun.

**Fix**: Use the actual name(s) shown by GraphQL. For matrix jobs you need one entry per matrix axis value — there is no wildcard.

### Pitfall 4 — CI runs twice per PR

**Symptom**: A single dependabot PR shows four CI runs (push + pull_request × matrix axes). Runner minutes wasted.

**Cause**: `on: { push:, pull_request: }` triggers both events when dependabot pushes to its PR branch.

**Fix**: Scope `push` to the base branch: `push: { branches: [ master ] }`. PR commits only fire `pull_request`; direct pushes to master only fire `push`. No overlap.

---

## Verification (mandatory before reporting done)

Do not tell the user "done" until all six checks pass.

1. **CI runs on the PR**: open any dependabot PR, confirm `build (..., ..., ...)` checks appear under the PR's Checks tab.
2. **Required check is actually required**: GraphQL query in Step 3 returns `isRequired: true` for the CI check(s).
3. **Patch auto-merges**: trigger a patch bump (e.g. dependabot creates one overnight), confirm it gets approved and merged within a few minutes.
4. **GitHub Actions major auto-merges**: trigger a github-actions major bump. Confirm same as above.
5. **Maven major does NOT auto-merge**: trigger a maven major bump. Confirm the PR is approved by the bot but stays open with `auto-merge: false`.
6. **No double-runs**: in the Actions tab, the latest dependabot PR should have exactly `N` CI runs (where `N` is the size of the matrix), not `2N`.

If any check fails, do not report success. Go back to Pitfalls and diagnose.

---

## Snags to watch for

- **`enforce_admins: false` does not apply to Dependabot**: Dependabot opens PRs as itself, so the admin bypass does not affect it. Good — the protection will still gate Dependabot.
- **`@dependabot recreate` resets the PR**: if you find a PR that has been force-pushed mid-flight, give it a minute. The auto-merge workflow re-runs on `pull_request` events.
- **First run after enabling**: the first time branch protection is set up, GitHub can take 30–60 seconds to register the required checks. Re-querying GraphQL immediately may still show `isRequired: false`. Wait, then re-check.
- **Admins can push even with branch protection** when `enforce_admins: false`. So you can hot-fix without disabling protection.

---

## Quick reference — all `gh` commands used in this skill

```bash
# Auth check
gh auth status

# Current branch protection
gh api repos/<owner>/<repo>/branches/master/protection

# PR list
gh pr list --state open --json number,title,headRefName,mergeStateStatus

# PR status
gh pr view <N> --json mergeStateStatus,statusCheckRollup

# Real check names + isRequired
gh api graphql -F query='...'   # see Step 3

# Set branch protection
gh api -X PUT repos/<owner>/<repo>/branches/master/protection \
  -H "Accept: application/vnd.github+json" --input - <<'EOF'
{ ... }
EOF
```

---

## When the user pushes back

- "I don't want CI to run twice" → Step 2 + Pitfall 4.
- "Major versions should not auto-merge at all" → drop the third `||` branch in Step 1.
- "Maven major should also auto-merge" → remove the `dependabot/github_actions/*` prefix gate in Step 1.
- "I want strict review" → set `required_pull_request_reviews` in Step 3. Note: this will likely conflict with `enforce_admins: false`; pick one.
- "I want a different merge method" → change `--rebase` to `--merge` or `--squash` in `auto-merge.yml`.
