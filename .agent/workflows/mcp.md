---
description: MCP (Model Context Protocol) server building workflow. Design tools, resources, and prompts for AI integrations. Use when creating new MCP servers or adding capabilities to existing ones.
---

# /mcp - Build MCP Server

$ARGUMENTS

---

## Purpose

This command activates the `mcp-builder` skill to design and build MCP (Model Context Protocol) servers. These servers expose tools, resources, and prompts that AI systems can use.

---

## Sub-commands

```
/mcp create <name>     - Create new MCP server from scratch
/mcp add tool <name>   - Add a new tool to existing server
/mcp add resource      - Add a new resource endpoint
/mcp test              - Test MCP server tools
/mcp config            - Generate configuration for Claude Desktop
```

---

## Behavior

When `/mcp` is triggered:

### Step 1: Clarify Requirements

Ask:
1. What will this MCP server do? (tools/resources/prompts)
2. Transport type: **Stdio** (CLI), **SSE** (web), or **WebSocket** (real-time)?
3. What external APIs or data sources will it connect to?
4. Will it handle files, images, or multimodal data?

### Step 2: Project Structure

```
my-mcp-server/
├── src/
│   ├── index.ts          # Main entry point & server setup
│   ├── tools/            # Individual tool implementations
│   │   ├── get-data.ts
│   │   └── create-item.ts
│   ├── resources/        # Resource handlers
│   └── types.ts          # Shared types & schemas
├── package.json
└── tsconfig.json
```

### Step 3: Tool Design Principles

Every tool MUST have:

```typescript
{
  name: "action_noun",           // action-oriented: get_weather, create_user
  description: "Clear...",       // what it does, when to use it
  inputSchema: {
    type: "object",
    properties: {
      param: {
        type: "string",
        description: "What this param does"  // required!
      }
    },
    required: ["param"]
  }
}
```

**Good tool names**: `get_weather`, `create_user`, `search_documents`
**Bad tool names**: `tool1`, `do_thing`, `process`

### Step 4: Resource URI Patterns

| Pattern | Example | Use For |
|---------|---------|---------|
| Fixed | `docs://readme` | Static documents |
| Parameterized | `users://{userId}` | Dynamic records |
| Collection | `files://project/*` | Grouped data |

### Step 5: Security Checklist

Before shipping, verify:
- [ ] All tool inputs validated with schema
- [ ] API keys in environment variables only
- [ ] No secrets logged
- [ ] Resource access is scoped (least privilege)
- [ ] Error messages don't expose internals

### Step 6: Claude Desktop Configuration

```json
{
  "mcpServers": {
    "my-server": {
      "command": "node",
      "args": ["path/to/dist/index.js"],
      "env": {
        "API_KEY": "your-key-here"
      }
    }
  }
}
```

---

## Output Format

```markdown
## 🔌 MCP Server: [Name]

### Architecture
- Transport: [Stdio/SSE/WebSocket]
- Tools: [N tools]
- Resources: [N resources]
- External deps: [APIs/services]

### Tools Designed
| Tool | Purpose | Input | Output |
|------|---------|-------|--------|
| get_data | Fetches X | `{id: string}` | `{data: object}` |
| create_item | Creates Y | `{name, type}` | `{id, status}` |

### Resources Designed
| URI | Type | Description |
|-----|------|-------------|
| `config://settings` | Static | App configuration |
| `users://{id}` | Dynamic | User profile |

### Security Review
- [ ] Input validation: ✅
- [ ] Secrets management: ✅
- [ ] Error handling: ✅

### Configuration
[Claude Desktop JSON config block]
```

---

## Examples

```
/mcp create github-integration
/mcp create database-explorer
/mcp add tool search-documents
/mcp add resource project-files
/mcp test
/mcp config
```

---

## Key Principles

- **Single purpose per tool** - one thing, done well
- **Descriptions are critical** - the AI uses them to decide which tool to call
- **Always validate inputs** - garbage in, errors out
- **Fail gracefully** - structured errors, not crashes
- **Never log secrets** - environment variables only
