# Sidd's Claude Code Plugins

A personal marketplace of plugins for [Claude Code](https://code.claude.com/docs/en/plugins), modeled after [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official).

> **Trust before you install.** Anthropic does not vet third-party plugins. Read each plugin's source before enabling it.

## Install this marketplace

In Claude Code:

```
/plugin marketplace add siddvoh/sidds-claude-plugins
```

Then browse with `/plugin` and pick something to install, or jump straight to one:

```
/plugin install research@sidds-claude-plugins
```

## Layout

```
.
├── .claude-plugin/
│   └── marketplace.json      # The only file Claude Code actually requires
└── plugins/                  # Plugins maintained in this repo
    └── research/             # ai-analysis skill — fair AI/LLM comparisons
```

## Anatomy of a plugin

Every plugin is a folder with a `.claude-plugin/plugin.json` manifest. Optional pieces:

| Folder        | Purpose                                                                 |
|---------------|-------------------------------------------------------------------------|
| `skills/`     | Model-invoked capabilities AND user-invoked slash commands (preferred)  |
| `commands/`   | Legacy slash command format (use `skills/` for new plugins)             |
| `agents/`     | Subagent definitions                                                    |
| `hooks/`      | Lifecycle hook scripts                                                  |
| `.mcp.json`   | MCP server config                                                       |

See `plugins/hello-sidd/` for a minimal working example.

## Adding a new plugin

1. Create a folder under `plugins/<your-plugin>/`.
2. Add `.claude-plugin/plugin.json` with `name`, `description`, `author`.
3. Add a skill, command, agent, or hook.
4. Register the plugin in `.claude-plugin/marketplace.json` with `"source": "./plugins/<your-plugin>"`.
5. Commit and push. The marketplace updates as soon as the new commit is on `main`.

## License

MIT, see [LICENSE](LICENSE). Each plugin may carry its own license, check inside the plugin folder.
