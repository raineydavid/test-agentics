# 🎄 Day 18: LangGraph Deep Dive

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Days 1-17
- LangChain familiarity
- `pip install langgraph`

## 📝 Overview

**LangGraph** extends LangChain with graph-based orchestration for building complex, stateful multi-agent systems!

### Why LangGraph?

- **Cycles**: Handle iterative refinement
- **State**: Persistent state across nodes
- **Branching**: Conditional paths
- **Human-in-loop**: Pause for human input
- **Streaming**: Real-time outputs

### LangGraph Concepts

```
┌─────────────────────────────────────────────────────────────┐
│                    LANGGRAPH CONCEPTS                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  State: Shared data structure passed between nodes          │
│  Nodes: Functions that process and update state             │
│  Edges: Connections between nodes (can be conditional)      │
│  Graph: The complete workflow definition                     │
│                                                              │
│  [Start] → [Node A] → [Condition] → [Node B] → [End]        │
│                            ↓                                 │
│                        [Node C]                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Basic LangGraph Agent

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated, Sequence
from langchain_core.messages import BaseMessage, HumanMessage, AIMessage
import operator

# Define state
class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], operator.add]
    next_step: str

# Define nodes
def agent_node(state: AgentState) -> AgentState:
    """The main agent reasoning node"""
    messages = state["messages"]
    
    # Call LLM with messages
    response = llm.invoke(messages)
    
    # Check if we need tools
    if response.tool_calls:
        return {"messages": [response], "next_step": "tools"}
    else:
        return {"messages": [response], "next_step": "end"}

def tool_node(state: AgentState) -> AgentState:
    """Execute tool calls"""
    last_message = state["messages"][-1]
    results = []
    
    for tool_call in last_message.tool_calls:
        tool = tools_by_name[tool_call["name"]]
        result = tool.invoke(tool_call["args"])
        results.append(ToolMessage(content=result, tool_call_id=tool_call["id"]))
    
    return {"messages": results, "next_step": "agent"}

# Define routing
def should_continue(state: AgentState) -> str:
    return state["next_step"]

# Build graph
workflow = StateGraph(AgentState)

workflow.add_node("agent", agent_node)
workflow.add_node("tools", tool_node)

workflow.set_entry_point("agent")

workflow.add_conditional_edges(
    "agent",
    should_continue,
    {
        "tools": "tools",
        "end": END
    }
)

workflow.add_edge("tools", "agent")

# Compile
app = workflow.compile()

# Run
result = app.invoke({
    "messages": [HumanMessage(content="What is 25 * 47?")],
    "next_step": ""
})
```

### Multi-Agent Graph

```python
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode

class MultiAgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], operator.add]
    current_agent: str
    task: str
    results: dict

# Define agent nodes
def researcher_node(state: MultiAgentState) -> MultiAgentState:
    """Research agent gathers information"""
    response = researcher_llm.invoke(state["messages"])
    return {
        "messages": [response],
        "results": {**state.get("results", {}), "research": response.content},
        "current_agent": "writer"
    }

def writer_node(state: MultiAgentState) -> MultiAgentState:
    """Writer agent creates content"""
    context = state["results"].get("research", "")
    prompt = f"Based on this research: {context}\n\nWrite about: {state['task']}"
    response = writer_llm.invoke([HumanMessage(content=prompt)])
    return {
        "messages": [response],
        "results": {**state["results"], "draft": response.content},
        "current_agent": "critic"
    }

def critic_node(state: MultiAgentState) -> MultiAgentState:
    """Critic agent reviews and provides feedback"""
    draft = state["results"].get("draft", "")
    prompt = f"Review this draft and provide feedback:\n\n{draft}"
    response = critic_llm.invoke([HumanMessage(content=prompt)])
    
    # Decide if revision needed
    if "APPROVED" in response.content:
        return {
            "messages": [response],
            "results": {**state["results"], "feedback": response.content},
            "current_agent": "end"
        }
    else:
        return {
            "messages": [response],
            "results": {**state["results"], "feedback": response.content},
            "current_agent": "writer"  # Send back for revision
        }

def route_agent(state: MultiAgentState) -> str:
    return state["current_agent"]

# Build multi-agent graph
workflow = StateGraph(MultiAgentState)

workflow.add_node("researcher", researcher_node)
workflow.add_node("writer", writer_node)
workflow.add_node("critic", critic_node)

workflow.set_entry_point("researcher")

workflow.add_edge("researcher", "writer")
workflow.add_conditional_edges(
    "writer",
    lambda x: "critic",
    {"critic": "critic"}
)
workflow.add_conditional_edges(
    "critic",
    route_agent,
    {
        "writer": "writer",
        "end": END
    }
)

app = workflow.compile()
```

### Human-in-the-Loop

```python
from langgraph.checkpoint.sqlite import SqliteSaver

# Add checkpointing for persistence
memory = SqliteSaver.from_conn_string(":memory:")

app = workflow.compile(
    checkpointer=memory,
    interrupt_before=["critic"]  # Pause before critic reviews
)

# Start the workflow
config = {"configurable": {"thread_id": "1"}}
result = app.invoke(
    {"messages": [HumanMessage(content="Write about AI")], "task": "AI"},
    config
)

# Later, resume with human input
app.update_state(
    config,
    {"messages": [HumanMessage(content="Focus more on agents")]}
)
result = app.invoke(None, config)  # Continue from checkpoint
```

### Streaming

```python
# Stream events
for event in app.stream(
    {"messages": [HumanMessage(content="Research and write about AI")]},
    stream_mode="values"
):
    print(f"Current state: {event}")

# Stream specific outputs
async for chunk in app.astream(
    {"messages": [HumanMessage(content="Hello")]},
    stream_mode="messages"
):
    print(chunk)
```

## 📖 Reading Materials

1. [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
2. [LangGraph Tutorials](https://langchain-ai.github.io/langgraph/tutorials/)
3. [Multi-Agent with LangGraph](https://blog.langchain.dev/langgraph-multi-agent/)

## ✅ Activity: Build with LangGraph

Create `day18-langgraph.py` implementing:

1. A stateful agent graph
2. Conditional routing between nodes
3. At least 3 specialized agent nodes

### Complete This Day

[![Complete Day 18](https://img.shields.io/badge/Complete--Day--18-28a745?logo=github)](../../issues/new?title=Complete+Day+18&labels=complete-day-18&body=✅+I%27ve+completed+Day+18%21)

---

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1-17 | Previous Days | Complete |
| 🎁 **Day 18** | LangGraph Deep Dive | ✅ Current |
| 🎁 **Day 19** | CrewAI Framework | 🔜 Next |
