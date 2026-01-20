# Wanplugins

A collection of Claude Code plugins.

## Available Plugins

| Plugin | Description |
|--------|-------------|
| [fantasia](./fantasia) | Codebase-aware development accelerator. Map patterns, plan with understanding, build with parallel agents, review thoroughly. |

## Installation

Install any plugin directly from this repository:

```bash
# Install fantasia
claude plugin add https://github.com/wannabefro/wanplugins/fantasia

# Or clone and install locally
git clone https://github.com/wannabefro/wanplugins
claude plugin add ./wanplugins/fantasia
```

## Plugin Structure

Each plugin follows the standard Claude Code plugin format:

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json      # Plugin metadata (name, version, description)
├── commands/            # Slash commands (/plugin:command)
├── skills/              # Skills that can be invoked
├── agents/              # Specialized subagents
├── hooks/               # Event hooks (PreToolUse, PostToolUse, etc.)
└── README.md            # Plugin documentation
```

## Contributing

To add a new plugin:

1. Create a new directory with your plugin name
2. Add `.claude-plugin/plugin.json` with metadata
3. Add your commands, skills, agents, and hooks
4. Update this README with your plugin in the table
5. Submit a PR

## License

MIT
