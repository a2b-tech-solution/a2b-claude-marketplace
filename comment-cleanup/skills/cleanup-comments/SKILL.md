---
name: cleanup-comments
description: Use when the user wants to clean up comments left in code — especially agent-written scratch notes used to coordinate a task (plan steps, "fixed this for X", references to a PR/issue/task) or comments that just restate what already-readable code does. Invoke with /cleanup-comments or when the user says things like "clean up these comments", "remove the comments the agent left", "ล้าง comment ที่ agent เขียนไว้คุยกัน". Do NOT use it to write new documentation, docstrings, or comments — it only removes, never adds.
---

# Cleanup Comments

You are cleaning up comments that piled up in code — most often left behind by an agent using
comments to narrate a plan or communicate with itself/another agent mid-task. Your job is to
strip out everything that doesn't need to survive the task, and leave only what a future
reader genuinely needs. You DELETE, you never ADD or REWRITE code logic.

## 1. Determine scope
- If the user names files/paths, use those.
- Otherwise default to what actually changed: `git diff` (staged + unstaged) against the
  current branch's merge-base with the default branch. This targets comments the recent work
  actually introduced, not a codebase-wide sweep.
- If scope is still ambiguous (e.g. no diff and no path given), ask before scanning the whole
  repo — a full-repo comment purge is a much bigger, riskier action than cleaning up one
  task's leftovers.

## 2. Classify every comment in scope
For each comment, decide which bucket it's in:

**DELETE — restates what the code already says**
The comment repeats what a reasonably-named identifier or the code's own structure already
communicates. Example: `// increment counter` above `counter++`, or `// loop through users`
above `for (const user of users)`.

**DELETE — agent scratch / coordination artifact**
Left behind to narrate a plan, mark progress, or communicate with another agent/pass rather
than to inform a future human reader. Signs: references the task/fix/PR/issue itself
("added for the caching fix", "per review comment", "changed to satisfy the linter"),
numbers out a plan ("// Step 1: parse input", "// Step 2: validate"), or is a bare marker
("// AI generated", "// done", "// TODO: revisit" with no reason attached).

**KEEP — explains a genuinely non-obvious WHY**
A hidden constraint, a subtle invariant, a workaround for a specific external bug, or
behavior that would surprise a reader encountering it cold. Test: if you deleted this comment,
would a careful reader be able to reconstruct the reasoning from the code alone? If no, keep
it. Example: `// API returns 200 with an error body on timeout — must check payload.status`.

**NEVER TOUCH — not an explanatory comment at all**
- Docstrings/API docs consumed by tooling (JSDoc, Sphinx/Python docstrings, Rustdoc, Godoc,
  TSDoc) — these are part of the public contract, not scratch notes.
- Directives: `eslint-disable`, `type: ignore`, `# noqa`, `@ts-expect-error`, pragmas,
  shebangs, license/copyright headers.
- Comments inside string literals or template content (not real comments — leave verbatim).

**FLAG, don't guess**
- Commented-out code blocks (might be deliberately disabled, not narration — ask rather than
  delete).
- `TODO`/`FIXME` that state a real, still-open concern with a reason — these look like scratch
  notes but may be legitimate backlog markers. List them for the user to decide.
- Anything you're genuinely unsure fits DELETE or KEEP.

## 3. Apply the edits
Remove DELETE-bucket comments with `Edit`, precisely — delete the comment line/inline
fragment only, never touch the code it was attached to. If removing a comment leaves a blank
line where there wasn't one before, clean that up too. Do not reflow, rename, or "improve"
anything else while you're in the file — this is a comment-only pass.

## 4. Report
End with a compact summary, not a wall of diff:
- Count removed (broken down: restated-code vs scratch/coordination), count kept, count
  flagged.
- File:line list for anything flagged in the "FLAG, don't guess" bucket, with the reason it's
  ambiguous, so the user decides rather than you guessing wrong in either direction.

Respond in the same language the user writes in (Thai, English, or mixed — mirror them).
