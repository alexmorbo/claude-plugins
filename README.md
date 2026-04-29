# claude-plugins

Personal collection of [Claude Code](https://docs.claude.com/en/docs/claude-code) plugins, distributed as a single marketplace.

## Available plugins

| Plugin | Description |
|--------|-------------|
| [go-standards](plugins/go-standards/) | Go microservices development standards (Clean Architecture, DDD, testing, linting, error handling, version-aware modernization, library catalog). 7 auto-activated skills, 5 slash commands (`/ca-init-go`, `/ca-validate-go`, `/go-review`, `/go-gen-test`, `/go-audit`), `goimports` + generated-files hooks. |

## Install

In any Claude Code session:

```
/plugin marketplace add git@github.com:alexmorbo/claude-plugins.git
/plugin install go-standards@claude-plugins
```

HTTPS variant if SSH isn't set up:

```
/plugin marketplace add https://github.com/alexmorbo/claude-plugins.git
```

After install, run `/plugin` to see installed plugins, enable/disable them, or browse skills/commands.

## Update

```
/plugin marketplace update claude-plugins
/plugin install go-standards@claude-plugins
```

## Uninstall

```
/plugin uninstall go-standards@claude-plugins
/plugin marketplace remove claude-plugins
```

## Repo layout

```
claude-plugins/
├── .claude-plugin/
│   └── marketplace.json          # Marketplace index (lists plugins below)
├── plugins/
│   └── go-standards/             # Each plugin lives in its own subdir
│       ├── .claude-plugin/plugin.json
│       ├── skills/
│       ├── commands/
│       ├── hooks/
│       ├── references/
│       └── README.md
└── README.md
```

## Adding a new plugin

1. Create `plugins/<name>/` with the same internal layout as `go-standards`.
2. Add `.claude-plugin/plugin.json` inside it.
3. Append an entry to `.claude-plugin/marketplace.json` → `plugins[]` with `"source": "./plugins/<name>"`.
4. Bump versions, commit, push.

Users installing for the first time pick up the new plugin immediately; existing users run `/plugin marketplace update claude-plugins` and then `/plugin install <name>@claude-plugins`.
