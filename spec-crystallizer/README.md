# Spec Crystallizer

A three-part plugin for turning a rough idea into an MVP-ready spec, without a tool steering
you off course.

- **`spec-interview`** (skill) — an interactive, *non-leading* interviewer. It asks open
  questions, reflects your own understanding back to you (rubber-duck style), tracks
  assumptions and open questions out loud, and stops at "clear enough for an MVP" instead of
  chasing 100%. It writes the spec in EARS notation to `specs/<slug>.md`.
- **`spec-cold-reader`** (subagent) — a *zero-context* critic. You hand it only the spec
  file. Because it knows nothing you said but didn't write, whatever it has to guess is
  exactly the thing that was only ever in your head. It mirrors its understanding back and
  flags every gap — it never fixes or redesigns.
- **`spec-fact-finder`** (subagent) — a *reality-grounding* researcher. You hand it the cold
  reader's report plus the codebase/data it should search. It maps each open question to the
  spec aspect it belongs to, digs up what actually exists in code/config/data, and lays out
  the real options side by side — without picking one. It turns "I have no idea" into "here's
  what's really out there to choose from."

## The loop

```
/spec-interview <one-line idea>
        │  interview (non-leading) + rubber-duck playback
        ▼
specs/<slug>.md   (draft — EARS, with Assumptions + Open questions kept, not buried)
        │  hand the file to the cold reader
        ▼
spec-cold-reader specs/<slug>.md
        │  "here's what I understand / here's what I had to guess / here's what's ambiguous"
        ▼
(optional) spec-fact-finder <cold-reader report> + <codebase>
        │  "here's what's real, mapped to each open question, with grounded options"
        ▼
compare the cold reader's understanding, and the fact finder's options, with what you MEANT
        │  the mismatches = your hidden assumptions
        ▼
another short /spec-interview pass to close them → repeat until they match
```

## Folder structure

```
spec-crystallizer/
├── .claude-plugin/
│   └── plugin.json          # manifest (the ONLY file in this directory)
├── skills/
│   └── spec-interview/
│       └── SKILL.md         # the interviewer
├── agents/
│   ├── spec-cold-reader.md  # the cold-read critic
│   └── spec-fact-finder.md  # the reality-grounding researcher
└── README.md
```

## Install — Claude Code

Unzip anywhere, then load the plugin directory:

```bash
# one-off for a single session
claude --plugin-dir /path/to/spec-crystallizer

# or, to keep it available, add its parent folder as a local marketplace
/plugin marketplace add /path/to    # the folder that CONTAINS spec-crystallizer
/plugin install spec-crystallizer
```

Then in a session:

```
/spec-interview a queue-monitor dashboard for our loan SIT environment
```

and after a draft exists:

```
@spec-cold-reader specs/queue-monitor-dashboard.md
```

and, to ground the cold reader's open questions in real code/config/data:

```
@spec-fact-finder <cold reader's report> specs/queue-monitor-dashboard.md <where to search>
```

> Note: subagents are loaded at session start. If you edit `agents/spec-cold-reader.md` or
> `agents/spec-fact-finder.md` on disk, restart the session to pick up the change (edits made
> via `/agents` apply immediately).

## Install — Claude Cowork

Cowork customization is done through plugins that bundle skills and sub-agents, so the same
package works. Open Cowork's plugin settings and install this plugin folder, then start a
task with something like "help me spec the queue-monitor dashboard" (the `spec-interview`
skill triggers on that), later "use the spec cold reader on specs/queue-monitor-dashboard.md",
and optionally "use the spec fact finder to ground the cold reader's open questions in the
actual codebase".

## Design notes

- The interviewer is a **skill** (not a legacy slash command) on purpose: custom slash
  commands have been merged into skills, and skills are the format that is portable across
  both Claude Code and Cowork.
- The value of the split is the *opposite context properties* of the parts: the interviewer
  shares your context (good for dialogue) and therefore slowly stops being an outsider; the
  cold reader runs in an isolated context (good for catching what you left unsaid) and
  therefore stays a true outsider. The fact finder also runs isolated, but pointed outward at
  real code/data instead of at the spec text — it turns the cold reader's "I had to guess"
  into "here's what's actually out there," without ever choosing for you. Use all three.
- The fact finder is deliberately **read-only and non-leading**, same as the cold reader: it
  cites sources for every claim, never invents an option it didn't find evidence for, and
  never ranks or recommends between the real options it surfaces. Grounding the question in
  reality is not the same as answering it — the choice stays with the human.
- Keep the `⚠️ Assumptions` and `❓ Open questions` sections in every spec. Deleting them
  re-hides exactly the things this tool exists to surface.
