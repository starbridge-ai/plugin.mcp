# Contributing

Notes for developers working on the Starbridge Claude Code plugin. End-user docs live in
[`README.md`](./README.md).

## Test locally

Point Claude Code at the checked-out directory instead of installing from the marketplace:

```bash
claude --plugin-dir /path/to/starbridge-mcp-plugin
```

Run `/reload-plugins` after editing any plugin file.

## Layout

```text
starbridge-mcp-plugin/
├── .claude-plugin/
│   ├── plugin.json        # plugin manifest
│   └── marketplace.json   # makes this repo installable as a marketplace
├── .mcp.json              # Starbridge OAuth MCP server (HTTP transport)
├── README.md              # end-user docs
└── CONTRIBUTING.md        # this file
```

## MCP server

The plugin points at `https://dashboard.starbridge.ai/mcp/oauth` (HTTP transport, OAuth 2.0). Claude
Code handles dynamic client registration + PKCE automatically. Run `/mcp` after connecting to see
the live tool list.

## Releasing

Bump the `version` in both `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`, then
run `claude plugin validate .`.
