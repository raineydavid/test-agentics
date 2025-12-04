# 🎄 Day 16: Agent Orchestration

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Days 1-15
- Understanding of multi-agent communication

## 📝 Overview

**Agent Orchestration** is the art of coordinating multiple agents to work together efficiently. Today we learn how to be the conductor of an AI orchestra!

### Orchestration Patterns

```
┌─────────────────────────────────────────────────────────────┐
│              ORCHESTRATION PATTERNS                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. CENTRALIZED           2. DECENTRALIZED                   │
│     [Orchestrator]           [A] ←→ [B]                     │
│      ↙   ↓   ↘                ↕     ↕                       │
│    [A] [B] [C]               [C] ←→ [D]                     │
│                                                              │
│  3. WORKFLOW-BASED        4. DYNAMIC ROUTING                 │
│    Start → [A] → [B] → End   Router selects agent           │
│              ↓                based on task type             │
│             [C]                                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Orchestrator Agent

```python
class Orchestrator:
    def __init__(self, llm, agents: Dict[str, Agent]):
        self.llm = llm
        self.agents = agents
        self.task_log = []
    
    def run(self, task: str) -> str:
        # Plan the workflow
        workflow = self.plan_workflow(task)
        
        # Execute workflow
        context = {"original_task": task}
        
        for step in workflow:
            agent = self.agents[step["agent"]]
            subtask = step["task"]
            
            # Execute and collect result
            result = agent.run(subtask, context)
            context[step["agent"]] = result
            
            self.task_log.append({
                "step": step,
                "result": result
            })
        
        # Synthesize final result
        return self.synthesize(task, context)
    
    def plan_workflow(self, task: str) -> List[dict]:
        available_agents = "\n".join([
            f"- {name}: {agent.role.description}"
            for name, agent in self.agents.items()
        ])
        
        prompt = f"""
        Task: {task}
        
        Available agents:
        {available_agents}
        
        Create a workflow to complete this task.
        Return as JSON: [
            {{"agent": "agent_name", "task": "subtask description"}},
            ...
        ]
        """
        response = self.llm.generate(prompt)
        return json.loads(response)
    
    def synthesize(self, original_task: str, context: dict) -> str:
        prompt = f"""
        Original task: {original_task}
        
        Results from agents:
        {json.dumps(context, indent=2)}
        
        Synthesize a final response that addresses the original task.
        """
        return self.llm.generate(prompt)
```

### Dynamic Router

```python
class DynamicRouter:
    def __init__(self, llm, agents: Dict[str, Agent]):
        self.llm = llm
        self.agents = agents
    
    def route(self, task: str) -> str:
        # Classify the task
        agent_name = self.classify_task(task)
        
        # Route to appropriate agent
        if agent_name in self.agents:
            return self.agents[agent_name].run(task)
        
        # Fallback: use default or error
        return self.handle_unknown(task)
    
    def classify_task(self, task: str) -> str:
        agent_descriptions = "\n".join([
            f"- {name}: {agent.role.description}"
            for name, agent in self.agents.items()
        ])
        
        prompt = f"""
        Task: {task}
        
        Available agents:
        {agent_descriptions}
        
        Which agent is best suited for this task?
        Return only the agent name.
        """
        return self.llm.generate(prompt).strip()
```

### Workflow Engine

```python
from enum import Enum

class StepStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"

@dataclass
class WorkflowStep:
    id: str
    agent: str
    task: str
    dependencies: List[str]  # IDs of steps that must complete first
    status: StepStatus = StepStatus.PENDING
    result: Optional[str] = None

class WorkflowEngine:
    def __init__(self, agents: Dict[str, Agent]):
        self.agents = agents
        self.steps: Dict[str, WorkflowStep] = {}
    
    def add_step(self, step: WorkflowStep):
        self.steps[step.id] = step
    
    def run(self) -> dict:
        results = {}
        
        while not self.is_complete():
            # Find ready steps (dependencies met)
            ready = self.get_ready_steps()
            
            for step in ready:
                step.status = StepStatus.RUNNING
                
                try:
                    # Build context from completed dependencies
                    context = {
                        dep: self.steps[dep].result 
                        for dep in step.dependencies
                    }
                    
                    # Execute
                    agent = self.agents[step.agent]
                    step.result = agent.run(step.task, context)
                    step.status = StepStatus.COMPLETED
                    results[step.id] = step.result
                    
                except Exception as e:
                    step.status = StepStatus.FAILED
                    step.result = str(e)
        
        return results
    
    def is_complete(self) -> bool:
        return all(
            s.status in [StepStatus.COMPLETED, StepStatus.FAILED]
            for s in self.steps.values()
        )
    
    def get_ready_steps(self) -> List[WorkflowStep]:
        ready = []
        for step in self.steps.values():
            if step.status != StepStatus.PENDING:
                continue
            
            # Check if all dependencies are complete
            deps_complete = all(
                self.steps[dep].status == StepStatus.COMPLETED
                for dep in step.dependencies
            )
            
            if deps_complete:
                ready.append(step)
        
        return ready
```

### Supervisor Pattern

```python
class Supervisor:
    """Monitors agent work and intervenes when needed"""
    
    def __init__(self, llm, agents: Dict[str, Agent]):
        self.llm = llm
        self.agents = agents
        self.intervention_count = 0
    
    def supervised_run(self, agent_name: str, task: str) -> str:
        agent = self.agents[agent_name]
        
        # Initial run
        result = agent.run(task)
        
        # Review the work
        review = self.review_work(task, result)
        
        while not review["acceptable"]:
            self.intervention_count += 1
            
            if self.intervention_count > 3:
                # Take over or escalate
                return self.handle_escalation(task, result, review)
            
            # Provide feedback and retry
            revised_task = f"""
            Original task: {task}
            Your previous result: {result}
            Feedback: {review["feedback"]}
            
            Please revise your work based on the feedback.
            """
            result = agent.run(revised_task)
            review = self.review_work(task, result)
        
        return result
    
    def review_work(self, task: str, result: str) -> dict:
        prompt = f"""
        Task: {task}
        Agent's result: {result}
        
        Review this work:
        1. Does it address the task completely?
        2. Is it accurate and high quality?
        3. Are there any issues?
        
        Return JSON: {{
            "acceptable": true/false,
            "feedback": "specific feedback if not acceptable"
        }}
        """
        return json.loads(self.llm.generate(prompt))
```

## 📖 Reading Materials

1. [LangGraph Orchestration](https://langchain-ai.github.io/langgraph/)
2. [Prefect for Workflows](https://www.prefect.io/)
3. [Multi-Agent Orchestration Patterns](https://arxiv.org/abs/2308.08155)

## ✅ Activity: Build an Orchestrator

Create `day16-orchestrator.py` implementing:

1. An orchestrator that plans and coordinates
2. A workflow engine with dependencies
3. A supervisor for quality control

### Complete This Day

[![Complete Day 16](https://img.shields.io/badge/Complete--Day--16-28a745?logo=github)](../../issues/new?title=Complete+Day+16&labels=complete-day-16&body=✅+I%27ve+completed+Day+16%21)

---

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1-15 | Previous Days | Complete |
| 🎁 **Day 16** | Agent Orchestration | ✅ Current |
| 🎁 **Day 17** | LangChain Agents | 🔜 Next |
