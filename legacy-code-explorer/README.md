# Legacy Code Explorer

A single-skill plugin for the moment before you refactor legacy code: you need to understand
it — the flow, the UI behavior, how pieces of logic actually relate — without an agent
quietly redesigning it on you.

- **`legacy-code-explorer`** (skill) — explores the codebase for evidence (`file:line`
  citations, not vibes), answers whatever you specifically asked about flow/UI/logic
  coupling, grades each area's refactor risk (Low/Medium/High/Critical), and writes it all up
  as a markdown knowledge document at `docs/legacy/<slug>.md`.

## Why this exists
Refactoring is supposed to change the *structure* while keeping the *output* identical. Legacy
code makes that hard because behavior often isn't written down anywhere — it's implied by call
order, a global mutated three files away, a UI branch that only fires on a rare input. This
skill's job is to surface that BEFORE you start restructuring, so "same output" is something
you can actually check yourself against, not just hope for.

## Folder structure
```
legacy-code-explorer/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   └── legacy-code-explorer/
│       └── SKILL.md
└── README.md
```

## Install — Claude Code
```bash
# one-off for a single session
claude --plugin-dir /path/to/legacy-code-explorer

# or, to keep it available, add its parent folder as a local marketplace
/plugin marketplace add /path/to    # the folder that CONTAINS legacy-code-explorer
/plugin install legacy-code-explorer
```

Then in a session:
```
/legacy-code-explorer explain how checkout flow updates the cart UI in src/checkout
```

## Install — Claude Cowork
Skills are the portable format across Claude Code and Cowork. Install this plugin folder in
Cowork's plugin settings, then start a task with something like "help me understand this
legacy code before I refactor it" — the `legacy-code-explorer` skill triggers on that.

## What it will and won't do
- **Will**: trace real code paths with citations, grade how risky each part is to touch,
  answer your specific questions, list the exact behaviors a refactor must preserve, and flag
  what it genuinely couldn't determine from reading alone.
- **Won't**: propose a new architecture, suggest a "better" structure, or write refactored
  code. It documents what IS — the refactor design is a separate step you drive yourself,
  informed by the knowledge document it produces.
