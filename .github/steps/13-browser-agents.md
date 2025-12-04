# 🎄 Day 13: Browser Agents

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Days 1-12
- Understanding of web browsers

## 📝 Overview

**Browser Agents** can navigate the web, interact with websites, and extract information - like having a tireless research assistant!

### Browser Agent Capabilities

- Navigate to URLs
- Click buttons and links
- Fill forms
- Extract text and data
- Take screenshots
- Handle authentication

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    BROWSER AGENT                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Task → Observe Page → Decide Action → Execute → Observe    │
│                                                              │
│  Actions: click, type, scroll, navigate, extract, screenshot│
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Using Playwright for Browser Control

```python
from playwright.sync_api import sync_playwright

class BrowserController:
    def __init__(self, headless=True):
        self.playwright = sync_playwright().start()
        self.browser = self.playwright.chromium.launch(headless=headless)
        self.page = self.browser.new_page()
    
    def navigate(self, url: str):
        self.page.goto(url)
        return self.get_page_state()
    
    def click(self, selector: str):
        self.page.click(selector)
        self.page.wait_for_load_state('networkidle')
        return self.get_page_state()
    
    def type_text(self, selector: str, text: str):
        self.page.fill(selector, text)
        return self.get_page_state()
    
    def screenshot(self) -> bytes:
        return self.page.screenshot()
    
    def get_page_state(self) -> dict:
        return {
            'url': self.page.url,
            'title': self.page.title(),
            'content': self.page.content()[:5000],  # Truncate
            'visible_text': self.page.inner_text('body')[:2000]
        }
    
    def extract_elements(self, selector: str) -> list:
        elements = self.page.query_selector_all(selector)
        return [{'text': el.inner_text(), 'tag': el.evaluate('el => el.tagName')} 
                for el in elements]
    
    def close(self):
        self.browser.close()
        self.playwright.stop()
```

### Building the Browser Agent

```python
class BrowserAgent:
    def __init__(self, llm, browser: BrowserController):
        self.llm = llm
        self.browser = browser
        self.action_history = []
    
    def run(self, task: str) -> str:
        max_steps = 20
        
        for step in range(max_steps):
            # Get current page state
            state = self.browser.get_page_state()
            
            # Decide next action
            action = self.decide_action(task, state, self.action_history)
            
            if action['type'] == 'done':
                return action['result']
            
            # Execute action
            result = self.execute_action(action)
            self.action_history.append({
                'action': action,
                'result': result
            })
        
        return "Task incomplete after max steps"
    
    def decide_action(self, task: str, state: dict, history: list) -> dict:
        prompt = f"""
        Task: {task}
        
        Current page:
        - URL: {state['url']}
        - Title: {state['title']}
        - Visible text: {state['visible_text'][:1000]}
        
        Previous actions: {self.format_history(history)}
        
        What action should be taken next?
        
        Available actions:
        - navigate: Go to a URL
        - click: Click an element (provide CSS selector)
        - type: Type text (provide selector and text)
        - extract: Extract information
        - scroll: Scroll the page
        - done: Task complete (provide result)
        
        Return JSON: {{"type": "action_type", "selector": "...", "value": "...", "result": "..."}}
        """
        response = self.llm.generate(prompt)
        return json.loads(response)
    
    def execute_action(self, action: dict) -> dict:
        action_type = action['type']
        
        if action_type == 'navigate':
            return self.browser.navigate(action['value'])
        elif action_type == 'click':
            return self.browser.click(action['selector'])
        elif action_type == 'type':
            return self.browser.type_text(action['selector'], action['value'])
        elif action_type == 'extract':
            return {'extracted': self.browser.get_page_state()['visible_text']}
        elif action_type == 'scroll':
            self.browser.page.evaluate('window.scrollBy(0, 500)')
            return self.browser.get_page_state()
```

### Vision-Based Browser Agent

Using screenshots for more robust understanding:

```python
class VisionBrowserAgent(BrowserAgent):
    def decide_action(self, task: str, state: dict, history: list) -> dict:
        # Take screenshot
        screenshot = self.browser.screenshot()
        screenshot_b64 = base64.b64encode(screenshot).decode()
        
        # Use vision model
        response = self.llm.generate_with_image(
            prompt=f"""
            Task: {task}
            Previous actions: {self.format_history(history)}
            
            Look at this screenshot and decide the next action.
            Return JSON with action type and details.
            """,
            image=screenshot_b64
        )
        return json.loads(response)
```

### Browser Agent Tools

```python
# Define as tools for a general agent
browser_tools = [
    {
        "name": "navigate",
        "description": "Navigate to a URL",
        "parameters": {"url": "string"}
    },
    {
        "name": "click",
        "description": "Click an element on the page",
        "parameters": {"selector": "CSS selector or description"}
    },
    {
        "name": "type",
        "description": "Type text into an input field",
        "parameters": {"selector": "string", "text": "string"}
    },
    {
        "name": "extract_text",
        "description": "Extract text from the current page",
        "parameters": {"selector": "optional CSS selector"}
    },
    {
        "name": "screenshot",
        "description": "Take a screenshot of the current page",
        "parameters": {}
    }
]
```

## 📖 Reading Materials

1. [Playwright Documentation](https://playwright.dev/python/docs/intro)
2. [Claude Computer Use](https://docs.anthropic.com/en/docs/agents-and-tools/computer-use)
3. [WebVoyager Paper](https://arxiv.org/abs/2401.13919)

## ✅ Activity: Build a Browser Agent

Create `day13-browser-agent.py` implementing:

1. Basic browser control with Playwright
2. Action decision based on page state
3. Complete a simple web task (e.g., search, form fill)

### Complete This Day

[![Complete Day 13](https://img.shields.io/badge/Complete--Day--13-28a745?logo=github)](../../issues/new?title=Complete+Day+13&labels=complete-day-13&body=✅+I%27ve+completed+Day+13%21)

---

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1-12 | Previous Days | Complete |
| 🎁 **Day 13** | Browser Agents | ✅ Current |
| 🎁 **Day 14** | Multi-Agent Basics | 🔜 Next |
