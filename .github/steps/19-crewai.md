# 🎄 Day 19: CrewAI Framework

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Days 1-18
- `pip install crewai crewai-tools`

## 📝 Overview

**CrewAI** is a framework for orchestrating role-playing AI agents that collaborate like a real team!

### CrewAI Concepts

```
┌─────────────────────────────────────────────────────────────┐
│                    CREWAI CONCEPTS                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Agent: An AI with a role, goal, and backstory              │
│  Task: A specific job for an agent to complete              │
│  Tool: Capabilities an agent can use                         │
│  Crew: A team of agents working together                     │
│  Process: How the crew executes (sequential, hierarchical)  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Basic CrewAI Setup

```python
from crewai import Agent, Task, Crew, Process
from crewai_tools import SerperDevTool, WebsiteSearchTool

# Define tools
search_tool = SerperDevTool()

# Define agents
researcher = Agent(
    role="Senior Research Analyst",
    goal="Uncover cutting-edge developments in AI and data science",
    backstory="""You work at a leading tech think tank. Your expertise 
    lies in identifying emerging trends and technologies.""",
    tools=[search_tool],
    verbose=True,
    allow_delegation=False
)

writer = Agent(
    role="Tech Content Strategist",
    goal="Craft compelling content on tech advancements",
    backstory="""You are a renowned Content Strategist, known for 
    your insightful and engaging articles.""",
    tools=[],
    verbose=True,
    allow_delegation=False
)

# Define tasks
research_task = Task(
    description="""Conduct comprehensive research on the latest AI agents.
    Identify key trends, breakthrough technologies, and potential impacts.""",
    expected_output="A detailed report with key findings",
    agent=researcher
)

write_task = Task(
    description="""Using the research findings, write an engaging blog post
    about the state of AI agents in 2024.""",
    expected_output="A well-structured blog post with clear sections",
    agent=writer,
    context=[research_task]  # Depends on research
)

# Create crew
crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    process=Process.sequential,
    verbose=True
)

# Execute
result = crew.kickoff()
print(result)
```

### Advanced Agent Configuration

```python
from crewai import Agent
from langchain_anthropic import ChatAnthropic

# Use custom LLM
custom_llm = ChatAnthropic(model="claude-sonnet-4-20250514")

expert_agent = Agent(
    role="AI Expert",
    goal="Provide expert analysis on AI systems",
    backstory="""You are a world-renowned AI researcher with 20 years 
    of experience. You've published hundreds of papers and advised 
    major tech companies on AI strategy.""",
    llm=custom_llm,
    tools=[search_tool],
    max_iter=15,
    max_rpm=10,  # Rate limit
    verbose=True,
    allow_delegation=True,
    step_callback=lambda step: print(f"Step: {step}"),  # Monitoring
    memory=True  # Enable memory
)
```

### Custom Tools

```python
from crewai.tools import BaseTool
from pydantic import BaseModel, Field

class CalculatorInput(BaseModel):
    expression: str = Field(description="Math expression to evaluate")

class CalculatorTool(BaseTool):
    name: str = "Calculator"
    description: str = "Useful for performing mathematical calculations"
    args_schema: type[BaseModel] = CalculatorInput
    
    def _run(self, expression: str) -> str:
        try:
            result = eval(expression)
            return str(result)
        except Exception as e:
            return f"Error: {e}"

# Use in agent
math_agent = Agent(
    role="Data Analyst",
    goal="Analyze numerical data",
    backstory="Expert in statistics and data analysis",
    tools=[CalculatorTool()]
)
```

### Hierarchical Process

```python
# Manager agent oversees the crew
manager = Agent(
    role="Project Manager",
    goal="Coordinate the team to deliver high-quality output",
    backstory="Experienced PM who ensures smooth collaboration",
    allow_delegation=True
)

crew = Crew(
    agents=[researcher, writer, manager],
    tasks=[research_task, write_task],
    process=Process.hierarchical,
    manager_agent=manager,
    verbose=True
)
```

### Task Dependencies and Context

```python
# Complex task flow
task1 = Task(
    description="Research market trends",
    agent=researcher,
    expected_output="Market analysis report"
)

task2 = Task(
    description="Analyze competitors based on market research",
    agent=analyst,
    expected_output="Competitor analysis",
    context=[task1]  # Uses output from task1
)

task3 = Task(
    description="Create strategy recommendations",
    agent=strategist,
    expected_output="Strategy document",
    context=[task1, task2]  # Uses outputs from both
)

task4 = Task(
    description="Write executive summary",
    agent=writer,
    expected_output="Executive summary",
    context=[task3],
    output_file="summary.md"  # Save to file
)
```

### Async Execution

```python
import asyncio

async def run_crew():
    crew = Crew(
        agents=[researcher, writer],
        tasks=[research_task, write_task],
        process=Process.sequential
    )
    
    result = await crew.kickoff_async()
    return result

# Run async
result = asyncio.run(run_crew())
```

### Callbacks and Monitoring

```python
def task_callback(task_output):
    print(f"Task completed: {task_output.description}")
    print(f"Output: {task_output.raw_output[:200]}...")

def step_callback(step):
    print(f"Agent {step.agent} is working on: {step.task}")

crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    process=Process.sequential,
    task_callback=task_callback,
    step_callback=step_callback
)
```

### Full Example: Content Team

```python
from crewai import Agent, Task, Crew, Process

# The Team
researcher = Agent(
    role="Content Researcher",
    goal="Find accurate, interesting information",
    backstory="Meticulous researcher with journalism background"
)

seo_expert = Agent(
    role="SEO Specialist",
    goal="Optimize content for search engines",
    backstory="10 years in digital marketing and SEO"
)

writer = Agent(
    role="Content Writer",
    goal="Create engaging, informative content",
    backstory="Award-winning blogger and copywriter"
)

editor = Agent(
    role="Editor",
    goal="Ensure content quality and accuracy",
    backstory="Former newspaper editor with keen eye for detail"
)

# The Tasks
research = Task(
    description="Research {topic} thoroughly",
    agent=researcher,
    expected_output="Research notes with sources"
)

seo_analysis = Task(
    description="Identify keywords and SEO strategy for {topic}",
    agent=seo_expert,
    expected_output="SEO brief with keywords"
)

writing = Task(
    description="Write blog post incorporating research and SEO",
    agent=writer,
    context=[research, seo_analysis],
    expected_output="Draft blog post"
)

editing = Task(
    description="Edit and polish the blog post",
    agent=editor,
    context=[writing],
    expected_output="Final blog post",
    output_file="blog_post.md"
)

# The Crew
content_crew = Crew(
    agents=[researcher, seo_expert, writer, editor],
    tasks=[research, seo_analysis, writing, editing],
    process=Process.sequential,
    verbose=True
)

# Execute
result = content_crew.kickoff(inputs={"topic": "AI Agents in 2024"})
```

## 📖 Reading Materials

1. [CrewAI Documentation](https://docs.crewai.com/)
2. [CrewAI GitHub](https://github.com/joaomdmoura/crewAI)
3. [CrewAI Examples](https://github.com/joaomdmoura/crewAI-examples)

## ✅ Activity: Build a CrewAI Crew

Create `day19-crewai.py` implementing:

1. At least 4 specialized agents
2. Tasks with dependencies
3. A meaningful workflow (content, research, analysis, etc.)

### Complete This Day

[![Complete Day 19](https://img.shields.io/badge/Complete--Day--19-28a745?logo=github)](../../issues/new?title=Complete+Day+19&labels=complete-day-19&body=✅+I%27ve+completed+Day+19%21)

---

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1-18 | Previous Days | Complete |
| 🎁 **Day 19** | CrewAI Framework | ✅ Current |
| 🎁 **Day 20** | AutoGen & Multi-Agent Conversations | 🔜 Next |
