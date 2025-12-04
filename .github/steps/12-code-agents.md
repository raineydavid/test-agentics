# 🎄 Day 12: Code Agents

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Days 1-11
- Python or JavaScript environment

## 📝 Overview

**Code Agents** can write, execute, and debug code to solve problems. They're one of the most powerful agent applications!

### What Code Agents Can Do

- Write code to solve problems
- Execute code and observe results
- Debug errors iteratively
- Analyze files and data
- Create and modify projects

### Code Execution Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     CODE AGENT                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Task → Generate Code → Execute in Sandbox → Observe Output │
│                              ↓                               │
│                      Error? → Debug → Retry                  │
│                              ↓                               │
│                        Return Result                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Safe Code Execution

```python
import subprocess
import tempfile
import os

class CodeExecutor:
    def __init__(self, timeout=30, max_memory_mb=512):
        self.timeout = timeout
        self.max_memory = max_memory_mb
    
    def execute_python(self, code: str) -> dict:
        """Safely execute Python code in an isolated environment"""
        with tempfile.NamedTemporaryFile(
            mode='w', suffix='.py', delete=False
        ) as f:
            f.write(code)
            temp_file = f.name
        
        try:
            result = subprocess.run(
                ['python', temp_file],
                capture_output=True,
                text=True,
                timeout=self.timeout,
                env={**os.environ, 'PYTHONDONTWRITEBYTECODE': '1'}
            )
            
            return {
                'success': result.returncode == 0,
                'stdout': result.stdout,
                'stderr': result.stderr,
                'return_code': result.returncode
            }
        except subprocess.TimeoutExpired:
            return {
                'success': False,
                'stdout': '',
                'stderr': 'Execution timed out',
                'return_code': -1
            }
        finally:
            os.unlink(temp_file)
```

### Building a Code Agent

```python
class CodeAgent:
    def __init__(self, llm, executor):
        self.llm = llm
        self.executor = executor
        self.max_attempts = 5
    
    def solve(self, problem: str) -> str:
        # Generate initial code
        code = self.generate_code(problem)
        
        for attempt in range(self.max_attempts):
            # Execute the code
            result = self.executor.execute_python(code)
            
            if result['success']:
                return {
                    'code': code,
                    'output': result['stdout'],
                    'attempts': attempt + 1
                }
            
            # Debug and fix
            code = self.debug_code(problem, code, result['stderr'])
        
        return {'error': 'Could not solve after max attempts'}
    
    def generate_code(self, problem: str) -> str:
        prompt = f"""
        Write Python code to solve this problem:
        
        {problem}
        
        Requirements:
        - Write clean, working code
        - Include print statements to show results
        - Handle potential errors
        
        Return only the code, no explanations.
        """
        return self.llm.generate(prompt)
    
    def debug_code(self, problem: str, code: str, error: str) -> str:
        prompt = f"""
        The following code has an error:
        
        Code:
        ```python
        {code}
        ```
        
        Error:
        {error}
        
        Original problem: {problem}
        
        Fix the code and return only the corrected code.
        """
        return self.llm.generate(prompt)
```

### Advanced: REPL-style Agent

```python
class REPLAgent:
    """Interactive code agent that maintains state"""
    
    def __init__(self, llm):
        self.llm = llm
        self.history = []
        self.variables = {}
    
    def run(self, task: str) -> str:
        # Analyze task and plan steps
        plan = self.create_plan(task)
        
        for step in plan:
            # Generate code for this step
            code = self.generate_step_code(step, self.variables)
            
            # Execute and capture new variables
            result = self.execute_with_state(code)
            
            # Update state
            self.variables.update(result.get('new_vars', {}))
            self.history.append({
                'step': step,
                'code': code,
                'result': result
            })
        
        return self.synthesize_result()
```

### Security Considerations

| Risk | Mitigation |
|------|------------|
| Infinite loops | Timeout limits |
| Memory exhaustion | Memory limits |
| File system access | Sandboxing |
| Network access | Network isolation |
| Malicious code | Code review/sanitization |

## 📖 Reading Materials

1. [OpenAI Code Interpreter](https://platform.openai.com/docs/assistants/tools/code-interpreter)
2. [E2B Sandboxes](https://e2b.dev/docs)
3. [Claude Computer Use](https://docs.anthropic.com/en/docs/agents-and-tools/computer-use)

## ✅ Activity: Build a Code Agent

Create `day12-code-agent.py` implementing:

1. Safe code execution sandbox
2. Code generation from natural language
3. Iterative debugging
4. Result extraction

### Complete This Day

[![Complete Day 12](https://img.shields.io/badge/Complete--Day--12-28a745?logo=github)](../../issues/new?title=Complete+Day+12&labels=complete-day-12&body=✅+I%27ve+completed+Day+12%21)

---

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1-11 | Previous Days | Complete |
| 🎁 **Day 12** | Code Agents | ✅ Current |
| 🎁 **Day 13** | Browser Agents | 🔜 Next |
