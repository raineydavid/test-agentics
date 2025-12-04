# 🎄 Day 20: AutoGen & Multi-Agent Conversations

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Days 1-19
- `pip install pyautogen`

## 📝 Overview

**AutoGen** by Microsoft enables building multi-agent systems where agents can have conversations with each other and with humans!

### AutoGen Concepts

```
┌─────────────────────────────────────────────────────────────┐
│                    AUTOGEN CONCEPTS                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ConversableAgent: Base class for all agents                │
│  AssistantAgent: AI-powered agent                           │
│  UserProxyAgent: Represents human, can execute code         │
│  GroupChat: Multi-agent conversation                        │
│  GroupChatManager: Orchestrates group conversations         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Basic Two-Agent Chat

```python
import autogen

# Configure LLM
config_list = [
    {
        "model": "claude-sonnet-4-20250514",
        "api_key": "your-api-key",
        "api_type": "anthropic"
    }
]

llm_config = {
    "config_list": config_list,
    "temperature": 0.7
}

# Create assistant agent
assistant = autogen.AssistantAgent(
    name="assistant",
    llm_config=llm_config,
    system_message="""You are a helpful AI assistant. 
    Solve tasks step by step. If code is needed, write Python code."""
)

# Create user proxy (can execute code)
user_proxy = autogen.UserProxyAgent(
    name="user_proxy",
    human_input_mode="NEVER",  # or "ALWAYS", "TERMINATE"
    max_consecutive_auto_reply=10,
    is_termination_msg=lambda x: x.get("content", "").rstrip().endswith("TERMINATE"),
    code_execution_config={
        "work_dir": "coding",
        "use_docker": False
    }
)

# Start conversation
user_proxy.initiate_chat(
    assistant,
    message="Calculate the factorial of 10 and plot the growth of factorials from 1-10"
)
```

### Group Chat

```python
# Create specialized agents
researcher = autogen.AssistantAgent(
    name="Researcher",
    llm_config=llm_config,
    system_message="""You are a research specialist.
    Find and analyze information on given topics."""
)

writer = autogen.AssistantAgent(
    name="Writer",
    llm_config=llm_config,
    system_message="""You are a content writer.
    Create engaging content based on research."""
)

critic = autogen.AssistantAgent(
    name="Critic",
    llm_config=llm_config,
    system_message="""You are a content critic.
    Review content and provide constructive feedback."""
)

user_proxy = autogen.UserProxyAgent(
    name="User",
    human_input_mode="TERMINATE",
    max_consecutive_auto_reply=0
)

# Create group chat
groupchat = autogen.GroupChat(
    agents=[user_proxy, researcher, writer, critic],
    messages=[],
    max_round=12
)

manager = autogen.GroupChatManager(
    groupchat=groupchat,
    llm_config=llm_config
)

# Start group conversation
user_proxy.initiate_chat(
    manager,
    message="Create a blog post about quantum computing for beginners"
)
```

### Custom Speaker Selection

```python
def custom_speaker_selection(
    last_speaker: autogen.Agent,
    groupchat: autogen.GroupChat
) -> autogen.Agent:
    """Custom logic to select next speaker"""
    messages = groupchat.messages
    
    if len(messages) <= 1:
        return researcher  # Start with research
    
    last_message = messages[-1]["content"].lower()
    
    if "research" in last_message and last_speaker == researcher:
        return writer
    elif "draft" in last_message and last_speaker == writer:
        return critic
    elif "approved" in last_message:
        return user_proxy
    
    # Default: let the manager decide
    return None

groupchat = autogen.GroupChat(
    agents=[user_proxy, researcher, writer, critic],
    messages=[],
    max_round=12,
    speaker_selection_method=custom_speaker_selection
)
```

### Agent with Tools/Functions

```python
# Define functions
def get_weather(city: str) -> str:
    """Get weather for a city"""
    return f"Weather in {city}: Sunny, 72°F"

def calculate(expression: str) -> str:
    """Evaluate a math expression"""
    return str(eval(expression))

# Register functions with assistant
assistant = autogen.AssistantAgent(
    name="assistant",
    llm_config={
        "config_list": config_list,
        "functions": [
            {
                "name": "get_weather",
                "description": "Get weather for a city",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "city": {"type": "string", "description": "City name"}
                    },
                    "required": ["city"]
                }
            },
            {
                "name": "calculate",
                "description": "Evaluate a math expression",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "expression": {"type": "string"}
                    },
                    "required": ["expression"]
                }
            }
        ]
    }
)

# Register function implementations
user_proxy.register_function(
    function_map={
        "get_weather": get_weather,
        "calculate": calculate
    }
)
```

### Nested Chats

```python
# Inner chat for specific sub-tasks
def research_chat(message: str):
    """Conduct a research chat"""
    inner_assistant = autogen.AssistantAgent(
        name="inner_researcher",
        llm_config=llm_config,
        system_message="You are a deep researcher."
    )
    
    inner_user = autogen.UserProxyAgent(
        name="inner_user",
        human_input_mode="NEVER",
        max_consecutive_auto_reply=3
    )
    
    inner_user.initiate_chat(inner_assistant, message=message)
    return inner_user.last_message()["content"]

# Main agent can trigger nested chats
main_assistant = autogen.AssistantAgent(
    name="main_assistant",
    llm_config=llm_config,
    system_message="""You coordinate complex tasks.
    Use research_chat for deep research needs."""
)

user_proxy.register_function(
    function_map={"research_chat": research_chat}
)
```

### Teachable Agent (Learning)

```python
from autogen.agentchat.contrib.teachable_agent import TeachableAgent

teachable = TeachableAgent(
    name="teachable_assistant",
    llm_config=llm_config,
    teach_config={
        "verbosity": 1,
        "reset_db": False,
        "path_to_db_dir": "./teachable_db"
    }
)

# The agent learns from corrections
user_proxy.initiate_chat(
    teachable,
    message="My favorite programming language is Python"
)

# Later...
user_proxy.initiate_chat(
    teachable,
    message="What's my favorite programming language?"
)
# Will remember: "Your favorite language is Python"
```

### Conversation Patterns

```python
# Sequential chat
result = user_proxy.initiate_chats([
    {
        "recipient": researcher,
        "message": "Research AI trends",
        "max_turns": 3
    },
    {
        "recipient": writer,
        "message": "Write about the research findings",
        "max_turns": 3
    }
])

# Two-agent chat with carryover
user_proxy.initiate_chat(
    assistant,
    message="Hello",
    carryover="Previous conversation context here"
)
```

## 📖 Reading Materials

1. [AutoGen Documentation](https://microsoft.github.io/autogen/)
2. [AutoGen GitHub](https://github.com/microsoft/autogen)
3. [AutoGen Examples](https://microsoft.github.io/autogen/docs/Examples)

## ✅ Activity: Build with AutoGen

Create `day20-autogen.py` implementing:

1. A group chat with 3+ agents
2. Custom speaker selection
3. At least one tool/function

### Complete This Day

[![Complete Day 20](https://img.shields.io/badge/Complete--Day--20-28a745?logo=github)](../../issues/new?title=Complete+Day+20&labels=complete-day-20&body=✅+I%27ve+completed+Day+20%21)

---

🎉 **Day 20 Complete! Only 5 days left!** 🎉

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1-19 | Previous Days | Complete |
| 🎁 **Day 20** | AutoGen | ✅ Current |
| 🎁 **Day 21** | MCP (Model Context Protocol) | 🔜 Next |
