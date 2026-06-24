# dependabot-automerge-skill

An [opencode](https://opencode.ai) skill that configures GitHub Dependabot auto-merge for any repository. It is built from real incident post-mortems — the seven "Pitfalls" sections are the things that have actually broken in production.

## What it does

When triggered, the skill:

1. Inspects the current repo state (dependabot.yml, workflows, branch protection, open Dependabot PRs, available PAT secrets, `allow_auto_merge` flag).
2. Creates / updates `.github/workflows/auto-merge.yml` with the correct decision policy. Uses a PAT for the merge step (required for PRs that touch `.github/workflows/`).
3. Adjusts the CI workflow (`build.yml`) so it runs on `pull_request` and does not double-trigger on `push`.
4. Enables `allow_auto_merge` at the repo level.
5. Sets branch protection with the **actual** required check name(s) — discovered from GraphQL, not guessed.
6. Verifies the setup against eight concrete checks before reporting success.

## When the skill triggers

It activates on phrases like:

- "set up dependabot auto-merge"
- "PR stuck waiting for checks"
- "auto-merge skipped CI"
- "branch protection required status check"
- "I want major version Dependabot PRs to merge automatically"
- "PR stuck at `BEHIND`"
- "`gh pr merge --auto` returns 422 / auto-merge not allowed"
- "OAuth App cannot create or update workflow"
- "I changed auto-merge.yml and nothing happened"

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

## Layout

```
dependabot-automerge-skill/
├── SKILL.md          # the skill — loaded by opencode
└── README.md         # this file
```

The skill itself is a single `SKILL.md` file. opencode scans for `**/SKILL.md` inside any directory listed under `skills.paths`, so this whole directory can be dropped in as-is.

## Why a single file?

opencode skills are markdown. They are loaded as system context for the agent. Putting the entire procedure in one file keeps the agent's prompt focused: it sees the strategy, the four pitfalls, and the verification checklist in one place, with no extra navigation.

## License

Same as the originating repo (`java_pojo_generator`, MIT).
