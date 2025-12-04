# 🎄 Day 23: Agent Safety & Guardrails

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Days 1-22
- Understanding of agent architecture

## 📝 Overview

Building **safe and reliable agents** is crucial. Today we explore guardrails, safety mechanisms, and best practices!

### Why Safety Matters

```
┌─────────────────────────────────────────────────────────────┐
│                    AGENT RISKS                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  🔴 Harmful outputs (toxic, biased, misleading)             │
│  🔴 Unauthorized actions (data access, modifications)        │
│  🔴 Infinite loops and resource exhaustion                   │
│  🔴 Prompt injection attacks                                 │
│  🔴 Data leakage and privacy violations                      │
│  🔴 Unintended side effects                                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Defense in Depth

```
┌─────────────────────────────────────────────────────────────┐
│                  DEFENSE LAYERS                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Layer 1: Input Validation                                   │
│      ↓                                                       │
│  Layer 2: Prompt Engineering                                 │
│      ↓                                                       │
│  Layer 3: Output Filtering                                   │
│      ↓                                                       │
│  Layer 4: Action Validation                                  │
│      ↓                                                       │
│  Layer 5: Monitoring & Alerting                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Input Validation

```python
import re
from typing import Optional

class InputValidator:
    def __init__(self):
        self.blocked_patterns = [
            r"ignore\s+previous\s+instructions",
            r"system\s+prompt",
            r"you\s+are\s+now",
            r"act\s+as\s+if",
            r"pretend\s+to\s+be",
        ]
        self.max_length = 10000
    
    def validate(self, user_input: str) -> tuple[bool, Optional[str]]:
        # Length check
        if len(user_input) > self.max_length:
            return False, "Input too long"
        
        # Pattern check (prompt injection detection)
        lower_input = user_input.lower()
        for pattern in self.blocked_patterns:
            if re.search(pattern, lower_input):
                return False, "Potentially harmful input detected"
        
        # Character encoding check
        try:
            user_input.encode('utf-8')
        except UnicodeError:
            return False, "Invalid character encoding"
        
        return True, None

class SafeAgent:
    def __init__(self, agent, validator: InputValidator):
        self.agent = agent
        self.validator = validator
    
    def run(self, user_input: str) -> str:
        is_valid, error = self.validator.validate(user_input)
        
        if not is_valid:
            return f"I cannot process this request: {error}"
        
        return self.agent.run(user_input)
```

### Output Filtering

```python
class OutputFilter:
    def __init__(self, llm=None):
        self.llm = llm
        self.blocked_content = [
            "password",
            "api_key",
            "secret",
            "private_key"
        ]
    
    def filter(self, output: str) -> tuple[str, bool]:
        """Filter output. Returns (filtered_output, was_modified)"""
        
        # Check for sensitive data patterns
        filtered = output
        modified = False
        
        for pattern in self.blocked_content:
            if pattern.lower() in output.lower():
                filtered = re.sub(
                    rf'{pattern}\s*[=:]\s*\S+',
                    f'{pattern}=[REDACTED]',
                    filtered,
                    flags=re.IGNORECASE
                )
                modified = True
        
        # Optional: Use LLM to check for harmful content
        if self.llm:
            is_safe = self.check_safety_with_llm(filtered)
            if not is_safe:
                return "I cannot provide that response.", True
        
        return filtered, modified
    
    def check_safety_with_llm(self, content: str) -> bool:
        prompt = f"""
        Evaluate if this content is safe to share:
        
        {content[:1000]}
        
        Check for:
        - Harmful instructions
        - Personal information
        - Offensive content
        - Misinformation
        
        Respond with only: SAFE or UNSAFE
        """
        response = self.llm.generate(prompt)
        return "SAFE" in response
```

### Action Guardrails

```python
from enum import Enum
from typing import List, Callable

class ActionRisk(Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    CRITICAL = "critical"

class ActionGuardrail:
    def __init__(self):
        self.action_risks = {
            "read_file": ActionRisk.LOW,
            "write_file": ActionRisk.MEDIUM,
            "execute_code": ActionRisk.HIGH,
            "delete_file": ActionRisk.CRITICAL,
            "send_email": ActionRisk.HIGH,
            "make_api_call": ActionRisk.MEDIUM
        }
        self.blocked_actions = ["delete_database", "format_disk"]
        self.require_confirmation = [ActionRisk.HIGH, ActionRisk.CRITICAL]
    
    def check_action(
        self,
        action_name: str,
        arguments: dict,
        confirm_callback: Callable = None
    ) -> tuple[bool, str]:
        # Check if action is blocked
        if action_name in self.blocked_actions:
            return False, f"Action '{action_name}' is not allowed"
        
        # Get risk level
        risk = self.action_risks.get(action_name, ActionRisk.MEDIUM)
        
        # Require confirmation for high-risk actions
        if risk in self.require_confirmation:
            if confirm_callback:
                confirmed = confirm_callback(action_name, arguments)
                if not confirmed:
                    return False, "Action not confirmed by user"
            else:
                return False, f"High-risk action '{action_name}' requires confirmation"
        
        # Validate arguments
        is_valid, error = self.validate_arguments(action_name, arguments)
        if not is_valid:
            return False, error
        
        return True, "Action approved"
    
    def validate_arguments(self, action: str, args: dict) -> tuple[bool, str]:
        if action == "write_file":
            path = args.get("path", "")
            # Prevent writing to system directories
            if path.startswith("/etc/") or path.startswith("/system/"):
                return False, "Cannot write to system directories"
        
        if action == "execute_code":
            code = args.get("code", "")
            # Check for dangerous operations
            dangerous_patterns = ["os.system", "subprocess", "eval(", "__import__"]
            for pattern in dangerous_patterns:
                if pattern in code:
                    return False, f"Code contains potentially dangerous operation: {pattern}"
        
        return True, None
```

### Rate Limiting & Resource Management

```python
import time
from collections import defaultdict

class RateLimiter:
    def __init__(
        self,
        max_requests_per_minute: int = 60,
        max_tokens_per_minute: int = 100000
    ):
        self.max_requests = max_requests_per_minute
        self.max_tokens = max_tokens_per_minute
        self.request_timestamps = []
        self.token_usage = []
    
    def check_rate_limit(self, estimated_tokens: int = 0) -> tuple[bool, str]:
        now = time.time()
        minute_ago = now - 60
        
        # Clean old entries
        self.request_timestamps = [
            ts for ts in self.request_timestamps if ts > minute_ago
        ]
        self.token_usage = [
            (ts, tokens) for ts, tokens in self.token_usage if ts > minute_ago
        ]
        
        # Check request count
        if len(self.request_timestamps) >= self.max_requests:
            return False, "Request rate limit exceeded"
        
        # Check token usage
        total_tokens = sum(tokens for _, tokens in self.token_usage)
        if total_tokens + estimated_tokens > self.max_tokens:
            return False, "Token rate limit exceeded"
        
        return True, "OK"
    
    def record_usage(self, tokens_used: int):
        now = time.time()
        self.request_timestamps.append(now)
        self.token_usage.append((now, tokens_used))

class ResourceManager:
    def __init__(self, max_memory_mb: int = 512, max_time_seconds: int = 300):
        self.max_memory = max_memory_mb * 1024 * 1024
        self.max_time = max_time_seconds
        self.start_time = None
    
    def start_task(self):
        self.start_time = time.time()
    
    def check_resources(self) -> tuple[bool, str]:
        import psutil
        
        # Check memory
        process = psutil.Process()
        memory = process.memory_info().rss
        if memory > self.max_memory:
            return False, "Memory limit exceeded"
        
        # Check time
        if self.start_time:
            elapsed = time.time() - self.start_time
            if elapsed > self.max_time:
                return False, "Time limit exceeded"
        
        return True, "OK"
```

### Comprehensive Safe Agent

```python
class SafeAgentSystem:
    def __init__(self, agent, llm):
        self.agent = agent
        self.input_validator = InputValidator()
        self.output_filter = OutputFilter(llm)
        self.action_guardrail = ActionGuardrail()
        self.rate_limiter = RateLimiter()
        self.resource_manager = ResourceManager()
        self.audit_log = []
    
    def run(self, user_input: str) -> str:
        # Log the request
        self.audit_log.append({
            "timestamp": time.time(),
            "type": "request",
            "input": user_input[:100]  # Truncate for logging
        })
        
        # Check rate limits
        allowed, reason = self.rate_limiter.check_rate_limit()
        if not allowed:
            return f"Request denied: {reason}"
        
        # Validate input
        is_valid, error = self.input_validator.validate(user_input)
        if not is_valid:
            return f"Invalid input: {error}"
        
        # Start resource tracking
        self.resource_manager.start_task()
        
        try:
            # Run agent with action interception
            result = self.agent.run(
                user_input,
                action_callback=self.intercept_action
            )
            
            # Filter output
            filtered_result, was_modified = self.output_filter.filter(result)
            
            if was_modified:
                self.audit_log.append({
                    "timestamp": time.time(),
                    "type": "output_filtered",
                    "original_length": len(result)
                })
            
            return filtered_result
        
        except Exception as e:
            self.audit_log.append({
                "timestamp": time.time(),
                "type": "error",
                "error": str(e)
            })
            return "An error occurred while processing your request."
    
    def intercept_action(self, action_name: str, arguments: dict) -> tuple[bool, str]:
        # Check resources
        ok, reason = self.resource_manager.check_resources()
        if not ok:
            return False, reason
        
        # Check action guardrails
        return self.action_guardrail.check_action(action_name, arguments)
```

## 📖 Reading Materials

1. [Anthropic Safety Research](https://www.anthropic.com/research)
2. [Guardrails AI](https://www.guardrailsai.com/)
3. [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/)

## ✅ Activity: Build Safe Guardrails

Create `day23-safety.py` implementing:

1. Input validation with prompt injection detection
2. Output filtering for sensitive data
3. Action guardrails with risk levels
4. Rate limiting

### Complete This Day

[![Complete Day 23](https://img.shields.io/badge/Complete--Day--23-28a745?logo=github)](../../issues/new?title=Complete+Day+23&labels=complete-day-23&body=✅+I%27ve+completed+Day+23%21)

---

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1-22 | Previous Days | Complete |
| 🎁 **Day 23** | Agent Safety & Guardrails | ✅ Current |
| 🎁 **Day 24** | Production Deployment | 🔜 Next |
