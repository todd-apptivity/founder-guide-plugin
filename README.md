# Founder Guide plugin

Connects your [Founder Guide](https://www.founder-guide.com) workspace to your AI tool. The
plugin bundles two things:

- **The Founder Guide MCP server** (`https://www.founder-guide.com/api/mcp`). It lets your
  AI tool read and, with your permission, update your workspace. You sign in through
  Founder Guide in your browser, so you never paste a password or API key.
- **The `founder-guide-tools` skill**, which teaches your AI tool to use those tools well.

MCP connections come with the Operator, Builder, and Founder Circle subscriptions.

## Install

**Claude Code**

```
claude plugin install founder-guide --marketplace todd-apptivity/founder-guide-plugin
```

Or, inside Claude Code:

```
/plugin marketplace add todd-apptivity/founder-guide-plugin
/plugin install founder-guide@founder-guide
```

Then type `/mcp`, pick `founder-guide`, and sign in.

**Claude on the web, Claude Desktop, Cowork**: open Customize, then Plugins, add the
marketplace `todd-apptivity/founder-guide-plugin`, and install Founder Guide. Then connect it
from the plugin's Connectors tab.

**Other AI tools**: see [Connect Your AI Tool to Founder Guide](https://www.founder-guide.com/help/connecting-external-ai-agents-mcp).

## Updating

Claude Code: `claude plugin update founder-guide@founder-guide`, or turn on auto-update for
this marketplace under `/plugin`, then Marketplaces.

## About this repository

This repository is generated. Its source lives in Founder Guide's private repository and is
published here by `scripts/publish-plugin.mjs`. Pull requests here are overwritten on the
next publish; report issues at https://www.founder-guide.com/help instead.
