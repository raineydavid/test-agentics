# 🎄 Day 21: MCP (Model Context Protocol)

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Days 1-20
- Node.js 18+ or Python 3.10+

## 📝 Overview

**MCP (Model Context Protocol)** is an open standard by Anthropic for connecting AI assistants to external tools and data sources!

### What is MCP?

```
┌─────────────────────────────────────────────────────────────┐
│                    MCP ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────┐         ┌──────────┐         ┌──────────┐     │
│  │  Claude  │ ←─MCP─→ │  Server  │ ←─────→ │   Data   │     │
│  │  (Host)  │         │ (Bridge) │         │  Source  │     │
│  └──────────┘         └──────────┘         └──────────┘     │
│                                                              │
│  Host: AI application (Claude Desktop, custom app)          │
│  Server: Provides tools, resources, prompts via MCP         │
│  Data Source: Databases, APIs, file systems, services       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### MCP Capabilities

| Capability | Description |
|------------|-------------|
| **Tools** | Functions the AI can call |
| **Resources** | Data the AI can read |
| **Prompts** | Pre-defined prompt templates |
| **Sampling** | Request LLM completions |

### Setting Up an MCP Server (TypeScript)

```typescript
// server.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
  {
    name: "my-mcp-server",
    version: "1.0.0",
  },
  {
    capabilities: {
      tools: {},
    },
  }
);

// Define tools
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "get_weather",
        description: "Get weather for a location",
        inputSchema: {
          type: "object",
          properties: {
            city: { type: "string", description: "City name" },
          },
          required: ["city"],
        },
      },
      {
        name: "calculate",
        description: "Perform calculations",
        inputSchema: {
          type: "object",
          properties: {
            expression: { type: "string" },
          },
          required: ["expression"],
        },
      },
    ],
  };
});

// Handle tool calls
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  if (name === "get_weather") {
    const city = args?.city as string;
    // In real implementation, call weather API
    return {
      content: [
        { type: "text", text: `Weather in ${city}: Sunny, 72°F` },
      ],
    };
  }

  if (name === "calculate") {
    const expression = args?.expression as string;
    try {
      const result = eval(expression);
      return {
        content: [{ type: "text", text: `Result: ${result}` }],
      };
    } catch (error) {
      return {
        content: [{ type: "text", text: `Error: ${error}` }],
        isError: true,
      };
    }
  }

  throw new Error(`Unknown tool: ${name}`);
});

// Start server
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("MCP Server running");
}

main();
```

### MCP Server (Python)

```python
# server.py
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

server = Server("my-mcp-server")

@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="get_weather",
            description="Get weather for a location",
            inputSchema={
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "City name"}
                },
                "required": ["city"]
            }
        ),
        Tool(
            name="search_database",
            description="Search the database",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {"type": "string"}
                },
                "required": ["query"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "get_weather":
        city = arguments["city"]
        # Call weather API
        return [TextContent(type="text", text=f"Weather in {city}: Sunny")]
    
    elif name == "search_database":
        query = arguments["query"]
        # Search database
        results = ["Result 1", "Result 2"]
        return [TextContent(type="text", text=str(results))]
    
    raise ValueError(f"Unknown tool: {name}")

async def main():
    async with stdio_server() as (read, write):
        await server.run(read, write)

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

### Resources in MCP

```python
from mcp.types import Resource

@server.list_resources()
async def list_resources():
    return [
        Resource(
            uri="file:///data/config.json",
            name="Configuration",
            description="Application configuration",
            mimeType="application/json"
        ),
        Resource(
            uri="db://users",
            name="Users Database",
            description="User records",
            mimeType="application/json"
        )
    ]

@server.read_resource()
async def read_resource(uri: str):
    if uri == "file:///data/config.json":
        config = load_config()
        return config
    
    elif uri.startswith("db://"):
        table = uri.replace("db://", "")
        data = query_database(table)
        return data
```

### Prompts in MCP

```python
from mcp.types import Prompt, PromptArgument

@server.list_prompts()
async def list_prompts():
    return [
        Prompt(
            name="code_review",
            description="Review code for best practices",
            arguments=[
                PromptArgument(
                    name="code",
                    description="The code to review",
                    required=True
                ),
                PromptArgument(
                    name="language",
                    description="Programming language",
                    required=False
                )
            ]
        )
    ]

@server.get_prompt()
async def get_prompt(name: str, arguments: dict):
    if name == "code_review":
        code = arguments["code"]
        language = arguments.get("language", "unknown")
        
        return {
            "messages": [
                {
                    "role": "user",
                    "content": f"""Review this {language} code for:
                    - Best practices
                    - Potential bugs
                    - Performance issues
                    
                    Code:
                    ```
                    {code}
                    ```"""
                }
            ]
        }
```

### Configuring Claude Desktop

```json
// claude_desktop_config.json
{
  "mcpServers": {
    "my-tools": {
      "command": "node",
      "args": ["path/to/server.js"]
    },
    "python-tools": {
      "command": "python",
      "args": ["path/to/server.py"]
    }
  }
}
```

### Using MCP in Custom Applications

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def use_mcp_server():
    server_params = StdioServerParameters(
        command="python",
        args=["server.py"]
    )
    
    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            # Initialize
            await session.initialize()
            
            # List tools
            tools = await session.list_tools()
            print(f"Available tools: {tools}")
            
            # Call a tool
            result = await session.call_tool(
                "get_weather",
                {"city": "Tokyo"}
            )
            print(f"Result: {result}")
```

## 📖 Reading Materials

1. [MCP Documentation](https://modelcontextprotocol.io/)
2. [MCP GitHub](https://github.com/modelcontextprotocol)
3. [Building MCP Servers](https://modelcontextprotocol.io/quickstart)

## ✅ Activity: Build an MCP Server

Create `day21-mcp-server.py` (or `.ts`) implementing:

1. At least 3 tools
2. At least 1 resource
3. At least 1 prompt template

### Complete This Day

[![Complete Day 21](https://img.shields.io/badge/Complete--Day--21-28a745?logo=github)](../../issues/new?title=Complete+Day+21&labels=complete-day-21&body=✅+I%27ve+completed+Day+21%21)

---

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1-20 | Previous Days | Complete |
| 🎁 **Day 21** | MCP Introduction | ✅ Current |
| 🎁 **Day 22** | Building Custom MCP Servers | 🔜 Next |
