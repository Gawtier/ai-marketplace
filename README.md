# Forest Admin AI Marketplace

AI skills and plugins for Forest Admin integrations.

## Available Plugins

### forest-mcp

MCP server skill for querying and manipulating Forest Admin data. Provides tools for:
- Listing and searching records
- Creating, updating, and deleting records
- Exploring collection schemas and relations
- Filtering with comprehensive operators

### forest-code

Write and maintain Forest Admin agent customization code (backend only). Two skills, auto-selected from the project's `package.json` / `Gemfile`:
- **forest-code** — the modern agent (`@forestadmin/agent` Node.js + `forest_admin_agent` Ruby): actions, fields, hooks, segments, charts, relationships, datasources.
- **forest-legacy** — legacy agents (`forest-express-sequelize`, `forest-express-mongoose`, `forest-rails`): Smart Actions, Smart Fields, Smart Segments, Smart Collections, routes.

### forest-docs

Connects your AI client to the Forest Admin documentation MCP server (hosted by Mintlify): search and read the docs on demand. Works in any MCP-capable tool, not only Claude.

## Installation

### Claude Code

**Option 1: Via marketplace (recommended)**
```
/plugin marketplace add ForestAdmin/ai-marketplace
/plugin install forest-mcp
/plugin install forest-code
/plugin install forest-docs
```

**Option 2: Manual copy**

Copy the skill folder to your skills directory:

```bash
# Personal (all projects)
cp -r forest-mcp/skills/forest-mcp ~/.claude/skills/

# Project-specific (shared via git)
cp -r forest-mcp/skills/forest-mcp .claude/skills/
```

### Claude Desktop

1. Create a ZIP of the skill folder:
   ```bash
   cd forest-mcp/skills
   zip -r forest-mcp.zip forest-mcp/
   ```

2. In Claude Desktop:
   - Go to **Settings > Capabilities**
   - Enable **Code execution and file creation**
   - In the Skills section, click **Upload skill**
   - Select the `forest-mcp.zip` file

## Repository Structure

```
ai-marketplace/
├── .claude-plugin/
│   └── marketplace.json          # Marketplace catalog
├── forest-mcp/                   # Plugin (skill)
│   ├── .claude-plugin/plugin.json
│   └── skills/forest-mcp/
├── forest-code/                  # Plugin (skills)
│   ├── .claude-plugin/plugin.json
│   └── skills/
│       ├── forest-code/          # modern agent
│       └── forest-legacy/        # legacy agents
└── forest-docs/                  # Plugin (MCP)
    ├── .claude-plugin/plugin.json
    └── .mcp.json                 # registers the docs MCP server
```

## License

MIT
