# 🎄 Day 11: RAG-Enhanced Agents

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Days 1-10
- Understanding of vector stores (Day 7)

## 📝 Overview

**RAG (Retrieval-Augmented Generation)** combined with agents creates powerful systems that can reason over external knowledge bases!

### Why RAG + Agents?

```
Standard LLM: Limited to training data
RAG: Can access external knowledge
RAG Agent: Can reason about WHEN and WHAT to retrieve
```

### RAG Agent Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     RAG AGENT                                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  User Query → Agent Reasoning → Retrieval Decision          │
│                      ↓                                       │
│              Formulate Search Query                          │
│                      ↓                                       │
│              Vector Store Search                             │
│                      ↓                                       │
│              Evaluate Results                                │
│                      ↓                                       │
│         Need more info? → Yes → Refine query                │
│                      ↓ No                                    │
│              Generate Response                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Agentic RAG vs Simple RAG

| Aspect | Simple RAG | Agentic RAG |
|--------|-----------|-------------|
| **Query** | Use user query as-is | Agent reformulates query |
| **Retrieval** | Single retrieval | Multiple retrievals as needed |
| **Evaluation** | None | Agent evaluates relevance |
| **Reasoning** | Direct synthesis | Multi-step reasoning |

### Implementing Agentic RAG

```python
class RAGAgent:
    def __init__(self, llm, vector_store, tools=None):
        self.llm = llm
        self.vector_store = vector_store
        self.tools = tools or []
    
    def run(self, query: str) -> str:
        # Step 1: Decide if retrieval is needed
        needs_retrieval = self.should_retrieve(query)
        
        if not needs_retrieval:
            return self.llm.generate(query)
        
        # Step 2: Formulate search queries
        search_queries = self.formulate_queries(query)
        
        # Step 3: Retrieve relevant documents
        all_docs = []
        for sq in search_queries:
            docs = self.retrieve(sq)
            all_docs.extend(docs)
        
        # Step 4: Evaluate and filter
        relevant_docs = self.evaluate_relevance(query, all_docs)
        
        # Step 5: Check if we need more information
        if self.needs_more_info(query, relevant_docs):
            additional_queries = self.generate_followup_queries(query, relevant_docs)
            for aq in additional_queries:
                more_docs = self.retrieve(aq)
                relevant_docs.extend(self.evaluate_relevance(query, more_docs))
        
        # Step 6: Generate final response
        return self.generate_response(query, relevant_docs)
    
    def should_retrieve(self, query: str) -> bool:
        prompt = f"""
        Query: {query}
        
        Does this query require retrieving external information, 
        or can it be answered from general knowledge?
        
        Respond with: RETRIEVE or GENERAL
        """
        response = self.llm.generate(prompt)
        return "RETRIEVE" in response
    
    def formulate_queries(self, query: str) -> List[str]:
        prompt = f"""
        User question: {query}
        
        Generate 2-3 search queries to find relevant information.
        Each query should target different aspects of the question.
        
        Return as a JSON list: ["query1", "query2", ...]
        """
        response = self.llm.generate(prompt)
        return json.loads(response)
    
    def evaluate_relevance(self, query: str, docs: List[str]) -> List[str]:
        relevant = []
        for doc in docs:
            prompt = f"""
            Query: {query}
            Document: {doc[:500]}...
            
            Is this document relevant to answering the query?
            Respond: RELEVANT or NOT_RELEVANT
            """
            if "RELEVANT" in self.llm.generate(prompt):
                relevant.append(doc)
        return relevant
    
    def needs_more_info(self, query: str, docs: List[str]) -> bool:
        prompt = f"""
        Query: {query}
        Retrieved information: {self.summarize_docs(docs)}
        
        Do we have enough information to fully answer the query?
        Respond: SUFFICIENT or NEED_MORE
        """
        return "NEED_MORE" in self.llm.generate(prompt)
```

### Advanced: Self-RAG

Self-RAG adds critique and refinement:

```python
class SelfRAGAgent(RAGAgent):
    def generate_response(self, query: str, docs: List[str]) -> str:
        # Generate initial response
        response = super().generate_response(query, docs)
        
        # Self-critique
        critique = self.critique_response(query, docs, response)
        
        if critique.has_issues:
            # Identify what's missing
            gaps = self.identify_gaps(critique)
            
            # Retrieve more targeted information
            for gap in gaps:
                additional_docs = self.retrieve(gap)
                docs.extend(additional_docs)
            
            # Regenerate with complete information
            response = self.regenerate_with_critique(
                query, docs, response, critique
            )
        
        return response
    
    def critique_response(self, query, docs, response):
        prompt = f"""
        Query: {query}
        Available evidence: {self.summarize_docs(docs)}
        Generated response: {response}
        
        Critique this response:
        1. Is it factually supported by the evidence?
        2. Does it fully address the query?
        3. Are there any unsupported claims?
        4. What information is missing?
        
        Return JSON: {{
            "has_issues": true/false,
            "issues": ["issue1", "issue2"],
            "missing_info": ["topic1", "topic2"]
        }}
        """
        return self.llm.generate(prompt)
```

### RAG as a Tool

```python
# Make RAG a tool that agents can use
class RAGTool:
    def __init__(self, vector_store, description):
        self.name = "search_knowledge_base"
        self.description = description
        self.vector_store = vector_store
    
    def run(self, query: str, n_results: int = 5) -> str:
        results = self.vector_store.query(
            query_texts=[query],
            n_results=n_results
        )
        return "\n\n".join(results["documents"][0])

# Usage in agent
tools = [
    RAGTool(
        company_docs_store,
        "Search company documentation and policies"
    ),
    RAGTool(
        product_store,
        "Search product specifications and manuals"
    )
]
```

## 📖 Reading Materials

1. [Self-RAG Paper](https://arxiv.org/abs/2310.11511)
2. [CRAG: Corrective RAG](https://arxiv.org/abs/2401.15884)
3. [LangChain RAG Agents](https://python.langchain.com/docs/tutorials/rag/)

## ✅ Activity: Build a RAG Agent

Create `day11-rag-agent.py` implementing:

1. Query reformulation
2. Multi-step retrieval
3. Relevance evaluation
4. Response with citations

### Complete This Day

[![Complete Day 11](https://img.shields.io/badge/Complete--Day--11-28a745?logo=github)](../../issues/new?title=Complete+Day+11&labels=complete-day-11&body=✅+I%27ve+completed+Day+11%21)

---

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1-10 | Foundations Complete | ✅ |
| 🎁 **Day 11** | RAG-Enhanced Agents | ✅ Current |
| 🎁 **Day 12** | Code Agents | 🔜 Next |
