# 🎄 Day 22: Building Custom MCP Servers

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Days 1-21
- Understanding of MCP basics

## 📝 Overview

Today we go deeper into building **production-ready MCP servers** with advanced patterns!

### Advanced MCP Server Patterns

```
┌─────────────────────────────────────────────────────────────┐
│              ADVANCED MCP PATTERNS                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Database Integration                                     │
│  2. API Wrappers                                             │
│  3. File System Access                                       │
│  4. Real-time Data Streaming                                 │
│  5. Authentication & Security                                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Database MCP Server

```python
import sqlite3
from mcp.server import Server
from mcp.types import Tool, TextContent, Resource

server = Server("database-server")

# Database connection
def get_db():
    return sqlite3.connect("data.db")

@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="query_database",
            description="Execute a SELECT query on the database",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "SQL SELECT query"
                    }
                },
                "required": ["query"]
            }
        ),
        Tool(
            name="get_table_schema",
            description="Get schema of a database table",
            inputSchema={
                "type": "object",
                "properties": {
                    "table_name": {"type": "string"}
                },
                "required": ["table_name"]
            }
        ),
        Tool(
            name="insert_record",
            description="Insert a record into a table",
            inputSchema={
                "type": "object",
                "properties": {
                    "table_name": {"type": "string"},
                    "data": {"type": "object"}
                },
                "required": ["table_name", "data"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    db = get_db()
    cursor = db.cursor()
    
    try:
        if name == "query_database":
            query = arguments["query"]
            # Security: Only allow SELECT
            if not query.strip().upper().startswith("SELECT"):
                return [TextContent(
                    type="text", 
                    text="Error: Only SELECT queries allowed"
                )]
            
            cursor.execute(query)
            results = cursor.fetchall()
            columns = [desc[0] for desc in cursor.description]
            
            formatted = [dict(zip(columns, row)) for row in results]
            return [TextContent(type="text", text=str(formatted))]
        
        elif name == "get_table_schema":
            table = arguments["table_name"]
            cursor.execute(f"PRAGMA table_info({table})")
            schema = cursor.fetchall()
            return [TextContent(type="text", text=str(schema))]
        
        elif name == "insert_record":
            table = arguments["table_name"]
            data = arguments["data"]
            columns = ", ".join(data.keys())
            placeholders = ", ".join(["?" for _ in data])
            values = list(data.values())
            
            cursor.execute(
                f"INSERT INTO {table} ({columns}) VALUES ({placeholders})",
                values
            )
            db.commit()
            return [TextContent(type="text", text="Record inserted")]
    
    finally:
        db.close()

# Resources: List tables
@server.list_resources()
async def list_resources():
    db = get_db()
    cursor = db.cursor()
    cursor.execute("SELECT name FROM sqlite_master WHERE type='table'")
    tables = cursor.fetchall()
    db.close()
    
    return [
        Resource(
            uri=f"db://tables/{table[0]}",
            name=f"Table: {table[0]}",
            description=f"Database table {table[0]}",
            mimeType="application/json"
        )
        for table in tables
    ]
```

### API Wrapper MCP Server

```python
import httpx
from mcp.server import Server
from mcp.types import Tool, TextContent

server = Server("api-wrapper-server")

API_BASE = "https://api.example.com"
API_KEY = "your-api-key"

async def api_request(method: str, endpoint: str, data: dict = None):
    async with httpx.AsyncClient() as client:
        headers = {"Authorization": f"Bearer {API_KEY}"}
        
        if method == "GET":
            response = await client.get(
                f"{API_BASE}{endpoint}",
                headers=headers,
                params=data
            )
        elif method == "POST":
            response = await client.post(
                f"{API_BASE}{endpoint}",
                headers=headers,
                json=data
            )
        
        return response.json()

@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="list_items",
            description="List all items from the API",
            inputSchema={
                "type": "object",
                "properties": {
                    "limit": {"type": "integer", "default": 10},
                    "offset": {"type": "integer", "default": 0}
                }
            }
        ),
        Tool(
            name="get_item",
            description="Get a specific item by ID",
            inputSchema={
                "type": "object",
                "properties": {
                    "item_id": {"type": "string"}
                },
                "required": ["item_id"]
            }
        ),
        Tool(
            name="create_item",
            description="Create a new item",
            inputSchema={
                "type": "object",
                "properties": {
                    "name": {"type": "string"},
                    "description": {"type": "string"}
                },
                "required": ["name"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "list_items":
        result = await api_request("GET", "/items", arguments)
        return [TextContent(type="text", text=str(result))]
    
    elif name == "get_item":
        item_id = arguments["item_id"]
        result = await api_request("GET", f"/items/{item_id}")
        return [TextContent(type="text", text=str(result))]
    
    elif name == "create_item":
        result = await api_request("POST", "/items", arguments)
        return [TextContent(type="text", text=str(result))]
```

### File System MCP Server

```python
import os
from pathlib import Path
from mcp.server import Server
from mcp.types import Tool, TextContent, Resource

server = Server("filesystem-server")

# Restrict to a safe directory
SAFE_ROOT = Path("/home/user/workspace")

def safe_path(relative_path: str) -> Path:
    """Ensure path is within safe root"""
    full_path = (SAFE_ROOT / relative_path).resolve()
    if not str(full_path).startswith(str(SAFE_ROOT)):
        raise ValueError("Path outside allowed directory")
    return full_path

@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="read_file",
            description="Read contents of a file",
            inputSchema={
                "type": "object",
                "properties": {
                    "path": {"type": "string"}
                },
                "required": ["path"]
            }
        ),
        Tool(
            name="write_file",
            description="Write content to a file",
            inputSchema={
                "type": "object",
                "properties": {
                    "path": {"type": "string"},
                    "content": {"type": "string"}
                },
                "required": ["path", "content"]
            }
        ),
        Tool(
            name="list_directory",
            description="List files in a directory",
            inputSchema={
                "type": "object",
                "properties": {
                    "path": {"type": "string", "default": "."}
                }
            }
        ),
        Tool(
            name="search_files",
            description="Search for files matching a pattern",
            inputSchema={
                "type": "object",
                "properties": {
                    "pattern": {"type": "string"},
                    "path": {"type": "string", "default": "."}
                },
                "required": ["pattern"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    try:
        if name == "read_file":
            path = safe_path(arguments["path"])
            content = path.read_text()
            return [TextContent(type="text", text=content)]
        
        elif name == "write_file":
            path = safe_path(arguments["path"])
            path.parent.mkdir(parents=True, exist_ok=True)
            path.write_text(arguments["content"])
            return [TextContent(type="text", text=f"Written to {path}")]
        
        elif name == "list_directory":
            path = safe_path(arguments.get("path", "."))
            items = list(path.iterdir())
            formatted = [
                f"{'📁' if i.is_dir() else '📄'} {i.name}"
                for i in items
            ]
            return [TextContent(type="text", text="\n".join(formatted))]
        
        elif name == "search_files":
            path = safe_path(arguments.get("path", "."))
            pattern = arguments["pattern"]
            matches = list(path.rglob(pattern))
            return [TextContent(
                type="text",
                text="\n".join(str(m) for m in matches[:50])
            )]
    
    except Exception as e:
        return [TextContent(type="text", text=f"Error: {e}")]
```

### Authentication Pattern

```python
import os
import jwt
from functools import wraps

# Get auth from environment
AUTH_SECRET = os.environ.get("MCP_AUTH_SECRET")

def require_auth(func):
    """Decorator to require authentication"""
    @wraps(func)
    async def wrapper(*args, **kwargs):
        # In real implementation, get token from context
        token = kwargs.get("auth_token")
        
        if not token:
            raise PermissionError("Authentication required")
        
        try:
            payload = jwt.decode(token, AUTH_SECRET, algorithms=["HS256"])
            kwargs["user"] = payload
        except jwt.InvalidTokenError:
            raise PermissionError("Invalid authentication token")
        
        return await func(*args, **kwargs)
    
    return wrapper

@server.call_tool()
@require_auth
async def call_tool(name: str, arguments: dict, user: dict = None):
    # Now we have authenticated user info
    if name == "admin_action" and user.get("role") != "admin":
        return [TextContent(type="text", text="Admin access required")]
    
    # ... rest of implementation
```

## 📖 Reading Materials

1. [MCP Server Examples](https://github.com/modelcontextprotocol/servers)
2. [MCP SDK Reference](https://modelcontextprotocol.io/sdk)
3. [Building Production MCP Servers](https://modelcontextprotocol.io/docs/concepts/servers)

## ✅ Activity: Build a Production MCP Server

Create `day22-mcp-advanced/` with:

1. A database-connected MCP server
2. Proper error handling
3. Input validation
4. Resource listing

### Complete This Day

[![Complete Day 22](https://img.shields.io/badge/Complete--Day--22-28a745?logo=github)](../../issues/new?title=Complete+Day+22&labels=complete-day-22&body=✅+I%27ve+completed+Day+22%21)

---

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1-21 | Previous Days | Complete |
| 🎁 **Day 22** | Custom MCP Servers | ✅ Current |
| 🎁 **Day 23** | Agent Safety & Guardrails | 🔜 Next |
