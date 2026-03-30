---
layout: post
title: "How I Build and Ship AI Applications — My Development Workflow and Toolkit"
date: 2026-03-30 15:00:00 +0700
categories: [ai, tutorial, development]
tags: [ai-development, workflow, tooling, deployment]
---

# How I Build and Ship AI Applications — My Development Workflow and Toolkit

After shipping dozens of AI-powered projects, I've developed a workflow that balances speed, reliability, and cost efficiency. This isn't just about writing prompts—it's about building systems that actually work in production.

## The Challenge: AI Development Is Different

Traditional software development has clear patterns: test-driven development, CI/CD pipelines, version control for code. But AI development? It's a Wild West. Models change, prompts break, and "it works on my machine" is a daily reality.

I've learned that successful AI development requires a different approach. Here's what works for me.

## Phase 1: Model Selection — Not All Models Are Created Equal

Before writing a single line of code, I choose the right model for the job. This isn't just about picking the biggest model available.

### The Model Decision Matrix

| Task Type | Recommended Models | Why |
|----------|------------------|-----|
| Code generation | Claude 3.5 Sonnet, GLM-4.7 | Better at following instructions, lower hallucination |
| Content creation | GPT-4, Claude 3 Opus | Creative, understands context |
| Data analysis | GLM-5, Claude 3.5 Sonnet | Strong reasoning, structured output |
| Simple chat | Local models, smaller LLMs | Cost-effective, fast response |

### My Rule of Thumb

```bash
# When I reach for different models
if "complex reasoning" in task:
    use claude-3-5-sonnet
elif "creative writing" in task:
    use gpt-4
elif "simple Q&A" in task:
    use local-model or glm-5
elif "code generation" in task:
    use claude-3-5-sonnet
```

## Phase 2: Prompt Engineering — Beyond the Basics

Everyone talks about prompt engineering, but most stop at "write clear prompts." That's not enough for production systems.

### Template-Based Prompting

I use structured templates for consistency:

```python
# prompt_template.py
PROMPT_TEMPLATES = {
    "code_review": """
You are an expert code reviewer. Analyze the following code:

```python
{code}
```

Provide feedback on:
1. Code quality and readability
2. Potential bugs or edge cases
3. Performance considerations
4. Security implications

Format your response as:
## Issues Found
- [Issue description]
- [Issue description]

## Recommendations
- [Suggestion]
- [Suggestion]
""",
    
    "content_generation": """
Generate {content_type} about {topic} targeting {audience}.

Requirements:
- Tone: {tone}
- Length: {length} words
- Key points to include: {key_points}
- Avoid: {avoid}

Structure:
1. Hook (opening paragraph)
2. Main content (2-3 paragraphs)
3. Conclusion with call-to-action
"""
}
```

### Dynamic Prompt Injection

For complex workflows, I inject context dynamically:

```python
def build_contextual_prompt(base_prompt, context_data):
    """Build prompts with relevant context"""
    prompt = base_prompt
    
    # Add relevant memory/context
    if "user_preferences" in context_data:
        prompt += f"\n\nUser preferences: {context_data['user_preferences']}"
    
    # Add previous conversation history
    if "history" in context_data:
        prompt += f"\n\nPrevious context: {context_data['history']}"
    
    # Add constraints
    if "constraints" in context_data:
        prompt += f"\n\nConstraints: {context_data['constraints']}"
    
    return prompt
```

## Phase 3: Development Workflow

### Local First, Then Cloud

I start development locally to iterate quickly without burning API tokens:

```bash
# My local development setup
mkdir -p ai-project
cd ai-project

# Create project structure
touch app.py prompts.py config.py tests/
touch prompts/code_review.py prompts/content_generation.py

# Local development requirements
pip install openai python-dotenv pytest
```

### Iterative Testing Loop

```python
# test_prompt.py
import pytest
from app import generate_review

@pytest.mark.parametrize("code_snippet, expected_issues", [
    ("def buggy_function():\n    return x", 1),
    ("def good_function():\n    return 'hello'", 0),
])
def test_code_review(code_snippet, expected_issues):
    result = generate_review(code_snippet)
    assert len(result["issues"]) == expected_issues
```

### Mock Testing for Consistency

AI models aren't deterministic, so I create deterministic mocks for unit tests:

```python
# mocks.py
MOCK_RESPONSES = {
    "code_review": {
        "issues": ["Variable 'x' is not defined"],
        "recommendations": ["Define variable 'x' before using it"]
    },
    "content_generation": {
        "content": "This is test content...",
        "word_count": 150
    }
}

def mock_ai_call(prompt, model="mock"):
    """Return deterministic responses for testing"""
    return MOCK_RESPONSES.get(prompt["type"], {})
```

## Phase 4: Production Considerations

### Cost Optimization

```python
# cost_monitor.py
import logging

class CostMonitor:
    def __init__(self, budget_per_day=10.0):
        self.budget = budget_per_day
        self.spent = 0.0
        self.logger = logging.getLogger(__name__)
    
    def check_cost(self, tokens_used, model="gpt-4"):
        """Check if call fits within budget"""
        # Rough cost estimates per 1K tokens
        costs = {
            "gpt-4": 0.03,  # input
            "gpt-4": 0.06,  # output
            "claude-3-5-sonnet": 0.015,
            "claude-3-5-sonnet": 0.075,
            "glm-5": 0.01,
        }
        
        estimated_cost = (tokens_used / 1000) * costs.get(model, 0.02)
        
        if self.spent + estimated_cost > self.budget:
            self.logger.warning(f"Cost limit reached: ${self.spent:.2f}/${self.budget}")
            return False
        
        self.spent += estimated_cost
        return True
```

### Fallback Strategies

```python
# fallback.py
class AIFallback:
    def __init__(self):
        self.models = ["claude-3-5-sonnet", "glm-5", "gpt-4"]
        self.fallback_chain = []
    
    def call_with_fallback(self, prompt):
        """Try models in order, fallback on failure"""
        for model in self.models:
            try:
                response = self._call_model(prompt, model)
                if self._validate_response(response):
                    return response
            except Exception as e:
                self.fallback_chain.append({"model": model, "error": str(e)})
                continue
        
        # Final fallback: simple template response
        return self._emergency_fallback(prompt)
```

### Monitoring and Observability

```python
# monitoring.py
import time
from dataclasses import dataclass

@dataclass
class AICallMetrics:
    model: str
    prompt: str
    response_time: float
    tokens_used: int
    success: bool
    error: str = None

class AIMonitor:
    def __init__(self):
        self.metrics = []
    
    def track_call(self, call_data):
        """Track AI call performance and metrics"""
        metrics = AICallMetrics(**call_data)
        self.metrics.append(metrics)
        
        # Log performance issues
        if metrics.response_time > 10:  # seconds
            logging.warning(f"Slow AI call: {metrics.response_time}s")
        
        if metrics.tokens_used > 10000:
            logging.warning(f"High token usage: {metrics.tokens_used}")
```

## Phase 5: Deployment Pipeline

### Containerization for Consistency

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create non-root user
RUN useradd -m -u 1001 appuser && chown -R appuser:appuser /app
USER appuser

# Expose port (if needed)
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD ["python", "app.py", "--check-health"] || exit 1

CMD ["python", "app.py"]
```

### CI/CD with AI-Specific Checks

```yaml
# .github/workflows/ai-workflow.yml
name: AI Application CI/CD

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install dependencies
      run: pip install -r requirements.txt
    
    - name: Run unit tests
      run: pytest tests/ -v
    
    - name: Test prompt templates
      run: python tests/test_prompts.py
    
    - name: Cost analysis test
      run: python tests/test_cost_limits.py
    
    - name: AI integration test
      env:
        OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      run: python tests/test_ai_calls.py
```

## My Essential Toolkit

### Development Tools

1. **Prompt Engineering**: `langchain` for prompt templates
2. **Testing**: `pytest` with mocking for non-deterministic AI calls
3. **Monitoring**: Custom metrics tracking + logging
4. **Cost Management**: Real-time cost tracking with budget alerts

### Production Tools

1. **Containerization**: Docker for consistent environments
2. **Orchestration**: Docker Compose for local dev, Kubernetes for production
3. **Monitoring**: Prometheus + Grafana for performance metrics
4. **Logging**: Structured logging with correlation IDs

### Observability Stack

```python
# observability.py
import structlog
from opentelemetry import trace

# Structured logging with AI-specific context
logger = structlog.get_logger()
tracer = trace.get_tracer(__name__)

def log_ai_call(model, prompt, response, duration, cost):
    """Log AI calls with full context"""
    logger.info(
        "ai_call_completed",
        model=model,
        prompt_preview=prompt[:100],
        response_preview=response[:100],
        duration_seconds=duration,
        estimated_cost=cost,
        timestamp=time.time()
    )
```

## Lessons Learned

### What Works

1. **Template-based prompting** - Consistent results, easier to test
2. **Cost monitoring** - Prevents bill shocks
3. **Fallback strategies** - Systems that don't fail completely
4. **Observability** - Can't optimize what you can't measure

### What Doesn't Work

1. **Single-model dependency** - Too risky, model changes break everything
2. **Ignoring costs** - AI bills can spiral out of control
3. **No testing** - AI responses are too unpredictable to skip testing
4. **Poor logging** - Debugging AI calls is impossible without proper logs

## The Future: AI Development Will Evolve

As AI models become more reliable, the patterns will converge toward traditional software development. But for now, this hybrid approach—combining software engineering best practices with AI-specific considerations—has served me well.

The key is treating AI as a component, not a magic solution. It has strengths and limitations, just like any other technology. Build systems that account for both.

---

**Tech used:** Python, Docker, OpenAI API, Anthropic API, Prometheus, Grafana  
**Difficulty:** Intermediate  
**Time to read:** ~10 minutes

What's your AI development workflow? Drop your tips in the comments!