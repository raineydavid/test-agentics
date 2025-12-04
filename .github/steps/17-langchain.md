# 🎄 Day 17: LangChain Agents

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Days 1-16
- `pip install langchain langchain-anthropic`

## 📝 Overview

**LangChain** is one of the most popular frameworks for building AI agents. Today we dive into building agents with LangChain!

### LangChain Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    LANGCHAIN STACK                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  LangChain Expression Language (LCEL)                       │
│       ↓                                                      │
│  Chains & Agents                                             │
│       ↓                                                      │
│  Tools & Retrievers                                          │
│       ↓                                                      │
│  Models (LLMs, Chat, Embeddings)                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Basic LangChain Agent

```python
from langchain_anthropic import ChatAnthropic
from langchain.agents import AgentExecutor, create_tool_calling_agent
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.tools import tool

# Initialize LLM
llm = ChatAnthropic(model="claude-sonnet-4-20250514")

# Define tools
@tool
def calculator(expression: str) -> float:
    """Evaluate a math expression. Use this for any calculations."""
    return eval(expression)

@tool  
def search(query: str) -> str:
    """Search the web for information."""
    # Implement actual search
    return f"Search results for: {query}"

tools = [calculator, search]

# Create prompt
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant. Use tools when needed."),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}")
])

# Create agent
agent = create_tool_calling_agent(llm, tools, prompt)

# Create executor
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,
    max_iterations=10
)

# Run
result = agent_executor.invoke({
    "input": "What is 25 * 47 + 123?"
})
print(result["output"])
```

### Custom Tools

```python
from langchain_core.tools import BaseTool
from pydantic import BaseModel, Field
from typing import Type

class SearchInput(BaseModel):
    query: str = Field(description="The search query")
    num_results: int = Field(default=5, description="Number of results")

class CustomSearchTool(BaseTool):
    name: str = "web_search"
    description: str = "Search the web for current information"
    args_schema: Type[BaseModel] = SearchInput
    
    def _run(self, query: str, num_results: int = 5) -> str:
        # Implement search logic
        results = perform_search(query, num_results)
        return "\n".join(results)
    
    async def _arun(self, query: str, num_results: int = 5) -> str:
        # Async version
        return await async_search(query, num_results)
```

### ReAct Agent in LangChain

```python
from langchain.agents import create_react_agent
from langchain import hub

# Pull a ReAct prompt from the hub
react_prompt = hub.pull("hwchase17/react")

# Create ReAct agent
react_agent = create_react_agent(
    llm=llm,
    tools=tools,
    prompt=react_prompt
)

# Execute
executor = AgentExecutor(
    agent=react_agent,
    tools=tools,
    verbose=True,
    handle_parsing_errors=True
)

result = executor.invoke({
    "input": "What's the weather in Tokyo and convert 30°C to Fahrenheit"
})
```

### Agent with Memory

```python
from langchain.memory import ConversationBufferMemory

# Create memory
memory = ConversationBufferMemory(
    memory_key="chat_history",
    return_messages=True
)

# Prompt with memory
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
    ("placeholder", "{chat_history}"),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}")
])

# Create agent with memory
agent = create_tool_calling_agent(llm, tools, prompt)

executor = AgentExecutor(
    agent=agent,
    tools=tools,
    memory=memory,
    verbose=True
)

# First interaction
executor.invoke({"input": "My name is Alice"})

# Memory is preserved
executor.invoke({"input": "What's my name?"})  # "Your name is Alice"
```

### Structured Output Agent

```python
from langchain_core.pydantic_v1 import BaseModel, Field

class ResearchResult(BaseModel):
    """Structured research output"""
    topic: str = Field(description="The research topic")
    summary: str = Field(description="Summary of findings")
    sources: list[str] = Field(description="List of sources")
    confidence: float = Field(description="Confidence score 0-1")

# Create structured output agent
structured_agent = create_tool_calling_agent(
    llm.with_structured_output(ResearchResult),
    tools,
    prompt
)
```

### Agent Types Comparison

| Type | Use Case | Pros | Cons |
|------|----------|------|------|
| **Tool Calling** | General purpose | Most reliable | Requires tool-calling LLM |
| **ReAct** | Reasoning tasks | Interpretable | More verbose |
| **OpenAI Functions** | OpenAI models | Native support | OpenAI only |
| **Structured Chat** | Complex tools | Type safety | More setup |

### Best Practices

```python
# 1. Handle errors gracefully
executor = AgentExecutor(
    agent=agent,
    tools=tools,
    handle_parsing_errors=True,
    max_iterations=15,
    early_stopping_method="generate"  # or "force"
)

# 2. Add callbacks for monitoring
from langchain.callbacks import StdOutCallbackHandler

executor.invoke(
    {"input": "Task"},
    config={"callbacks": [StdOutCallbackHandler()]}
)

# 3. Use async for better performance
result = await executor.ainvoke({"input": "Task"})
```

## 📖 Reading Materials

1. [LangChain Agents Docs](https://python.langchain.com/docs/modules/agents/)
2. [LangChain Tool Calling](https://python.langchain.com/docs/modules/agents/agent_types/tool_calling/)
3. [LangChain Hub](https://smith.langchain.com/hub)

## ✅ Activity: Build with LangChain

Create `day17-langchain.py` implementing:

1. A tool-calling agent with 3+ custom tools
2. Memory for multi-turn conversations
3. Error handling and callbacks

### Complete This Day

[![Complete Day 17](https://img.shields.io/badge/Complete--Day--17-28a745?logo=github)](../../issues/new?title=Complete+Day+17&labels=complete-day-17&body=✅+I%27ve+completed+Day+17%21)

---

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1-16 | Previous Days | Complete |
| 🎁 **Day 17** | LangChain Agents | ✅ Current |
| 🎁 **Day 18** | LangGraph Deep Dive | 🔜 Next |
