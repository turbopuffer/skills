# Turbopuffer Skills

Skills and plugins for AI coding agents working with turbopuffer — a serverless vector and full-text search database.

## Commands

```bash
npm run lint           # check with biome
npm run lint:fix       # auto-fix with biome
npm run format         # format with biome
npm run format:check   # check formatting
```

## Project Structure

```
.claude-plugin/          Claude Code marketplace catalog
.cursor-plugin/          Cursor marketplace catalog
plugins/turbopuffer/     the turbopuffer plugin
  .claude-plugin/        Claude Code plugin manifest
  .cursor-plugin/        Cursor plugin manifest
  CLAUDE.md              plugin-level instructions
  skills/tpuf/           the tpuf skill
    SKILL.md             router — matches user intent to the right reference
    references/          topic files loaded on demand
```

## Plugin Model

One orchestration skill (`tpuf`) with a routing table in SKILL.md. The skill matches user intent to the correct reference file and loads it on demand. The same skill works in both Claude Code and Cursor — only the manifest directories differ.

## Key Conventions

- **Router pattern**: SKILL.md routes requests to the right reference. References are loaded on demand, not all at once.
- **Topic files are hand-maintained**: no generation pipeline. Each reference is a curated guide for one topic.
- **Gotchas encode LLM errors**: the "Gotchas" section in SKILL.md captures mistakes the LLM commonly makes.
- **MCP server**: the plugin registers a turbopuffer MCP server that provides `execute` and `search_docs` tools.
- **Dual-platform manifests**: `.claude-plugin/` and `.cursor-plugin/` contain identical plugin.json — keep them in sync.

## Authoring Guidance

- Keep references focused on one topic each
- Include concrete code examples in references
- Update gotchas when evals reveal new LLM failure modes
- Don't put multiple topics in one reference file
- Don't add frontmatter to reference files (only SKILL.md has frontmatter)
- Bump version in both `.claude-plugin/plugin.json` and `.cursor-plugin/plugin.json` when updating skills
