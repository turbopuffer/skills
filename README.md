# turbopuffer skills

Skills and plugins for AI coding agents working with [turbopuffer](https://turbopuffer.com).

Works with **Claude Code** and **Cursor**.

## Install

### Claude Code

```bash
/plugin marketplace add turbopuffer/skills
/plugin install turbopuffer@turbopuffer-skills
```

### Cursor

Browse the [Cursor Marketplace](https://cursor.com/marketplace) and search for "turbopuffer", or use `/add-plugin` in chat.

### Configuration

The skill calls the turbopuffer API with your credentials. Get a key at [turbopuffer.com/dashboard](https://turbopuffer.com/dashboard) and pick a [region](https://turbopuffer.com/docs/regions).

```bash
export TURBOPUFFER_API_KEY=tpuf_...
export TURBOPUFFER_REGION=gcp-us-central1
```

## Local development

```bash
git clone https://github.com/turbopuffer/skills.git
cd skills
npm install
```

### Claude Code

```bash
mkdir -p ~/.claude/plugins/local
ln -s "$(pwd)/plugins/turbopuffer" ~/.claude/plugins/local/turbopuffer
```

Restart Claude Code. Remove with `rm ~/.claude/plugins/local/turbopuffer`.

### Cursor

```bash
mkdir -p ~/.cursor/plugins/local
ln -s "$(pwd)/plugins/turbopuffer" ~/.cursor/plugins/local/turbopuffer
```

Restart Cursor. Remove with `rm ~/.cursor/plugins/local/turbopuffer`.

### Linting

```bash
npm run lint           # check
npm run lint:fix       # auto-fix
```

## License

MIT
