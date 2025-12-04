# 🎄 Day 14: Multi-Agent Basics

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Days 1-13
- Understanding of single agents

## 📝 Overview

**Multi-Agent Systems** use multiple specialized agents working together to solve complex problems. Think of it as a team of AI specialists!

### Why Multi-Agent?

| Single Agent | Multi-Agent |
|-------------|-------------|
| One perspective | Multiple perspectives |
| Limited expertise | Specialized experts |
| Sequential thinking | Parallel processing |
| Single point of failure | Redundancy |

### Multi-Agent Architectures

```
┌─────────────────────────────────────────────────────────────┐
│              MULTI-AGENT ARCHITECTURES                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. HIERARCHICAL          2. FLAT/PEER                       │
│                                                              │
│     [Manager]                [Agent A] ←→ [Agent B]          │
│        ↓                          ↘   ↗                      │
│  [Agent A] [Agent B]            [Agent C]                    │
│                                                              │
│  3. SEQUENTIAL            4. DEBATE                          │
│                                                              │
│  [A] → [B] → [C]          [Pro] ←→ [Con]                    │
│                                 ↓                            │
│                             [Judge]                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Defining Agent Roles

```python
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class AgentRole:
    name: str
    description: str
    skills: List[str]
    tools: List[str]
    system_prompt: str

# Define specialized agents
researcher = AgentRole(
    name="Researcher",
    description="Expert at finding and analyzing information",
    skills=["web search", "data analysis", "summarization"],
    tools=["web_search", "read_document"],
    system_prompt="""You are a research specialist. Your job is to:
    - Find accurate information from reliable sources
    - Analyze and summarize findings
    - Cite your sources"""
)

writer = AgentRole(
    name="Writer",
    description="Expert at creating clear, engaging content",
    skills=["writing", "editing", "storytelling"],
    tools=["text_editor"],
    system_prompt="""You are a writing specialist. Your job is to:
    - Create clear, engaging content
    - Adapt tone for the audience
    - Structure information logically"""
)

critic = AgentRole(
    name="Critic",
    description="Expert at evaluating and improving work",
    skills=["analysis", "feedback", "quality assurance"],
    tools=["review_checklist"],
    system_prompt="""You are a quality critic. Your job is to:
    - Evaluate work against requirements
    - Provide constructive feedback
    - Identify areas for improvement"""
)
```

### Simple Multi-Agent System

```python
class MultiAgentSystem:
    def __init__(self, llm):
        self.llm = llm
        self.agents = {}
    
    def add_agent(self, role: AgentRole):
        self.agents[role.name] = Agent(self.llm, role)
    
    def sequential_pipeline(self, task: str, agent_sequence: List[str]) -> str:
        """Run agents in sequence, each building on previous work"""
        context = {"original_task": task}
        result = task
        
        for agent_name in agent_sequence:
            agent = self.agents[agent_name]
            result = agent.run(result, context)
            context[agent_name] = result
        
        return result
    
    def parallel_execution(self, task: str, agent_names: List[str]) -> dict:
        """Run agents in parallel on the same task"""
        results = {}
        
        # In production, use asyncio or threads
        for agent_name in agent_names:
            agent = self.agents[agent_name]
            results[agent_name] = agent.run(task)
        
        return results
    
    def debate(self, topic: str, rounds: int = 3) -> str:
        """Two agents debate, third judges"""
        pro_agent = self.agents["Pro"]
        con_agent = self.agents["Con"]
        judge = self.agents["Judge"]
        
        debate_log = []
        
        for round_num in range(rounds):
            # Pro argues
            pro_arg = pro_agent.run(f"Argue FOR: {topic}\nPrevious debate: {debate_log}")
            debate_log.append(f"Pro: {pro_arg}")
            
            # Con responds
            con_arg = con_agent.run(f"Argue AGAINST: {topic}\nPrevious debate: {debate_log}")
            debate_log.append(f"Con: {con_arg}")
        
        # Judge decides
        verdict = judge.run(f"Judge this debate and provide final verdict:\n{debate_log}")
        return verdict
```

### Agent Class

```python
class Agent:
    def __init__(self, llm, role: AgentRole):
        self.llm = llm
        self.role = role
        self.memory = []
    
    def run(self, task: str, context: dict = None) -> str:
        messages = [
            {"role": "system", "content": self.role.system_prompt},
            {"role": "user", "content": self._build_prompt(task, context)}
        ]
        
        response = self.llm.generate(messages)
        self.memory.append({"task": task, "response": response})
        
        return response
    
    def _build_prompt(self, task: str, context: dict) -> str:
        prompt = f"Task: {task}\n"
        if context:
            prompt += f"\nContext from other agents:\n"
            for key, value in context.items():
                if key != "original_task":
                    prompt += f"- {key}: {value[:500]}...\n"
        return prompt
```

### Example: Content Creation Pipeline

```python
# Setup
mas = MultiAgentSystem(llm)
mas.add_agent(researcher)
mas.add_agent(writer)
mas.add_agent(critic)

# Run pipeline
topic = "The impact of AI agents on software development"

result = mas.sequential_pipeline(
    task=f"Create a blog post about: {topic}",
    agent_sequence=["Researcher", "Writer", "Critic"]
)
```

## 📖 Reading Materials

1. [Multi-Agent Collaboration](https://www.anthropic.com/research/building-effective-agents)
2. [AutoGen Framework](https://microsoft.github.io/autogen/)
3. [CrewAI Documentation](https://docs.crewai.com/)

## ✅ Activity: Build Multi-Agent System

Create `day14-multi-agent.py` implementing:

1. At least 3 specialized agent roles
2. Sequential pipeline execution
3. A debate or discussion mechanism

### Complete This Day

[![Complete Day 14](https://img.shields.io/badge/Complete--Day--14-28a745?logo=github)](../../issues/new?title=Complete+Day+14&labels=complete-day-14&body=✅+I%27ve+completed+Day+14%21)

---

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1-13 | Previous Days | Complete |
| 🎁 **Day 14** | Multi-Agent Basics | ✅ Current |
| 🎁 **Day 15** | Agent Communication Patterns | 🔜 Next |
