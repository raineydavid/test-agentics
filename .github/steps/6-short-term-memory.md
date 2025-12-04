# 🎄 Day 6: Short-term Memory

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Days 1-5
- Understanding of agent architecture

## 📝 Overview

**Short-term memory** gives agents the ability to maintain context within a conversation or task. Without it, every interaction would start fresh!

### What is Short-term Memory?

Short-term memory (also called **working memory** or **conversation memory**) stores:
- Recent conversation history
- Current task context  
- Intermediate results
- Tool outputs

### The Context Window Problem

LLMs have limited context windows. Managing short-term memory means:
1. Deciding what to keep
2. Deciding what to summarize
3. Deciding what to discard

```
┌─────────────────────────────────────────────────────────┐
│                   Context Window (e.g., 128k tokens)     │
├─────────────────────────────────────────────────────────┤
│ System Prompt │ Memory │ Recent Messages │ Current Turn │
│    (fixed)    │(managed)│   (sliding)    │   (new)      │
└─────────────────────────────────────────────────────────┘
```

### Memory Management Strategies

#### 1. Buffer Memory (Keep Everything)
```python
class BufferMemory:
    def __init__(self, max_messages=100):
        self.messages = []
        self.max_messages = max_messages
    
    def add(self, message):
        self.messages.append(message)
        if len(self.messages) > self.max_messages:
            self.messages.pop(0)  # Remove oldest
    
    def get_context(self):
        return self.messages
```

#### 2. Summary Memory (Compress Old Context)
```python
class SummaryMemory:
    def __init__(self, llm, max_recent=10):
        self.llm = llm
        self.summary = ""
        self.recent_messages = []
        self.max_recent = max_recent
    
    def add(self, message):
        self.recent_messages.append(message)
        
        if len(self.recent_messages) > self.max_recent:
            # Summarize old messages
            old_messages = self.recent_messages[:-self.max_recent]
            self.summary = self.llm.summarize(
                self.summary + "\n" + str(old_messages)
            )
            self.recent_messages = self.recent_messages[-self.max_recent:]
    
    def get_context(self):
        return {
            "summary": self.summary,
            "recent": self.recent_messages
        }
```

#### 3. Token-Based Window
```python
class TokenWindowMemory:
    def __init__(self, max_tokens=4000):
        self.messages = []
        self.max_tokens = max_tokens
    
    def add(self, message):
        self.messages.append(message)
        self._trim_to_token_limit()
    
    def _trim_to_token_limit(self):
        while self._count_tokens() > self.max_tokens:
            self.messages.pop(0)
    
    def _count_tokens(self):
        # Use tiktoken or similar
        return sum(count_tokens(m) for m in self.messages)
```

#### 4. Entity Memory (Track Important Things)
```python
class EntityMemory:
    def __init__(self, llm):
        self.llm = llm
        self.entities = {}  # {entity_name: description}
        self.messages = []
    
    def add(self, message):
        self.messages.append(message)
        # Extract and update entities
        new_entities = self.llm.extract_entities(message)
        self.entities.update(new_entities)
    
    def get_context(self):
        return {
            "entities": self.entities,
            "recent_messages": self.messages[-10:]
        }
```

### Implementing in an Agent

```python
class AgentWithMemory:
    def __init__(self, llm, tools, memory_type="buffer"):
        self.llm = llm
        self.tools = tools
        
        if memory_type == "buffer":
            self.memory = BufferMemory()
        elif memory_type == "summary":
            self.memory = SummaryMemory(llm)
        elif memory_type == "entity":
            self.memory = EntityMemory(llm)
    
    def chat(self, user_message):
        # Add user message to memory
        self.memory.add({"role": "user", "content": user_message})
        
        # Get context from memory
        context = self.memory.get_context()
        
        # Generate response with context
        response = self.llm.generate(
            system_prompt=self.system_prompt,
            context=context,
            message=user_message
        )
        
        # Add assistant response to memory
        self.memory.add({"role": "assistant", "content": response})
        
        return response
```

## 📖 Reading Materials

1. [LangChain Memory Types](https://python.langchain.com/docs/modules/memory/)
2. [Building Conversational Memory](https://www.pinecone.io/learn/langchain-conversational-memory/)
3. [Memory in AI Agents](https://lilianweng.github.io/posts/2023-06-23-agent/#memory)

## ✅ Activity: Implement Memory System

Create `day6-memory.py` implementing:

1. At least 2 different memory types
2. A simple chat agent using your memory
3. Demonstrate memory retention over multiple turns

**Test scenario:**
```
User: My name is Alex and I'm a software engineer.
Agent: Nice to meet you, Alex!...

[10 messages later]

User: What's my profession?
Agent: You mentioned you're a software engineer, Alex.
```

### Complete This Day

[![Complete Day 6](https://img.shields.io/badge/Complete--Day--6-28a745?logo=github)](../../issues/new?title=Complete+Day+6&labels=complete-day-6&body=✅+I%27ve+completed+Day+6%21)

---

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1-5 | Reasoning & Tools | Complete |
| 🎁 **Day 6** | Short-term Memory | ✅ Current |
| 🎁 **Day 7** | Long-term Memory & Vector Stores | 🔜 Next |
