# Turbopuffer — Claude Code Plugin

Use `/tpuf` for all turbopuffer operations. The skill routes to the right reference based on what the user wants to do.

## MCP Tools

- **`execute`** — runs TypeScript SDK code via `async function run(client) { ... }`
- **`search_docs`** — searches turbopuffer documentation

Requires `TURBOPUFFER_API_KEY` env var. If MCP tools aren't available, tell the user:

```
echo 'export TURBOPUFFER_API_KEY=tpuf_...' >> ~/.zshrc && source ~/.zshrc
```

Get a key at https://turbopuffer.com/dashboard — then restart Claude Code.
