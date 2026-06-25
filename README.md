# dependabot-automerge-skill

An [opencode](https://opencode.ai) skill that configures GitHub Dependabot auto-merge for any repository. It is built from real incident post-mortems — the twelve "Pitfalls" sections are the things that have actually broken in production. The skill is self-improving: after every successful run, it writes a per-project notes file and updates itself with what it learned.

## What it does

When triggered, the skill:

1. Inspects the current repo state (dependabot.yml, workflows, branch protection, open Dependabot PRs **and their author login**, available tokens, `allow_auto_merge` flag).
2. Creates / updates `.github/workflows/auto-merge.yml` with the correct decision policy. Uses a user OAuth token (admin shortcut) or a separate PAT for the merge step (required for PRs that touch `.github/workflows/`). Matches both `app/dependabot` and `dependabot[bot]` logins.
3. Adjusts the CI workflow (`build.yml`) so it runs on `pull_request` and does not double-trigger on `push`.
4. Enables `allow_auto_merge` at the repo level.
5. Sets branch protection with the **actual** required check name(s) — discovered from GraphQL, not guessed.
6. Hardens `dependabot.yml` (weekly schedule, grouped updates, sensible PR limit, labels) so the user doesn't drown in noise.
7. Verifies the setup against twelve concrete checks before reporting success, including a run-log diagnostic for the easy-to-miss `allow_auto_merge: false` failure mode, an empty-`MYTOKEN` detection diagnostic for the silent-failure case, a `pull_request`-vs-`push` event diagnostic for the "CI looks like it works but isn't gating" anti-pattern, and a duplicate-check-name detection diagnostic for repos with split CI workflows.
8. **Self-improvement**: writes `<project>/docs/dependabot-optimization-notes.md` and updates this skill with anything new it learned. Both are required deliverables, not optional polish.

## When the skill triggers

It activates on phrases like:

- "set up dependabot auto-merge"
- "PR stuck waiting for checks"
- "auto-merge skipped CI"
- "branch protection required status check"
- "I want major version Dependabot PRs to merge automatically"
- "PR stuck at `BEHIND`"
- "PR stuck at `DIRTY` (not BEHIND) after a sibling dependabot PR merged"
- "`gh pr merge --auto` returns 422 / auto-merge not allowed"
- "OAuth App cannot create or update workflow"
- "I changed auto-merge.yml and nothing happened"
- "auto-merge workflow never runs / always skipped"
- "app/dependabot vs dependabot[bot]"
- "I bumped the JDK matrix and now auto-merge is broken"
- "dependabot PRs stuck after I changed build.yml"
- "CI looks like it's running but isn't gating the PR"
- "all my dependabot PRs went BEHIND at once"
- "I have `gh` but I don't want to create a separate PAT"
- "rebase produced a real conflict not just BEHIND"
- "first batch of dependabot PRs are all major version bumps touching workflow files"
- "MYTOKEN is set but auto-merge still fails with empty GH_TOKEN"
- "two CI workflows produce the same `build (os, java)` check name"
- "dependabot PR titles look like `build(deps)(deps): ...` with duplicated scope"
- "dependabot PR title is `build(deps-dev)(deps-dev)` Maven double prefix"
- "no open dependabot PRs but I still want to verify MYTOKEN scope"
- "local clone is way behind origin and `git status` lies"
- "dependabot grouped PR closed my individual PRs — should I worry?"

See the `description` field in `SKILL.md` for the full trigger list.

## How to use

This is an external skill. Register it in your `opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "skills": {
    "paths": ["/home/xenoamess/workspace/dependabot-automerge-skill"]
  }
}
```

Restart opencode after adding the path. Then ask opencode to set up auto-merge for a repo.

## Self-improving

The skill maintains a `## Self-improvement loop` section at the bottom of `SKILL.md` that **Step 7 of Implementation** invokes after every successful run. The loop writes a per-project notes file at `<project>/docs/dependabot-optimization-notes.md` (template in `SKILL.md`) and updates the skill with new pitfalls, snags, trigger phrases, and verification checks. The skill's own git log is the audit trail of what was learned where — see the `### Worked example` subsections for prior runs (`XenoAmess/docker-image-rebecca`, `XenoAmess/x8l_idea_plugin`, `cyanpotion/cyan_zip`, `XenoAmess/jcpp-maven-plugin`, `XenoAmess/evosuite`).

## Layout

```
dependabot-automerge-skill/
├── SKILL.md          # the skill — loaded by opencode
└── README.md         # this file
```

The skill itself is a single `SKILL.md` file. opencode scans for `**/SKILL.md` inside any directory listed under `skills.paths`, so this whole directory can be dropped in as-is.

## Why a single file?

opencode skills are markdown. They are loaded as system context for the agent. Putting the entire procedure in one file keeps the agent's prompt focused: it sees the strategy, the twelve pitfalls, and the twelve-item verification checklist in one place, with no extra navigation.

## License

Same as the originating repo (`java_pojo_generator`, MIT).
