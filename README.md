# mairp's Claude Code plugins

A [Claude Code](https://claude.com/claude-code) plugin marketplace — practical
tools for AI infrastructure, security, and agent operations.

## Use it

```
/plugin marketplace add mairp/claude-plugins
/plugin install <plugin>@mairp
```

Then `/plugin` to browse, enable, or disable what you installed.

## Plugins

| Plugin | What it does |
|---|---|
| [`antares-scan`](https://github.com/mairp/antares-scan) | Fast, free, **first-pass** security smell test. Points a tiny local LLM (antares-1b, a ~2 GB Granite-4.0 fine-tune) at a folder for cheap vulnerability leads across a codebase in seconds — a triage aid before a deeper audit, **not** a substitute for Semgrep, CodeQL, or Bandit. |

Install a single plugin, e.g.:

```
/plugin marketplace add mairp/claude-plugins
/plugin install antares-scan@mairp
```

## License

Each plugin carries its own license (see its repository). This marketplace
catalog is Apache-2.0.
