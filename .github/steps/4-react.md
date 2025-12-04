# 🎄 Day 4: The ReAct Pattern

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Days 1-3
- Understanding of tool use

## 📝 Overview

**ReAct** (Reasoning + Acting) is a foundational agent pattern that interleaves thinking and action. Today we'll master this essential technique!

### What is ReAct?

ReAct combines:
- **Reasoning**: The agent thinks about what to do
- **Acting**: The agent takes an action
- **Observing**: The agent sees the result

### The ReAct Loop

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Question → Thought → Action → Observation → Thought...│
│                                                         │
│            ↻ (repeat until answer found)                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### ReAct Trace Example

```
Question: What is the capital of the country where the Eiffel Tower is located?

Thought 1: I need to find where the Eiffel Tower is located first.
Action 1: Search[Eiffel Tower location]
Observation 1: The Eiffel Tower is located in Paris, France.

Thought 2: The Eiffel Tower is in France. Now I need to find the capital of France.
Action 2: Search[capital of France]
Observation 2: The capital of France is Paris.

Thought 3: I found the answer. The capital of France is Paris.
Action 3: Finish[Paris]
```

### Implementing ReAct

```python
REACT_PROMPT = """
You are an AI assistant that follows the ReAct pattern.
For each step, provide:
- Thought: Your reasoning about what to do next
- Action: The action to take (Search, Calculate, Finish)
- Action Input: The input for the action

Available actions:
- Search[query]: Search for information
- Calculate[expression]: Perform calculations  
- Finish[answer]: Provide the final answer

Question: {question}

{agent_scratchpad}
"""

class ReActAgent:
    def __init__(self, llm, tools):
        self.llm = llm
        self.tools = tools
        self.max_iterations = 10
    
    def run(self, question: str) -> str:
        scratchpad = ""
        
        for i in range(self.max_iterations):
            # Get next thought/action from LLM
            prompt = REACT_PROMPT.format(
                question=question,
                agent_scratchpad=scratchpad
            )
            response = self.llm.generate(prompt)
            
            # Parse the response
            thought, action, action_input = self.parse_response(response)
            
            # Check if we're done
            if action == "Finish":
                return action_input
            
            # Execute action and get observation
            observation = self.execute_action(action, action_input)
            
            # Update scratchpad
            scratchpad += f"""
Thought {i+1}: {thought}
Action {i+1}: {action}[{action_input}]
Observation {i+1}: {observation}
"""
        
        return "Max iterations reached without answer"
    
    def parse_response(self, response: str):
        # Extract Thought, Action, Action Input from response
        # Implementation depends on LLM output format
        pass
    
    def execute_action(self, action: str, action_input: str):
        tool = self.tools.get(action)
        if tool:
            return tool.run(action_input)
        return f"Unknown action: {action}"
```

### Benefits of ReAct

| Benefit | Description |
|---------|-------------|
| **Interpretability** | We can see the agent's reasoning |
| **Debugging** | Easy to identify where things go wrong |
| **Grounding** | Actions provide real-world feedback |
| **Flexibility** | Can handle diverse tasks |

### ReAct vs. Pure Reasoning

```
Pure Reasoning (Chain-of-Thought):
  Think → Think → Think → Answer (no external input)

ReAct:
  Think → Act → Observe → Think → Act → Observe → Answer
  (grounded in real observations)
```

## 📖 Reading Materials

1. [ReAct Paper (Yao et al.)](https://arxiv.org/abs/2210.03629)
2. [LangChain ReAct Agent](https://python.langchain.com/docs/modules/agents/agent_types/react)
3. [Prompt Engineering Guide: ReAct](https://www.promptingguide.ai/techniques/react)

## ✅ Activity: Build a ReAct Agent

Create `day4-react-agent.py` implementing:

1. A ReAct agent with at least 2 tools
2. Clear thought/action/observation logging
3. Test it with a multi-step question

**Test questions:**
- "What is the population of the capital of Japan?"
- "What year did the founder of Microsoft graduate college?"

### Complete This Day

[![Complete Day 4](https://img.shields.io/badge/Complete--Day--4-28a745?logo=github)](../../issues/new?title=Complete+Day+4&labels=complete-day-4&body=✅+I%27ve+completed+Day+4%21)

---

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1-3 | Fundamentals & Tools | Complete |
| 🎁 **Day 4** | ReAct Pattern | ✅ Current |
| 🎁 **Day 5** | Chain of Thought Reasoning | 🔜 Next |
