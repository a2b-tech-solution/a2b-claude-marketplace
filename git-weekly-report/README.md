# Git Weekly Report

A slash command that reads a repo's git history for the past week, summarizes code activity,
and — the main point — analyzes what problems came up and how they were actually fixed. Meant
to be run on a schedule so you get a standing weekly digest without asking for it each time.

- **`/weekly-report`** (command) — pulls commits/merges since a window (default: 7 days),
  classifies them (feat/fix/refactor/chore/revert/...), and builds an "Issues & Fixes" section
  that pairs each bug fix with what broke and what resolved it, flagging areas that got patched
  more than once (a sign of an unstable spot worth a closer look).

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
/weekly-report develop         # report on the develop branch instead of whatever's checked out
/weekly-report develop 2 weeks ago
```

No branch is hardcoded to `main`/`master` — it reads whichever branch is checked out (`HEAD`)
by default, or the branch you name explicitly. Works the same for repos where work lands
straight on `develop`, a release branch, or anything else.

## Scheduling it weekly

The command itself just answers when you invoke it — turning it into a *standing* weekly
report needs something to fire it on a schedule. Two ways to do that, pick whichever fits
where you run Claude Code:

### Option A — Claude Code on the web / Remote (Routines)
If you're using Claude Code on the web (or another environment with `claude-code-remote`
Routines available), create a Routine that fires `/weekly-report` into a session with this
repo attached, e.g. every Monday 09:00 Bangkok time (02:00 UTC):

- Cron: `0 2 * * 1` (UTC)
- Prompt: `/weekly-report develop` (name the branch explicitly if it isn't the repo's GitHub
  default branch — see note below)
- Target: a session (existing or fresh-per-fire) with the repo you want reported on attached

Ask Claude in that environment to set this up for you — it can create the Routine directly.
You'll need to tell it: which repo, which branch, which day/time, and where you want the
report delivered (back into the session, Slack, email, etc.).

### Option B — local cron + headless Claude Code
If you're running Claude Code CLI locally against a repo on disk:

```cron
# Monday 09:00 local time
0 9 * * 1  cd /path/to/your/repo && git fetch origin develop && git checkout develop && git pull --ff-only && claude -p "/weekly-report" >> reports/git-weekly/cron.log 2>&1
```

Requires this plugin's `commands/` to be available to that `claude` invocation (installed
globally, or referenced via `--plugin-dir`/`CLAUDE_PLUGIN_DIRS`) and `claude -p` (headless/
print mode) to be authenticated non-interactively.

> **Repos where work pushes straight to `develop`:** the command itself has no `main`/`master`
> assumption — it reads whatever branch is checked out. The thing to watch is automation that
> *clones fresh* on each run (a fresh Routine session, a CI-triggered clone): a fresh clone
> checks out the repo's configured GitHub default branch, which may be `main` even if `develop`
> is where the real weekly activity happens. In that case either pass the branch explicitly
> (`/weekly-report develop`) or checkout it first, as in the cron example above.

## What it will and won't do
- **Will**: read real git log/diff output, cite commit hashes for every claim, distinguish
  "introduced this week" from pre-existing debt where the history supports it, and flag
  reverts / repeatedly-patched areas instead of just listing commits.
- **Won't**: invent a root cause the diff doesn't support (it says "unclear from commit
  message" instead of guessing), or silently drop commits outside the window — it always
  states the window it used.
