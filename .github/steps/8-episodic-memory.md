# 🎄 Day 8: Episodic Memory

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Days 1-7
- Understanding of vector stores

## 📝 Overview

**Episodic memory** stores specific experiences and events - the "what, when, where" of agent interactions. It's like autobiographical memory for AI agents!

### Types of Agent Memory

```
┌─────────────────────────────────────────────────────────────┐
│                    AGENT MEMORY TYPES                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Semantic Memory: Facts & Knowledge                          │
│    "Paris is the capital of France"                          │
│                                                              │
│  Episodic Memory: Specific Experiences                       │
│    "On Tuesday at 2pm, user asked me to debug their code"    │
│                                                              │
│  Procedural Memory: How to do things                         │
│    "Steps to deploy to AWS"                                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Why Episodic Memory Matters

Episodic memory enables:
- Learning from past successes and failures
- Contextual recall ("Last time you asked about X...")
- Building relationships over time
- Avoiding repeated mistakes

### Episodic Memory Structure

```python
@dataclass
class Episode:
    id: str
    timestamp: datetime
    task: str
    actions: List[dict]
    outcome: str  # success, failure, partial
    context: dict
    learnings: List[str]
    
    def to_dict(self):
        return {
            "id": self.id,
            "timestamp": self.timestamp.isoformat(),
            "task": self.task,
            "actions": self.actions,
            "outcome": self.outcome,
            "context": self.context,
            "learnings": self.learnings
        }
```

### Implementing Episodic Memory

```python
class EpisodicMemory:
    def __init__(self, vector_store, llm):
        self.vector_store = vector_store
        self.llm = llm
        self.current_episode = None
    
    def start_episode(self, task: str, context: dict = None):
        """Begin recording a new episode"""
        self.current_episode = Episode(
            id=str(uuid.uuid4()),
            timestamp=datetime.now(),
            task=task,
            actions=[],
            outcome=None,
            context=context or {},
            learnings=[]
        )
    
    def record_action(self, action: str, result: str):
        """Record an action within the current episode"""
        if self.current_episode:
            self.current_episode.actions.append({
                "action": action,
                "result": result,
                "timestamp": datetime.now().isoformat()
            })
    
    def end_episode(self, outcome: str):
        """Complete the episode and extract learnings"""
        if not self.current_episode:
            return
        
        self.current_episode.outcome = outcome
        
        # Use LLM to extract learnings
        self.current_episode.learnings = self._extract_learnings()
        
        # Store in vector database
        self._store_episode()
        
        episode = self.current_episode
        self.current_episode = None
        return episode
    
    def _extract_learnings(self) -> List[str]:
        """Use LLM to extract key learnings from the episode"""
        prompt = f"""
        Analyze this task episode and extract key learnings:
        
        Task: {self.current_episode.task}
        Actions taken: {self.current_episode.actions}
        Outcome: {self.current_episode.outcome}
        
        What are 2-3 key learnings from this experience?
        """
        response = self.llm.generate(prompt)
        return self._parse_learnings(response)
    
    def _store_episode(self):
        """Store episode in vector database"""
        # Create searchable text representation
        episode_text = f"""
        Task: {self.current_episode.task}
        Outcome: {self.current_episode.outcome}
        Learnings: {', '.join(self.current_episode.learnings)}
        """
        
        self.vector_store.add(
            documents=[episode_text],
            metadatas=[self.current_episode.to_dict()],
            ids=[self.current_episode.id]
        )
    
    def recall_similar_episodes(self, task: str, n: int = 3) -> List[Episode]:
        """Find similar past episodes"""
        results = self.vector_store.query(
            query_texts=[task],
            n_results=n
        )
        return [self._reconstruct_episode(m) for m in results["metadatas"][0]]
```

### Using Episodic Memory in Agents

```python
class EpisodicAgent:
    def __init__(self, llm, tools, episodic_memory):
        self.llm = llm
        self.tools = tools
        self.memory = episodic_memory
    
    def run(self, task: str) -> str:
        # Check for similar past experiences
        similar_episodes = self.memory.recall_similar_episodes(task)
        
        # Build context from past experiences
        experience_context = self._build_experience_context(similar_episodes)
        
        # Start recording new episode
        self.memory.start_episode(task)
        
        try:
            # Execute task with experience context
            result = self._execute_with_context(task, experience_context)
            self.memory.end_episode("success")
            return result
        except Exception as e:
            self.memory.end_episode("failure")
            raise
    
    def _build_experience_context(self, episodes: List[Episode]) -> str:
        if not episodes:
            return ""
        
        context = "Relevant past experiences:\n"
        for ep in episodes:
            context += f"- Task: {ep.task}\n"
            context += f"  Outcome: {ep.outcome}\n"
            context += f"  Learnings: {', '.join(ep.learnings)}\n"
        return context
```

## 📖 Reading Materials

1. [Cognitive Architectures for Language Agents](https://arxiv.org/abs/2309.02427)
2. [Generative Agents Paper](https://arxiv.org/abs/2304.03442)
3. [MemGPT: Memory Management](https://arxiv.org/abs/2310.08560)

## ✅ Activity: Build Episodic Memory

Create `day8-episodic-memory.py` implementing:

1. Episode data structure
2. EpisodicMemory class with start/record/end
3. Learning extraction using LLM
4. Demonstrate recalling past episodes for new tasks

### Complete This Day

[![Complete Day 8](https://img.shields.io/badge/Complete--Day--8-28a745?logo=github)](../../issues/new?title=Complete+Day+8&labels=complete-day-8&body=✅+I%27ve+completed+Day+8%21)

---

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1-7 | Memory Foundations | Complete |
| 🎁 **Day 8** | Episodic Memory | ✅ Current |
| 🎁 **Day 9** | Planning Strategies | 🔜 Next |
