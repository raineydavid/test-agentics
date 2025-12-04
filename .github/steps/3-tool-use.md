# 🎄 Day 3: Tool Use & Function Calling

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Days 1-2
- Python 3.9+ or Node.js 18+
- API key for Claude, OpenAI, or similar

## 📝 Overview

**Tool use** (also called **function calling**) is what transforms a chatbot into an agent. Today we learn how agents interact with external tools!

### What is Tool Use?

Tool use allows an LLM to:
1. Recognize when a tool is needed
2. Generate structured arguments for the tool
3. Receive and process tool results
4. Incorporate results into its response

### The Tool Use Flow

```
User Query → LLM → Tool Decision → Tool Execution → LLM → Response
                         ↓
                   Tool Arguments
                   (structured JSON)
```

### Defining Tools

#### Python (with Anthropic)
```python
import anthropic

tools = [
    {
        "name": "get_weather",
        "description": "Get the current weather for a location",
        "input_schema": {
            "type": "object",
            "properties": {
                "location": {
                    "type": "string",
                    "description": "City and state, e.g., San Francisco, CA"
                },
                "unit": {
                    "type": "string",
                    "enum": ["celsius", "fahrenheit"],
                    "description": "Temperature unit"
                }
            },
            "required": ["location"]
        }
    }
]

client = anthropic.Anthropic()
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "What's the weather in Tokyo?"}]
)
```

#### JavaScript (with Anthropic)
```javascript
import Anthropic from '@anthropic-ai/sdk';

const tools = [
    {
        name: "get_weather",
        description: "Get the current weather for a location",
        input_schema: {
            type: "object",
            properties: {
                location: {
                    type: "string",
                    description: "City and state, e.g., San Francisco, CA"
                }
            },
            required: ["location"]
        }
    }
];

const client = new Anthropic();
const response = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 1024,
    tools: tools,
    messages: [{ role: "user", content: "What's the weather in Tokyo?" }]
});
```

### Handling Tool Results

```python
# After receiving a tool_use response
if response.stop_reason == "tool_use":
    tool_use = response.content[0]  # Get the tool call
    
    # Execute the actual tool
    result = execute_tool(tool_use.name, tool_use.input)
    
    # Send result back to Claude
    follow_up = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        tools=tools,
        messages=[
            {"role": "user", "content": "What's the weather in Tokyo?"},
            {"role": "assistant", "content": response.content},
            {"role": "user", "content": [
                {
                    "type": "tool_result",
                    "tool_use_id": tool_use.id,
                    "content": str(result)
                }
            ]}
        ]
    )
```

### Common Tool Categories

| Category | Examples |
|----------|----------|
| **Search** | Web search, database queries |
| **Code** | Execute code, analyze files |
| **Communication** | Send emails, Slack messages |
| **Data** | API calls, file operations |
| **Math** | Calculations, statistics |

## 📖 Reading Materials

1. [Anthropic Tool Use Guide](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)
2. [OpenAI Function Calling](https://platform.openai.com/docs/guides/function-calling)
3. [LangChain Tools](https://python.langchain.com/docs/modules/tools/)

## ✅ Activity: Build Your First Tool

Create a file called `day3-tool.py` (or `.js`) implementing:

1. Define at least 2 tools with proper schemas
2. Handle tool calls from the LLM
3. Execute the tools and return results

**Starter template:**
```python
# day3-tool.py
import anthropic

def calculator(expression: str) -> float:
    """Safely evaluate a math expression"""
    return eval(expression)  # Note: Use safer eval in production!

def get_current_time(timezone: str = "UTC") -> str:
    """Get current time in specified timezone"""
    from datetime import datetime
    return datetime.now().isoformat()

# Define your tools and implement the agent loop!
```

### Complete This Day

[![Complete Day 3](https://img.shields.io/badge/Complete--Day--3-28a745?logo=github)](../../issues/new?title=Complete+Day+3&labels=complete-day-3&body=✅+I%27ve+completed+Day+3%21)

---

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1-2 | Fundamentals | Complete |
| 🎁 **Day 3** | Tool Use & Function Calling | ✅ Current |
| 🎁 **Day 4** | ReAct Pattern | 🔜 Next |
