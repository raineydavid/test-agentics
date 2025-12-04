# 🎄 Day 7: Long-term Memory & Vector Stores

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Days 1-6
- Understanding of embeddings (helpful but not required)

## 📝 Overview

**Long-term memory** allows agents to remember information across sessions using **vector stores** for semantic search.

### Short-term vs Long-term Memory

| Aspect | Short-term | Long-term |
|--------|------------|-----------|
| **Duration** | Current session | Persists forever |
| **Storage** | In-memory | Database/Vector Store |
| **Retrieval** | Sequential | Semantic search |
| **Capacity** | Limited (context window) | Unlimited |

### How Vector Memory Works

```
┌─────────────────────────────────────────────────────────────┐
│                     VECTOR MEMORY FLOW                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Text → Embedding Model → Vector [0.1, 0.3, ...] → Store    │
│                                                              │
│  Query → Embedding → Similarity Search → Relevant Memories   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Setting Up Vector Memory

#### Using ChromaDB (Local)
```python
import chromadb
from chromadb.utils import embedding_functions

# Initialize ChromaDB
client = chromadb.Client()
embedding_fn = embedding_functions.SentenceTransformerEmbeddingFunction(
    model_name="all-MiniLM-L6-v2"
)

# Create a collection
memory = client.create_collection(
    name="agent_memory",
    embedding_function=embedding_fn
)

# Store a memory
memory.add(
    documents=["User prefers Python over JavaScript"],
    metadatas=[{"type": "preference", "date": "2024-01-15"}],
    ids=["pref_001"]
)

# Retrieve relevant memories
results = memory.query(
    query_texts=["What programming language does the user like?"],
    n_results=5
)
```

#### Using Pinecone (Cloud)
```python
from pinecone import Pinecone
import openai

# Initialize
pc = Pinecone(api_key="your-api-key")
index = pc.Index("agent-memory")

# Create embedding
def embed(text):
    response = openai.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return response.data[0].embedding

# Store memory
index.upsert(vectors=[
    {
        "id": "mem_001",
        "values": embed("User works at Google as a ML engineer"),
        "metadata": {"type": "fact", "topic": "career"}
    }
])

# Query memories
results = index.query(
    vector=embed("Where does the user work?"),
    top_k=5,
    include_metadata=True
)
```

### Building a Long-term Memory Agent

```python
class LongTermMemoryAgent:
    def __init__(self, llm, vector_store):
        self.llm = llm
        self.vector_store = vector_store
    
    def remember(self, text: str, metadata: dict = None):
        """Store information in long-term memory"""
        self.vector_store.add(
            documents=[text],
            metadatas=[metadata or {}],
            ids=[f"mem_{uuid.uuid4()}"]
        )
    
    def recall(self, query: str, n: int = 5) -> list:
        """Retrieve relevant memories"""
        results = self.vector_store.query(
            query_texts=[query],
            n_results=n
        )
        return results["documents"][0]
    
    def chat(self, message: str) -> str:
        # Recall relevant memories
        memories = self.recall(message)
        
        # Build context with memories
        context = "Relevant memories:\n" + "\n".join(memories)
        
        # Generate response
        response = self.llm.generate(
            system_prompt=f"{self.system_prompt}\n\n{context}",
            message=message
        )
        
        # Optionally store new information
        if self._should_remember(message, response):
            self.remember(f"User said: {message}\nAssistant: {response}")
        
        return response
    
    def _should_remember(self, message, response) -> bool:
        """Decide if this interaction should be stored"""
        # Could use LLM to decide, or simple heuristics
        important_keywords = ["remember", "note", "important", "my name", "i am"]
        return any(kw in message.lower() for kw in important_keywords)
```

### Memory Types to Store

| Type | Example | Use Case |
|------|---------|----------|
| **Facts** | "User's birthday is March 15" | Personal info |
| **Preferences** | "Prefers detailed explanations" | Personalization |
| **Context** | "Working on a React project" | Task continuity |
| **Decisions** | "Chose AWS over GCP" | Decision tracking |

## 📖 Reading Materials

1. [ChromaDB Documentation](https://docs.trychroma.com/)
2. [Pinecone Documentation](https://docs.pinecone.io/)
3. [LangChain Vector Stores](https://python.langchain.com/docs/modules/data_connection/vectorstores/)

## ✅ Activity: Build Long-term Memory

Create `day7-long-term-memory.py` implementing:

1. Set up a vector store (ChromaDB recommended)
2. Implement remember() and recall() functions
3. Create an agent that uses long-term memory
4. Demonstrate memory persistence across "sessions"

### Complete This Day

[![Complete Day 7](https://img.shields.io/badge/Complete--Day--7-28a745?logo=github)](../../issues/new?title=Complete+Day+7&labels=complete-day-7&body=✅+I%27ve+completed+Day+7%21)

---

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1-6 | Foundations & Short-term Memory | Complete |
| 🎁 **Day 7** | Long-term Memory & Vector Stores | ✅ Current |
| 🎁 **Day 8** | Episodic Memory | 🔜 Next |
