# Git Weekly Report

A slash command that reads a repo's git history for the past week, summarizes code activity,
and — the main point — analyzes what problems came up and how they were actually fixed. Meant
to be run on a schedule so you get a standing weekly digest without asking for it each time.

- **`/weekly-report`** (command) — pulls commits/merges since a window (default: 7 days),
  classifies them (feat/fix/refactor/chore/revert/...), and builds an "Issues & Fixes" section
  that pairs each bug fix with what broke and what resolved it, flagging areas that got patched
  more than once (a sign of an unstable spot worth a closer look). If you don't name a branch,
  it auto-detects which one is the real integration point from merge activity — not from
  naming convention — so it works for teams that route everything through `UAT`, `SIT`,
  `staging`, or whatever their process actually uses instead of textbook git-flow.

## Folder structure
```
git-weekly-report/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   └── weekly-report.md
└── README.md
```

## Install — Claude Code
```bash
# one-off for a single session
claude --plugin-dir /path/to/git-weekly-report

# or, to keep it available, add its parent folder as a local marketplace
/plugin marketplace add /path/to    # the folder that CONTAINS git-weekly-report
/plugin install git-weekly-report
```

Then, inside the repo you want a report for:
```
/weekly-report
```

Arguments are optional, in any order:
```
/weekly-report 2 weeks ago     # widen the window
/weekly-report ../other-repo   # point at a different repo/subdir, still last 7 days
/weekly-report UAT             # report on the UAT branch explicitly, skip auto-detection
/weekly-report SIT 2 weeks ago
```

No branch is hardcoded to `main`/`master`/`develop`. If you don't name one, it doesn't just
default to whatever's checked out either — it looks at merge activity across every branch
(merge count into each branch over the last 90 days, how many distinct branches fed into it,
recency) and picks whichever one is actually functioning as the integration point, then states
its evidence in the report's Overview section (e.g. "picked `SIT`: 14 merges in 90 days from 6
distinct branches" vs. "`main`: 0 merges, last commit 4 months ago"). This is for teams that
don't run textbook git-flow and instead converge all work on something like `UAT` or `SIT` —
detection goes by actual activity, not by name.

## Scheduling it weekly

The command itself just answers when you invoke it — turning it into a *standing* weekly
report needs something to fire it on a schedule. Two ways to do that, pick whichever fits
where you run Claude Code:

### Option A — Claude Code on the web / Remote (Routines)
If you're using Claude Code on the web (or another environment with `claude-code-remote`
Routines available), create a Routine that fires `/weekly-report` into a session with this
repo attached, e.g. every Monday 09:00 Bangkok time (02:00 UTC):

- Cron: `0 2 * * 1` (UTC)
- Prompt: `/weekly-report` (auto-detects the integration branch from merge activity) or
  `/weekly-report UAT` to pin it explicitly — see note below on why pinning is often safer for
  a fresh-clone Routine
- Target: a session (existing or fresh-per-fire) with the repo you want reported on attached

Ask Claude in that environment to set this up for you — it can create the Routine directly.
You'll need to tell it: which repo, which branch, which day/time, and where you want the
report delivered (back into the session, Slack, email, etc.).

### Option B — local cron + headless Claude Code
If you're running Claude Code CLI locally against a repo on disk:

```cron
# Monday 09:00 local time — fetch everything so auto-detection can see all branches' activity
0 9 * * 1  cd /path/to/your/repo && git fetch --all --prune && claude -p "/weekly-report" >> reports/git-weekly/cron.log 2>&1
```

Requires this plugin's `commands/` to be available to that `claude` invocation (installed
globally, or referenced via `--plugin-dir`/`CLAUDE_PLUGIN_DIRS`) and `claude -p` (headless/
print mode) to be authenticated non-interactively.

> **Repos where a team doesn't run textbook git-flow** — e.g. everything converges on `UAT` or
> `SIT` instead of `develop`/`main`: leave the branch unspecified and let auto-detection find
> the real integration point from merge activity, or name it explicitly
> (`/weekly-report UAT`) if you already know it and want to skip the detection step (slightly
> faster, and immune to a quiet period where the usual branch temporarily looks less active
> than a short-lived one). For a fresh-clone automation environment, run `git fetch --all` (not
> just the default branch) first so every candidate branch's history is actually there to
> analyze — a shallow/single-branch clone will make detection blind to the rest.

## What it will and won't do
- **Will**: read real git log/diff output, cite commit hashes for every claim, distinguish
  "introduced this week" from pre-existing debt where the history supports it, flag reverts /
  repeatedly-patched areas instead of just listing commits, and pick the real integration
  branch by merge activity instead of assuming `main`/`develop` by name.
- **Won't**: invent a root cause the diff doesn't support (it says "unclear from commit
  message" instead of guessing), silently drop commits outside the window — it always states
  the window it used — or detect branches it can't see: a shallow or single-branch clone
  limits auto-detection to what's actually fetched, so it says so rather than guessing blind.
