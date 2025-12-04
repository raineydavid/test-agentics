# 🎄 Day 10: Error Handling & Recovery

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Days 1-9
- Understanding of agent architecture and planning

## 📝 Overview

Real-world agents encounter errors constantly. Today we learn to build **robust agents** that handle failures gracefully!

### Types of Agent Errors

```
┌─────────────────────────────────────────────────────────────┐
│                    ERROR CATEGORIES                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Tool Errors        → API failures, timeouts, invalid input │
│  LLM Errors         → Rate limits, malformed outputs        │
│  Logic Errors       → Wrong tool choice, infinite loops      │
│  Environmental      → Resource unavailable, permissions      │
│  Planning Errors    → Impossible plan, missing dependencies  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Error Handling Strategies

#### 1. Retry with Exponential Backoff

```python
import time
from functools import wraps

def retry_with_backoff(max_retries=3, base_delay=1):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_retries - 1:
                        raise
                    delay = base_delay * (2 ** attempt)
                    print(f"Attempt {attempt + 1} failed: {e}. Retrying in {delay}s...")
                    time.sleep(delay)
        return wrapper
    return decorator

class RobustAgent:
    @retry_with_backoff(max_retries=3)
    def call_tool(self, tool_name: str, args: dict):
        return self.tools[tool_name].run(**args)
```

#### 2. Self-Correction

```python
class SelfCorrectingAgent:
    def execute_with_correction(self, task: str, max_corrections: int = 3):
        for attempt in range(max_corrections):
            try:
                result = self.execute(task)
                
                # Validate result
                validation = self.validate_result(task, result)
                
                if validation.is_valid:
                    return result
                
                # Self-correct based on validation feedback
                task = self.create_correction_prompt(
                    original_task=task,
                    result=result,
                    feedback=validation.feedback
                )
                
            except Exception as e:
                task = self.create_error_recovery_prompt(task, e)
        
        raise MaxCorrectionsExceeded()
    
    def validate_result(self, task: str, result: str):
        prompt = f"""
        Task: {task}
        Result: {result}
        
        Evaluate if this result correctly addresses the task.
        Return JSON: {{"is_valid": true/false, "feedback": "..."}}
        """
        return self.llm.generate(prompt)
```

#### 3. Fallback Strategies

```python
class FallbackAgent:
    def __init__(self, primary_tools, fallback_tools):
        self.primary = primary_tools
        self.fallback = fallback_tools
    
    def call_with_fallback(self, tool_type: str, args: dict):
        # Try primary tool first
        primary_tool = self.primary.get(tool_type)
        if primary_tool:
            try:
                return primary_tool.run(**args)
            except Exception as e:
                print(f"Primary tool failed: {e}")
        
        # Fall back to alternative
        fallback_tool = self.fallback.get(tool_type)
        if fallback_tool:
            try:
                return fallback_tool.run(**args)
            except Exception as e:
                print(f"Fallback tool also failed: {e}")
        
        # Final fallback: ask LLM to handle without tool
        return self.llm_fallback(tool_type, args)
```

#### 4. Error Classification and Handling

```python
class ErrorClassifier:
    ERROR_HANDLERS = {
        "rate_limit": "wait_and_retry",
        "invalid_input": "fix_and_retry", 
        "not_found": "try_alternative",
        "timeout": "retry_with_longer_timeout",
        "permission": "escalate_to_user",
        "unknown": "log_and_continue"
    }
    
    def classify_error(self, error: Exception) -> str:
        error_str = str(error).lower()
        
        if "rate limit" in error_str or "429" in error_str:
            return "rate_limit"
        elif "invalid" in error_str or "validation" in error_str:
            return "invalid_input"
        elif "not found" in error_str or "404" in error_str:
            return "not_found"
        elif "timeout" in error_str:
            return "timeout"
        elif "permission" in error_str or "403" in error_str:
            return "permission"
        else:
            return "unknown"
    
    def handle_error(self, error: Exception, context: dict):
        error_type = self.classify_error(error)
        handler = self.ERROR_HANDLERS[error_type]
        return getattr(self, handler)(error, context)
```

#### 5. Graceful Degradation

```python
class GracefulAgent:
    def run(self, task: str) -> str:
        result = {
            "status": "success",
            "answer": None,
            "partial_results": [],
            "errors": []
        }
        
        steps = self.plan(task)
        
        for step in steps:
            try:
                step_result = self.execute_step(step)
                result["partial_results"].append(step_result)
            except Exception as e:
                result["errors"].append({"step": step, "error": str(e)})
                
                # Try to continue with partial information
                if not self.is_critical_step(step):
                    continue
                else:
                    result["status"] = "partial"
                    break
        
        # Synthesize best possible answer from partial results
        result["answer"] = self.synthesize_partial(
            task, 
            result["partial_results"],
            result["errors"]
        )
        
        return result
```

### Loop Detection

```python
class LoopDetector:
    def __init__(self, max_similar_actions=3):
        self.action_history = []
        self.max_similar = max_similar_actions
    
    def record_action(self, action: dict):
        self.action_history.append(action)
        
        if self.detect_loop():
            raise InfiniteLoopDetected(
                f"Agent appears stuck. Last {self.max_similar} actions were similar."
            )
    
    def detect_loop(self) -> bool:
        if len(self.action_history) < self.max_similar:
            return False
        
        recent = self.action_history[-self.max_similar:]
        
        # Check if recent actions are too similar
        similarity_scores = []
        for i in range(len(recent) - 1):
            score = self.compute_similarity(recent[i], recent[i+1])
            similarity_scores.append(score)
        
        return all(s > 0.9 for s in similarity_scores)
```

## 📖 Reading Materials

1. [Reflexion: Language Agents with Verbal Reinforcement](https://arxiv.org/abs/2303.11366)
2. [Self-Refine: Iterative Refinement](https://arxiv.org/abs/2303.17651)
3. [Building Reliable Agents](https://www.anthropic.com/research/building-effective-agents)

## ✅ Activity: Build Robust Error Handling

Create `day10-error-handling.py` implementing:

1. Retry with backoff
2. Self-correction capability
3. Loop detection
4. Graceful degradation

### Complete This Day

[![Complete Day 10](https://img.shields.io/badge/Complete--Day--10-28a745?logo=github)](../../issues/new?title=Complete+Day+10&labels=complete-day-10&body=✅+I%27ve+completed+Day+10%21)

---

🎉 **Congratulations! You've completed the first 10 days!** 🎉

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1-9 | Agent Foundations | Complete |
| 🎁 **Day 10** | Error Handling & Recovery | ✅ Current |
| 🎁 **Day 11** | RAG-Enhanced Agents | 🔜 Next |
