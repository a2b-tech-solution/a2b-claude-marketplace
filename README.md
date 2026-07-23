# a2b-marketplace

A personal Claude Code plugin marketplace. It is a catalog (`.claude-plugin/marketplace.json`)
plus a folder per plugin. Add it once, then install any plugin it lists — and keep adding
your own over time.

## Structure

```
marketplace/
├── .claude-plugin/
│   └── marketplace.json        # the catalog — lists every plugin and where it lives
├── spec-crystallizer/          # a plugin (lives UNDER the marketplace)
│   ├── .claude-plugin/plugin.json
│   ├── skills/spec-interview/SKILL.md
│   ├── agents/spec-cold-reader.md
│   └── README.md
└── README.md                   # this file
```

## First-time setup (Claude Code)

Unzip this so the folder lands at:
`/Users/mayza/Library/CloudStorage/OneDrive-Personal/claude-cowork/marketplace`

Then, from inside Claude Code:

```
/plugin marketplace add /Users/mayza/Library/CloudStorage/OneDrive-Personal/claude-cowork/marketplace
/plugin install spec-crystallizer@a2b-marketplace
```

`a2b-marketplace` is the `name` field in `marketplace.json`; that's the part after `@` when
you install. Verify the catalog anytime with:

```
cd /Users/mayza/Library/CloudStorage/OneDrive-Personal/claude-cowork/marketplace
claude plugin validate .
```

## Adding a NEW plugin later

1. Create a folder for it under `marketplace/`, e.g. `marketplace/my-next-plugin/`, with its
   own `.claude-plugin/plugin.json` and whatever it ships (`skills/`, `agents/`, `commands/`,
   `hooks/`).
2. Add an entry to the `plugins` array in `.claude-plugin/marketplace.json`:

   ```json
   {
     "name": "my-next-plugin",
     "description": "What it does.",
     "source": "./my-next-plugin",
     "category": "development"
   }
   ```

3. Refresh and install:

   ```
   /plugin marketplace update a2b-marketplace
   /plugin install my-next-plugin@a2b-marketplace
   ```

## Storing a lone skill (not a full plugin)

The marketplace distributes *plugins*, so wrap a standalone skill as a minimal plugin:

```
my-skill-plugin/
├── .claude-plugin/plugin.json      # name, version, description
└── skills/
    └── my-skill/
        └── SKILL.md
```

Then list it in `marketplace.json` the same way as above. A plugin can hold several skills
at once, so you can also group related skills into one plugin instead of one-per-plugin.

## Notes

- The plugin `name` is an immutable slug once installed — renaming it later breaks existing
  installs. Change the display label via `displayName` in the plugin's `plugin.json` instead.
- Local-path marketplaces resolve each plugin's `source` relative to this marketplace folder,
  so keep plugins inside it and use `./<folder>` paths. Use a recent Claude Code version.
- To share across machines later, push this whole `marketplace/` folder to a git repo and
  `add` it by URL instead of local path — same `marketplace.json`, no other changes.
