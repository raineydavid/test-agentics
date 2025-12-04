# 🎄 Day 9: Planning Strategies

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Days 1-8
- Understanding of agent architecture

## 📝 Overview

**Planning** is how agents break down complex tasks into achievable steps. Today we explore different planning strategies!

### Why Planning Matters

Without planning:
- Agent gets lost in complex tasks
- Inefficient tool usage
- Prone to going off-track
- Hard to recover from errors

With planning:
- Clear path to goal
- Efficient resource use
- Easier error recovery
- Better task completion

### Planning Strategies

#### 1. Plan-and-Execute (Upfront Planning)

```python
class PlanAndExecuteAgent:
    def __init__(self, llm, executor):
        self.planner = llm
        self.executor = executor
    
    def run(self, task: str) -> str:
        # Phase 1: Create complete plan upfront
        plan = self.create_plan(task)
        
        # Phase 2: Execute each step
        results = []
        for step in plan:
            result = self.executor.execute(step)
            results.append(result)
        
        return self.synthesize_results(results)
    
    def create_plan(self, task: str) -> List[str]:
        prompt = f"""
        Create a step-by-step plan to accomplish this task:
        
        Task: {task}
        
        Return a numbered list of steps. Each step should be:
        - Specific and actionable
        - Independent enough to execute separately
        - Building toward the final goal
        """
        response = self.planner.generate(prompt)
        return self.parse_plan(response)
```

#### 2. ReWOO (Reasoning Without Observation)

Plan all reasoning first, then execute tools:

```python
class ReWOOAgent:
    def run(self, task: str) -> str:
        # Generate complete reasoning trace with placeholders
        reasoning = self.generate_reasoning(task)
        
        # Extract and execute all tool calls
        tool_results = {}
        for tool_call in self.extract_tool_calls(reasoning):
            result = self.execute_tool(tool_call)
            tool_results[tool_call.id] = result
        
        # Fill in placeholders with results
        filled_reasoning = self.fill_placeholders(reasoning, tool_results)
        
        return self.extract_answer(filled_reasoning)
```

#### 3. Hierarchical Planning (Task Decomposition)

```python
class HierarchicalPlanner:
    def plan(self, task: str, depth: int = 0, max_depth: int = 3) -> dict:
        if depth >= max_depth:
            return {"task": task, "subtasks": [], "is_atomic": True}
        
        # Ask LLM to decompose
        subtasks = self.decompose(task)
        
        if len(subtasks) <= 1:
            return {"task": task, "subtasks": [], "is_atomic": True}
        
        return {
            "task": task,
            "subtasks": [
                self.plan(st, depth + 1, max_depth) 
                for st in subtasks
            ],
            "is_atomic": False
        }
    
    def decompose(self, task: str) -> List[str]:
        prompt = f"""
        Break down this task into 2-4 smaller subtasks:
        
        Task: {task}
        
        If the task is already simple/atomic, just return the task itself.
        """
        return self.llm.generate(prompt)
```

#### 4. Iterative Refinement Planning

```python
class IterativePlanner:
    def plan(self, task: str, max_iterations: int = 3) -> List[str]:
        # Initial plan
        plan = self.create_initial_plan(task)
        
        for i in range(max_iterations):
            # Critique the plan
            critique = self.critique_plan(task, plan)
            
            if critique.is_satisfactory:
                break
            
            # Refine based on critique
            plan = self.refine_plan(task, plan, critique)
        
        return plan
    
    def critique_plan(self, task: str, plan: List[str]) -> Critique:
        prompt = f"""
        Evaluate this plan for the given task:
        
        Task: {task}
        Plan: {plan}
        
        Consider:
        1. Completeness: Does it cover all requirements?
        2. Feasibility: Are all steps achievable?
        3. Efficiency: Is there a more direct path?
        4. Dependencies: Are steps in the right order?
        """
        return self.llm.generate(prompt)
```

#### 5. Adaptive Planning (Re-plan on Failure)

```python
class AdaptivePlanningAgent:
    def run(self, task: str) -> str:
        plan = self.create_plan(task)
        completed_steps = []
        
        while plan:
            step = plan.pop(0)
            try:
                result = self.execute_step(step)
                completed_steps.append({"step": step, "result": result})
            except Exception as e:
                # Re-plan from current state
                plan = self.replan(
                    original_task=task,
                    completed=completed_steps,
                    failed_step=step,
                    error=str(e)
                )
        
        return self.synthesize_results(completed_steps)
    
    def replan(self, original_task, completed, failed_step, error):
        prompt = f"""
        The original task was: {original_task}
        
        Completed steps: {completed}
        
        The following step failed: {failed_step}
        Error: {error}
        
        Create a new plan to complete the task from here.
        """
        return self.llm.generate(prompt)
```

### Comparing Strategies

| Strategy | Best For | Trade-offs |
|----------|----------|------------|
| **Plan-and-Execute** | Predictable tasks | Inflexible to changes |
| **ReWOO** | Parallel tool calls | Needs complete info upfront |
| **Hierarchical** | Complex tasks | Overhead for simple tasks |
| **Iterative** | Quality-critical | Slower, more LLM calls |
| **Adaptive** | Uncertain environments | May thrash on repeated failures |

## 📖 Reading Materials

1. [Plan-and-Solve Prompting](https://arxiv.org/abs/2305.04091)
2. [ReWOO Paper](https://arxiv.org/abs/2305.18323)
3. [LangGraph Planning](https://langchain-ai.github.io/langgraph/)

## ✅ Activity: Implement a Planner

Create `day9-planner.py` implementing:

1. At least 2 planning strategies
2. Plan visualization (print the plan structure)
3. Test with a complex multi-step task

### Complete This Day

[![Complete Day 9](https://img.shields.io/badge/Complete--Day--9-28a745?logo=github)](../../issues/new?title=Complete+Day+9&labels=complete-day-9&body=✅+I%27ve+completed+Day+9%21)

---

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1-8 | Memory Systems | Complete |
| 🎁 **Day 9** | Planning Strategies | ✅ Current |
| 🎁 **Day 10** | Error Handling & Recovery | 🔜 Next |
