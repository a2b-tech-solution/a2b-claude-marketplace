---
description: Weekly radar for a whole project — rolls up git activity across every repo the project uses (including bare mirrors) into one document, and flags risk building up, not just what already broke
argument-hint: "<project-name> [since] [config-path] — e.g. \"Project A\", \"2 weeks ago\""
---

# Weekly Project Radar (multi-repo)

Respond in the same language the user writes in (Thai, English, or mixed — mirror them). If
this command was fired by a schedule with no user message to mirror, default to Thai.

This command exists because a project is often more than one repository (backend, frontend,
mobile, infra, ...) and the useful unit of reporting is the *project*, not any single repo.
It also exists to work off **bare mirror repos** kept separate from anyone's working checkout
— a common setup for a monitoring box that just tracks history without touching a live project.

## 1. Resolve the project's repos

### Find the config
A config maps a project name to the repos it's made of. Look, in order:
1. A config path given explicitly as a token in `$ARGUMENTS` (ends in `.json`).
2. `./weekly-report.projects.json` in the current directory.
3. `~/.config/git-weekly-report/projects.json`.

If none of those exist, and the remaining token in `$ARGUMENTS` doesn't obviously name a
project you can otherwise identify, **ask the user for the repo paths directly** rather than
guessing — then offer to write a config file for next time (§5) so they don't have to repeat
themselves.

Config shape:
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
`branch` is optional per repo — omit it to auto-detect (same algorithm as `/weekly-report`
§1, run independently per repo, since different repos in one project can easily have different
integration branches).

### Match the project
Match the project-name token in `$ARGUMENTS` against `projects[].name`, case-insensitively. If
more than one config file defines a project by that name, say so and ask which — don't
silently pick one. If the name matches nothing, list the project names that *are* available
before falling back to asking for repo paths.

## 2. Sync each repo (bare-aware)
A bare mirror only reflects the remote as of its last fetch, so the report is only as fresh as
that sync:
- Detect bare vs. working-tree per repo: `git -C <path> rev-parse --is-bare-repository`.
- Bare: refresh with `git --git-dir=<path> remote update --prune` (falls back to
  `git --git-dir=<path> fetch --all --prune` if no tracking remotes are configured). Use
  `git --git-dir=<path>` for every subsequent command against that repo — a bare repo has no
  working tree to `cd` into, so never `cd` into one.
- Working tree: `git -C <path> fetch --all --prune` — fetch only, never `checkout`/`pull` here;
  this command reports on history, it doesn't move anyone's working copy. Use `git -C <path>`
  for subsequent commands.
- If a refresh fails (no network, remote misconfigured), don't abort — proceed with what's on
  disk, but say plainly in the report how stale that repo's data might be (its most recent
  commit timestamp vs. today).

## 3. Run the single-repo analysis per repo
For each repo in the project, run the same process as `/weekly-report` §§1 (branch
resolution)–4 (Issues & Fixes), scoped to that repo through the `$GIT` target from §2. Keep
each repo's raw findings — file/area names, commit hashes, timestamps — don't compress away
detail before the rollup in §4 needs it.

## 4. Roll up into one project-level radar
One document for the whole project, not one per repo:

1. **Project overview** — repos included, each one's branch (given vs. auto-detected + why),
   window covered, total commits/contributors/files across all repos, and each bare repo's
   sync freshness from §2.
2. **Per-repo summary** — one compact block per repo: activity-by-type counts and its own
   Issues & Fixes list.
3. **Cross-repo correlation** — fixes that plausibly span more than one repo: look for fixes
   landing in different repos within roughly 24-48h of each other, touching related-sounding
   areas, sharing a ticket/issue ID referenced in more than one repo's commit messages, or an
   obvious pattern like a frontend fix immediately following a backend fix to the same
   feature. State each as a hypothesis with the evidence (commit hashes + timestamps from both
   repos), not a certainty — never assert a cross-repo link you can't point to evidence for.
4. **Radar — what's building up, not just what already broke.** The reason this command exists
   instead of just running `/weekly-report` per repo. Across every repo in the project:
   - **Repeated-patch hotspots** — any file/area fixed 2+ times this window, in any repo —
     carried over from each repo's own analysis, surfaced at the project level.
   - **Unmerged long-lived branches** — per repo, branches with commits not in the integration
     branch, aged beyond ~2 weeks: `$GIT branch --no-merged <integration-branch>` cross-checked
     against each candidate's last-commit date via `for-each-ref`. Flag as integration risk
     building up — the longer a branch diverges, the worse the eventual merge.
   - **Reverts without a visible follow-up fix** — project-wide, not just per-repo.
   - **Trend vs. last week** — if an earlier report for this project exists (§5), read it and
     compare fix-commit counts, revert counts, and hotspot counts per repo, e.g. "fix commits
     in `backend`: 12 this week vs. 5 last week, +140%". No earlier report → say plainly this
     is week one and trend comparison starts next week. Never fabricate a trend without a prior
     report to diff against.

Keep the Radar section short and scannable — it exists to point at where to look before
something becomes an incident, not to restate the Issues & Fixes section in different words.

## 5. Save the report
Save to `reports/git-weekly/<project-slug>/<end-date>.md` (relative to the current working
directory — bare repos have nowhere to write into), creating directories as needed. Before
writing, check that same directory for the most recent prior report and use it for the §4 trend
comparison. If §1 had to ask for repo paths ad hoc because no config existed, offer — don't
just do it — to write `weekly-report.projects.json` next to the reports directory (or wherever
the user prefers) so the next run doesn't need the paths re-typed.

Use the Thai section headings when responding in Thai: ภาพรวมโปรเจกต์ / สรุปแยกราย repo /
ความเชื่อมโยงข้าม repo / เรดาร์ - สัญญาณที่ต้องจับตา / บันทึกรายงาน.
