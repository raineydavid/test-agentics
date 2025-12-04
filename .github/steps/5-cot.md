# 🎄 Day 5: Chain of Thought Reasoning

> To reset your progress, click: [![Reset Progress](https://img.shields.io/badge/Reset--Progress-ff3860?logo=mattermost)](../../issues/new?title=Reset+Progress&labels=reset&body=🔄+Reset+my+progress)

## 📋 Prerequisites

- Completed Days 1-4
- Understanding of ReAct

## 📝 Overview

**Chain of Thought (CoT)** prompting enables LLMs to break down complex problems into intermediate reasoning steps. It's fundamental to agent reasoning!

### What is Chain of Thought?

Instead of jumping directly to an answer, CoT guides the model through step-by-step reasoning:

```
Without CoT:
Q: Roger has 5 tennis balls. He buys 2 cans of 3. How many does he have?
A: 11

With CoT:
Q: Roger has 5 tennis balls. He buys 2 cans of 3. How many does he have?
A: Roger started with 5 balls.
   2 cans of 3 balls each = 2 × 3 = 6 balls.
   Total = 5 + 6 = 11 tennis balls.
```

### Types of Chain of Thought

#### 1. Zero-Shot CoT
Just add "Let's think step by step"

```python
prompt = """
Q: A bat and a ball cost $1.10 in total. The bat costs $1.00 more than the ball. 
How much does the ball cost?

Let's think step by step.
"""
```

#### 2. Few-Shot CoT
Provide examples with reasoning

```python
prompt = """
Q: There are 15 trees in the grove. Grove workers plant trees today. 
After they are done, there will be 21 trees. How many trees did they plant?
A: We start with 15 trees. Later we have 21 trees. 
The difference must be the number of trees planted.
21 - 15 = 6. The answer is 6.

Q: If there are 3 cars in the parking lot and 2 more cars arrive, 
how many cars are in the parking lot?
A: There are 3 cars in the parking lot already. 2 more arrive. 
Now there are 3 + 2 = 5 cars. The answer is 5.

Q: {new_question}
A:
"""
```

#### 3. Self-Consistency
Generate multiple CoT paths and take majority vote

```python
def self_consistent_cot(question: str, num_samples: int = 5):
    answers = []
    for _ in range(num_samples):
        # Generate CoT response with temperature > 0
        response = llm.generate(
            cot_prompt.format(question=question),
            temperature=0.7
        )
        answer = extract_answer(response)
        answers.append(answer)
    
    # Return most common answer
    return max(set(answers), key=answers.count)
```

### CoT in Agents

Agents use CoT for:
- **Planning**: Breaking tasks into subtasks
- **Tool Selection**: Reasoning about which tool to use
- **Error Analysis**: Understanding what went wrong
- **Decision Making**: Choosing between options

```python
AGENT_COT_PROMPT = """
Task: {task}

Let me think through this step by step:

1. First, I need to understand what's being asked...
2. The relevant tools I have available are...
3. The best approach would be...
4. Let me start by...

Based on this reasoning, I will:
Action: {action}
"""
```

### CoT + ReAct = Powerful Agents

```
Standard ReAct:
Thought: I should search for this.
Action: Search[query]

CoT-Enhanced ReAct:
Thought: The user is asking about X. To answer this, I need to:
  1. Find information about A
  2. Then relate it to B
  3. Finally calculate C
Let me start with step 1.
Action: Search[query about A]
```

## 📖 Reading Materials

1. [Chain-of-Thought Paper (Wei et al.)](https://arxiv.org/abs/2201.11903)
2. [Self-Consistency Paper](https://arxiv.org/abs/2203.11171)
3. [Automatic Chain-of-Thought](https://arxiv.org/abs/2210.03493)

## ✅ Activity: Implement CoT Reasoning

Create `day5-cot.py` implementing:

1. A function using zero-shot CoT
2. A function using few-shot CoT (with 3+ examples)
3. Bonus: Self-consistency with multiple samples

**Test problems:**
- Math word problems
- Logic puzzles
- Multi-step reasoning questions

### Complete This Day

[![Complete Day 5](https://img.shields.io/badge/Complete--Day--5-28a745?logo=github)](../../issues/new?title=Complete+Day+5&labels=complete-day-5&body=✅+I%27ve+completed+Day+5%21)

---

| Day | Topic | Status |
|-----|-------|--------|
| ✅ Day 1-4 | Agent Foundations | Complete |
| 🎁 **Day 5** | Chain of Thought Reasoning | ✅ Current |
| 🎁 **Day 6** | Short-term Memory | 🔜 Next |
