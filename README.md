

# 🎄 Day 2: Agent Architecture & Components

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Day 1
- Understanding of what AI Agents are

## 📝 Overview

Today we'll dive deep into the **architecture** and core **components** that make up an AI Agent system.

### Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                        AI AGENT                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │    Brain     │  │   Memory     │  │    Tools     │       │
│  │    (LLM)     │  │   System     │  │              │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │   Planner    │  │  Executor    │  │   Observer   │       │
│  │              │  │              │  │              │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Component Details

#### 1. Brain (LLM)
The reasoning engine that processes information and makes decisions.
```python
# The LLM is the core reasoning component
brain = ChatAnthropic(model="claude-sonnet-4-20250514")
```

#### 2. Memory System
Stores context, past interactions, and learned information.
- **Short-term**: Current conversation context
- **Long-term**: Persistent knowledge base
- **Episodic**: Specific past experiences

#### 3. Tools
External capabilities the agent can invoke.
```python
tools = [
    SearchTool(),
    CalculatorTool(),
    CodeExecutorTool(),
    FileSystemTool()
]
```

#### 4. Planner
Breaks down complex tasks into manageable steps.

#### 5. Executor
Carries out planned actions using available tools.

#### 6. Observer
Monitors the environment and action outcomes.

### Simple Agent Structure (Python)

```python
class SimpleAgent:
    def __init__(self, llm, tools, memory):
        self.llm = llm
        self.tools = tools
        self.memory = memory
    
    def run(self, task: str) -> str:
        # 1. Add task to memory
        self.memory.add({"role": "user", "content": task})
        
        # 2. Plan actions
        plan = self.llm.plan(task, self.tools)
        
        # 3. Execute plan
        for step in plan:
            result = self.execute_step(step)
            self.memory.add({"role": "assistant", "content": result})
        
        # 4. Generate final response
        return self.llm.respond(self.memory.get_context())
    
    def execute_step(self, step):
        tool = self.get_tool(step.tool_name)
        return tool.run(step.arguments)
```

## 📖 Reading Materials

1. [LangChain Agent Components](https://python.langchain.com/docs/modules/agents/)
2. [Agent Architecture Patterns](https://lilianweng.github.io/posts/2023-06-23-agent/)
3. [ReAct: Synergizing Reasoning and Acting](https://arxiv.org/abs/2210.03629)

## ✅ Activity: Design Your Agent Architecture

Create a file called `day2-architecture.md` with a diagram and description of an agent architecture for your Day 1 concept.

Include:
1. A text-based architecture diagram
2. Description of each component
3. How components interact

### Complete This Day

[![Complete Day 2](https://img.shields.io/badge/Complete--Day--2-28a745?logo=github)](../../issues/new?title=Complete+Day+2&labels=complete-day-2&body=✅+I%27ve+completed+Day+2%21)

---

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1 | What are AI Agents? | Complete |
| 🎁 **Day 2** | Agent Architecture & Components | ✅ Current |
| 🎁 **Day 3** | Tool Use & Function Calling | 🔜 Next |


