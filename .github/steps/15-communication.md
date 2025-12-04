# 🎄 Day 15: Agent Communication Patterns

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Days 1-14
- Understanding of multi-agent basics

## 📝 Overview

Effective **agent communication** is key to successful multi-agent systems. Today we explore different patterns for agents to share information and coordinate!

### Communication Patterns

```
┌─────────────────────────────────────────────────────────────┐
│              COMMUNICATION PATTERNS                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. DIRECT MESSAGING      2. BROADCAST                       │
│     A ──→ B                  A ──→ [B, C, D]                │
│                                                              │
│  3. SHARED BLACKBOARD     4. PUB/SUB                        │
│     [A, B, C] ←→ 📋        A publishes → Topic              │
│                             B, C subscribe to Topic          │
│                                                              │
│  5. CONVERSATION          6. DELEGATION                      │
│     A ←→ B ←→ C            Manager assigns tasks            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Message Structure

```python
@dataclass
class AgentMessage:
    id: str
    sender: str
    recipient: str  # or "broadcast"
    message_type: str  # request, response, inform, query
    content: dict
    timestamp: datetime
    thread_id: Optional[str] = None
    
    def to_dict(self):
        return {
            "id": self.id,
            "sender": self.sender,
            "recipient": self.recipient,
            "type": self.message_type,
            "content": self.content,
            "timestamp": self.timestamp.isoformat(),
            "thread_id": self.thread_id
        }
```

### Pattern 1: Direct Messaging

```python
class DirectMessaging:
    def __init__(self):
        self.message_queues = {}  # agent_name -> [messages]
    
    def register_agent(self, agent_name: str):
        self.message_queues[agent_name] = []
    
    def send(self, message: AgentMessage):
        if message.recipient in self.message_queues:
            self.message_queues[message.recipient].append(message)
    
    def receive(self, agent_name: str) -> List[AgentMessage]:
        messages = self.message_queues.get(agent_name, [])
        self.message_queues[agent_name] = []
        return messages
```

### Pattern 2: Shared Blackboard

```python
class Blackboard:
    """Shared workspace where agents can read/write"""
    
    def __init__(self):
        self.data = {}
        self.history = []
    
    def write(self, agent: str, key: str, value: any):
        self.data[key] = {
            "value": value,
            "author": agent,
            "timestamp": datetime.now()
        }
        self.history.append({
            "action": "write",
            "agent": agent,
            "key": key,
            "timestamp": datetime.now()
        })
    
    def read(self, key: str) -> any:
        return self.data.get(key, {}).get("value")
    
    def get_all(self) -> dict:
        return {k: v["value"] for k, v in self.data.items()}
    
    def get_by_agent(self, agent: str) -> dict:
        return {k: v["value"] for k, v in self.data.items() 
                if v["author"] == agent}

class BlackboardAgent:
    def __init__(self, name: str, llm, blackboard: Blackboard):
        self.name = name
        self.llm = llm
        self.blackboard = blackboard
    
    def run(self, task: str):
        # Read current blackboard state
        context = self.blackboard.get_all()
        
        # Process task with context
        result = self.llm.generate(
            f"Task: {task}\nShared context: {context}"
        )
        
        # Write results back
        self.blackboard.write(self.name, f"{self.name}_result", result)
        return result
```

### Pattern 3: Conversation (Multi-turn)

```python
class ConversationManager:
    def __init__(self):
        self.conversations = {}  # thread_id -> messages
    
    def start_conversation(self, participants: List[str]) -> str:
        thread_id = str(uuid.uuid4())
        self.conversations[thread_id] = {
            "participants": participants,
            "messages": []
        }
        return thread_id
    
    def add_message(self, thread_id: str, message: AgentMessage):
        self.conversations[thread_id]["messages"].append(message)
    
    def get_conversation(self, thread_id: str) -> List[AgentMessage]:
        return self.conversations[thread_id]["messages"]

class ConversationalAgent:
    def __init__(self, name: str, llm, conv_manager: ConversationManager):
        self.name = name
        self.llm = llm
        self.conv_manager = conv_manager
    
    def respond(self, thread_id: str) -> AgentMessage:
        # Get conversation history
        history = self.conv_manager.get_conversation(thread_id)
        
        # Generate response
        response = self.llm.generate(
            system=f"You are {self.name}. Respond to this conversation.",
            messages=[m.to_dict() for m in history]
        )
        
        # Create and add message
        message = AgentMessage(
            id=str(uuid.uuid4()),
            sender=self.name,
            recipient="conversation",
            message_type="response",
            content={"text": response},
            timestamp=datetime.now(),
            thread_id=thread_id
        )
        self.conv_manager.add_message(thread_id, message)
        return message
```

### Pattern 4: Pub/Sub

```python
class PubSubBroker:
    def __init__(self):
        self.subscriptions = {}  # topic -> [agent_names]
        self.messages = {}  # topic -> [messages]
    
    def subscribe(self, agent: str, topic: str):
        if topic not in self.subscriptions:
            self.subscriptions[topic] = []
        self.subscriptions[topic].append(agent)
    
    def publish(self, agent: str, topic: str, content: dict):
        if topic not in self.messages:
            self.messages[topic] = []
        
        message = {
            "sender": agent,
            "content": content,
            "timestamp": datetime.now()
        }
        self.messages[topic].append(message)
        
        # Notify subscribers
        return self.subscriptions.get(topic, [])
    
    def get_messages(self, topic: str, since: datetime = None) -> List:
        msgs = self.messages.get(topic, [])
        if since:
            msgs = [m for m in msgs if m["timestamp"] > since]
        return msgs
```

### Complete Communication System

```python
class AgentCommunicationHub:
    def __init__(self):
        self.direct = DirectMessaging()
        self.blackboard = Blackboard()
        self.conversations = ConversationManager()
        self.pubsub = PubSubBroker()
    
    def send_direct(self, from_agent, to_agent, content):
        message = AgentMessage(
            id=str(uuid.uuid4()),
            sender=from_agent,
            recipient=to_agent,
            message_type="direct",
            content=content,
            timestamp=datetime.now()
        )
        self.direct.send(message)
    
    def broadcast(self, from_agent, content):
        self.pubsub.publish(from_agent, "broadcast", content)
    
    def share_on_blackboard(self, agent, key, value):
        self.blackboard.write(agent, key, value)
```

## 📖 Reading Materials

1. [Multi-Agent Communication](https://arxiv.org/abs/2308.08155)
2. [AutoGen Conversations](https://microsoft.github.io/autogen/docs/tutorial/conversation-patterns)
3. [Agent Protocols](https://agentprotocol.ai/)

## ✅ Activity: Implement Communication

Create `day15-communication.py` implementing:

1. At least 2 communication patterns
2. A multi-agent task using your patterns
3. Message logging and visualization

### Complete This Day

[![Complete Day 15](https://img.shields.io/badge/Complete--Day--15-28a745?logo=github)](../../issues/new?title=Complete+Day+15&labels=complete-day-15&body=✅+I%27ve+completed+Day+15%21)

---

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1-14 | Previous Days | Complete |
| 🎁 **Day 15** | Agent Communication Patterns | ✅ Current |
| 🎁 **Day 16** | Agent Orchestration | 🔜 Next |
