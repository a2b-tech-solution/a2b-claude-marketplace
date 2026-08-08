---
description: Summarize this week's git activity — what shipped, what broke, and what fixed it
argument-hint: "[since] [branch] [path] e.g. \"7 days ago\", \"develop\", \"../other-repo\" — any order"
---

# Weekly Git Activity Report

Respond in the same language the user writes in (Thai, English, or mixed — mirror them). If
this command was fired by a schedule with no user message to mirror, default to Thai.

## 1. Scope the window
Parse each token in `$ARGUMENTS` independently — they can appear in any combination:
- A time expression (e.g. "2 weeks ago", "last monday", a date) → use it as `--since` instead
  of the default `--since="7 days ago"`.
- A path (exists as a directory) → the repo/subdir to scope into (`cd` there, or `git -C <path>`).
- Anything else that resolves via `git rev-parse --verify <token>` → treat it as the branch/ref
  to read history from (e.g. `develop`, `release/2.4`, a remote-tracking ref).

No branch name is assumed or hardcoded. Read history from whichever branch is checked out
(`HEAD`) at the time the command runs, or the branch named explicitly in `$ARGUMENTS` if given
— this works identically whether the team's primary line of development is `main`, `master`,
`develop`, or anything else. Do not special-case `main`/`master` as "the real branch" and
`develop` as a variant; treat whatever branch is in scope as the one to report on.
- Also include any branches merged into the scoped branch during the window, so a squash-merge
  or long-lived feature branch's history isn't silently dropped.
- Confirm you're in a git repo (`git rev-parse --is-inside-work-tree`). If not, say so and stop
  — don't guess a path.
- If a branch was given but doesn't exist locally, try `git fetch origin <branch>` before
  giving up — automation environments often only have the default branch checked out locally.

## 2. Pull the raw history
Gather, don't summarize yet:
- `git log --since="<window>" --pretty=format:'%h|%ad|%an|%s' --date=short --no-merges`
  for the individual commits, and separately the merge commits
  (`--merges --pretty=format:'%h|%ad|%an|%s'`) to catch PR titles.
- `git log --since="<window>" --stat --no-merges` (or `--numstat` if you need to compute
  totals) for what files/areas changed and how much.
- If commit messages reference issue/ticket IDs (`#123`, `JIRA-456`, etc.), note them — you'll
  need them for the Issues & Fixes section.

## 3. Classify every commit
Bucket each commit using its conventional-commit prefix if present (`feat`, `fix`, `refactor`,
`chore`, `docs`, `test`, `perf`, `revert`, `build`, `ci`), or infer from the message/diff when
there's no prefix. Also flag:
- **Reverts** — a `revert` commit or a commit that undoes a recent one; these often mean a
  fix didn't stick.
- **Fix-after-fix pairs** — two or more commits touching the same file/area within the window
  where a later one fixes an earlier one; this is a signal of an unstable area, call it out.
- **Merge commits without a body** — still useful for grouping commits by PR.

## 4. Build the "Issues & Fixes" analysis
This is the core of the report, not an afterthought. For every `fix`/`bugfix`/`hotfix`-flavored
commit (and every revert):
1. **What broke** — infer the problem from the commit message and the diff (what changed,
   what area/file). Don't invent a root cause you can't support from the diff — say "unclear
   from commit message" rather than guessing.
2. **How it was fixed** — the actual change, in one line, `file:line` or file-level if the diff
   is large.
3. **Introduced when** — if you can find the commit that introduced the bug (via `git log -S`
   or `git blame` on the fixed lines, or a nearby earlier commit touching the same code), say
   whether it was introduced this window or was pre-existing debt.

Group related fixes together (e.g. three commits all patching the same auth flow = one story,
not three bullets).

## 5. Write the report
Produce (and if writing to a file, save under `reports/git-weekly/<end-date>.md`, creating the
directory if it doesn't exist — otherwise just answer inline if there's no obvious place to
save in this repo):

1. **Overview** — window covered, commit count, contributors, files touched, rough size
   (lines added/removed).
2. **Activity by type** — a short table or list: features, fixes, refactors, chores, etc. with
   counts.
3. **Issues & Fixes** — the analysis from §4, the main content. What went wrong this week and
   what specifically resolved it. Call out any fix-after-fix pairs or reverts as areas that
   may need more attention, not just a clean log entry.
4. **Notable changes** — features/refactors worth flagging even though nothing broke.
5. **Open threads** — reverts without a visible follow-up fix, `WIP`/`TODO`-flagged commits,
   or areas that got patched multiple times and might warrant a closer look next week.

Keep it tight and evidence-based — cite commit hashes (`abc1234`) so anyone can verify a claim
against the actual log. Use the Thai section headings below when responding in Thai:
สรุปภาพรวม / กิจกรรมแยกตามประเภท / ปัญหาที่เกิดและวิธีแก้ / การเปลี่ยนแปลงที่น่าสนใจ / ประเด็นที่ยังค้างอยู่.
