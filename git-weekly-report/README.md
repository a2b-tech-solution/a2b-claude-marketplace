# Git Weekly Report

Two slash commands that turn git history into a weekly **radar**: what shipped, what broke,
what fixed it, and — the point of the radar framing — what's building up that hasn't caused an
incident yet. Built for a monitoring setup where you keep **bare mirror repos** separate from
anyone's actual working checkout, and report **per project**, where a project can be more than
one repository.

- **`/weekly-report`** — single-repo report. Pulls commits/merges since a window (default: 7
  days), classifies them (feat/fix/refactor/chore/revert/...), and builds an "Issues & Fixes"
  section pairing each bug fix with what broke and what resolved it. Auto-detects the real
  integration branch from merge activity (not naming convention) if you don't name one, and
  works against a normal repo or a bare mirror.
- **`/weekly-project-report`** — rolls up `/weekly-report`'s analysis across every repo in a
  project (via a config file mapping project → repos), adds cross-repo correlation, and a
  **Radar** section: repeated-patch hotspots, long-lived unmerged branches, and — once a prior
  report exists — trend vs. last week. This is the one meant for "what's about to become a
  problem," not just "what already did."

## Folder structure
```
git-weekly-report/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   ├── weekly-report.md
│   └── weekly-project-report.md
├── weekly-report.projects.example.json
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

## `/weekly-report` — single repo

Inside (or pointed at) the repo you want a report for:
```
/weekly-report
```

Arguments are optional, in any order:
```
/weekly-report 2 weeks ago         # widen the window
/weekly-report ../other-repo       # a different repo/subdir, still last 7 days
/weekly-report /srv/bare/app.git   # a bare mirror — no working tree needed
/weekly-report UAT                 # pin the branch explicitly, skip auto-detection
/weekly-report SIT 2 weeks ago
```

No branch is hardcoded to `main`/`master`/`develop`. If you don't name one, it doesn't default
to whatever's checked out either — it ranks branches by merge activity (merges into each branch
over the last 90 days, how many distinct branches fed into it, recency) and picks whichever one
is actually functioning as the integration point, stating its evidence in the report (e.g.
"picked `SIT`: 14 merges in 90 days from 6 distinct branches" vs. "`main`: 0 merges, last commit
4 months ago"). Built for teams that don't run textbook git-flow and converge work on something
like `UAT` or `SIT` instead — detection goes by activity, not by name.

**Bare repos**: pass a path to a bare repo/mirror and it works the same way — every git command
goes through `--git-dir=<path>` instead of assuming a working tree to `cd` into. It never
`checkout`s or `pull`s; reading history doesn't require a working tree at all.

## `/weekly-project-report` — a project across several repos

Most real projects aren't one repository. This command reads a config that maps a **project
name** to the repos that make it up, runs the single-repo analysis on each, and produces one
document for the whole project.

Config file (default lookup: a path you pass explicitly, then `./weekly-report.projects.json`,
then `~/.config/git-weekly-report/projects.json` — see `weekly-report.projects.example.json`
in this folder):
```json
{
  "projects": [
    {
      "name": "Project A",
      "repos": [
        { "name": "backend", "path": "/srv/bare/project-a-backend.git", "branch": "SIT" },
        { "name": "frontend", "path": "/srv/bare/project-a-frontend.git" }
      ]
    }
  ]
}
```
`branch` is optional per repo — leave it out to auto-detect, same as `/weekly-report`. Different
repos in the same project can have different integration branches; each is detected/used
independently.

```
/weekly-project-report "Project A"
/weekly-project-report "Project A" 2 weeks ago
```

If no config is found and the project name doesn't resolve, it asks for repo paths directly
instead of guessing, then offers to write a config file so you don't retype them next time.

The report it produces: project overview, one summary block per repo (activity-by-type +
Issues & Fixes), a **cross-repo correlation** section (fixes landing in different repos within
~24-48h of each other, shared ticket IDs, etc. — stated as a hypothesis with evidence, never a
flat assertion), and the **Radar** section — repeated-patch hotspots, branches that have been
diverging unmerged for 2+ weeks (integration risk building up), reverts without a visible
follow-up, and, once a previous week's report exists in the same output folder, a trend
comparison (e.g. "fix commits in `backend`: 12 this week vs. 5 last week, +140%"). No prior
report → it says so and starts the trend baseline this week, rather than inventing one.

## A monitoring setup: bare mirrors kept separate from the live project

Both commands are built around this operational shape, not just tolerant of it:

1. Keep a **bare mirror** of each repo you want to monitor, separate from anyone's working
   checkout — e.g. `git clone --bare <url> /srv/bare/<repo>.git`. Nobody develops against these;
   they exist purely for history reads.
2. Before each run, refresh the mirrors: `git --git-dir=/srv/bare/<repo>.git remote update
   --prune` (both commands do this automatically for repos listed in a project config — see
   §2 of `/weekly-project-report`'s own instructions — but do it yourself first if running
   `/weekly-report` standalone against a mirror you manage outside a project config).
3. Point `/weekly-report` at one mirror, or `/weekly-project-report` at a project config
   listing several, and schedule it (below).

If a refresh fails (no network, misconfigured remote) the report still runs off whatever's on
disk, but says plainly how stale that data might be — it won't silently report as current.

## Scheduling it weekly

Neither command schedules itself — that needs something to fire it periodically. Two ways to do
that, pick whichever fits where you run Claude Code:

### Option A — Claude Code on the web / Remote (Routines)
If you're using Claude Code on the web (or another environment with `claude-code-remote`
Routines available), create a Routine that fires `/weekly-report` or `/weekly-project-report`
into a session with the mirrors/repos attached, e.g. every Monday 09:00 Bangkok time (02:00 UTC):

- Cron: `0 2 * * 1` (UTC)
- Prompt: `/weekly-project-report "Project A"` (or `/weekly-report` for a single repo)
- Target: a session (existing or fresh-per-fire) with access to the mirror paths/config used

Ask Claude in that environment to set this up for you — it can create the Routine directly.
You'll need to tell it: which project (or repo), which day/time, and where you want the report
delivered (back into the session, Slack, email, etc.).

### Option B — local cron + headless Claude Code
```cron
# Monday 09:00 local time
0 9 * * 1  claude -p '/weekly-project-report "Project A"' >> /var/log/git-weekly-report/project-a.log 2>&1
```
Run from wherever `weekly-report.projects.json` lives (or pass its path as an argument). For a
single repo instead: `claude -p "/weekly-report /srv/bare/app.git"`.

Requires this plugin's `commands/` to be available to that `claude` invocation (installed
globally, or referenced via `--plugin-dir`/`CLAUDE_PLUGIN_DIRS`) and `claude -p` (headless/
print mode) to be authenticated non-interactively.

## What it will and won't do
- **Will**: read real git log/diff output, cite commit hashes for every claim, pick the real
  integration branch by merge activity instead of assuming `main`/`develop` by name, work
  against bare mirrors without a working tree, roll multiple repos up into one per-project
  report, and surface risk signals (hotspots, long-unmerged branches, week-over-week trend)
  before they're an incident, not just log what already happened.
- **Won't**: invent a root cause the diff doesn't support (says "unclear from commit message"
  instead of guessing), assert a cross-repo correlation without commit-level evidence, fabricate
  a trend when no prior report exists to compare against, or silently report on stale mirror
  data without saying how stale — a shallow/single-branch clone or a failed mirror refresh
  limits what it can see, and it says so rather than guessing blind.
