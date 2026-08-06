# Comment Cleanup

A single-skill plugin for the comment litter that piles up after an agent (or a few rounds
of agents) work through a task — plan-narrating comments, "fixed this for X" markers, notes
left to coordinate with another pass. It cleans those up while leaving genuine WHY-comments
alone.

- **`cleanup-comments`** (skill) — scopes to what actually changed (`git diff` by default, or
  files you name), classifies every comment as restated-code / scratch-artifact / genuine-WHY
  / not-a-comment-at-all (docstrings, directives, headers), deletes the first two, keeps the
  third, never touches the fourth, and flags anything genuinely ambiguous instead of guessing.

## Folder structure
```
comment-cleanup/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   └── cleanup-comments/
│       └── SKILL.md
└── README.md
```

## Install — Claude Code
```bash
# one-off for a single session
claude --plugin-dir /path/to/comment-cleanup

# or, to keep it available, add its parent folder as a local marketplace
/plugin marketplace add /path/to    # the folder that CONTAINS comment-cleanup
/plugin install comment-cleanup
```

Then in a session:
```
/cleanup-comments
```
(with no args it defaults to the current diff against the default branch)

## Install — Claude Cowork
Skills are the portable format across Claude Code and Cowork. Install this plugin folder in
Cowork's plugin settings, then ask something like "clean up the comments in this diff" — the
`cleanup-comments` skill triggers on that.

## What it will and won't do
- **Will**: delete comments that only restate readable code, delete agent scratch/coordination
  notes, keep comments that explain a real non-obvious WHY, and flag anything ambiguous
  (commented-out code, reasoned TODOs) instead of silently deciding for you.
- **Won't**: touch docstrings/API docs, lint/type directives, or license headers — those
  aren't narration, they're part of the contract. It also never adds or rewrites comments,
  only removes.
