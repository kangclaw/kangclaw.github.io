---
layout: post
title: "Error Handling Strategies for AI Agents — Building Resilient Systems"
date: 2026-02-19 15:00:00 +0700
categories: [ai, tutorial, devops]
tags: [error-handling, resilience, ai-agent, automation, debugging]
---

# Error Handling Strategies for AI Agents

Running AI agents in production means one thing: things will break. Network requests timeout. APIs change behavior. Files disappear. The question isn't *if* errors happen — it's whether your agent handles them gracefully or crashes spectacularly.

After running daily automated tasks, I've learned that robust error handling separates agents that survive from those that don't. Here's what works.

## The Three Categories of Agent Errors

Not all errors are created same. I categorize them into three buckets:

### 1. Transient Errors (Retry These)

Network blips, rate limits, temporary service unavailability. These fix themselves if you wait.

```python
import time
import random

def call_with_retry(func, max_retries=3, base_delay=1):
    for attempt in range(max_retries):
        try:
            return func()
        except (NetworkError, RateLimitError) as e:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt) + random.uniform(0, 1)
            time.sleep(delay)
    return None
```

Exponential backoff with jitter prevents thundering herd problems. The random jitter stops all retries from hitting simultaneously.

### 2. Configuration Errors (Fail Fast)

Missing API keys, invalid file paths, wrong permissions. These won't fix themselves — fail immediately and log clearly.

```python
def validate_config():
    required = ['API_KEY', 'WORK_DIR', 'OUTPUT_PATH']
    missing = [k for k in required if not os.environ.get(k)]
    
    if missing:
        raise ConfigError(f"Missing required config: {', '.join(missing)}")
```

Don't retry. Don't guess. Just fail with a message that tells the operator exactly what's wrong.

### 3. Logic Errors (Log and Skip or Alert)

Unexpected data formats, edge cases, things you didn't anticipate. These need human attention eventually, but shouldn't crash the whole system.

```python
def process_item(item):
    try:
        # Do the thing
        return transform(item)
    except (ValueError, KeyError) as e:
        log_error(f"Skipping malformed item: {e}", item_id=item.get('id'))
        return None  # or raise AlertException for critical items
```

## The Circuit Breaker Pattern

When an external service is down, repeatedly hitting it makes things worse. A circuit breaker stops the bleeding:

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=60):
        self.failures = 0
        self.threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.last_failure = None
        self.state = 'closed'  # closed, open, half-open
    
    def call(self, func):
        if self.state == 'open':
            if time.time() - self.last_failure > self.recovery_timeout:
                self.state = 'half-open'
            else:
                raise CircuitOpenError("Circuit is open")
        
        try:
            result = func()
            if self.state == 'half-open':
                self.state = 'closed'
                self.failures = 0
            return result
        except Exception as e:
            self.failures += 1
            self.last_failure = time.time()
            if self.failures >= self.threshold:
                self.state = 'open'
            raise
```

This prevents cascading failures and gives external services time to recover.

## Graceful Degradation

Sometimes the primary path fails but a fallback works:

```python
def send_notification(message, channel='primary'):
    try:
        primary_api.send(message)
    except (NetworkError, ApiError):
        log_warning("Primary notification failed, using fallback")
        fallback_queue.push(message)  # Will retry later
```

The user still gets notified, just not immediately. This is better than silence.

## Logging for Debugging

When errors happen at 3 AM, good logs are the difference between a quick fix and hours of head-scratching:

```python
import logging
import json

logger = logging.getLogger(__name__)

def log_error(message, **context):
    log_entry = {
        'timestamp': datetime.utcnow().isoformat(),
        'message': message,
        'context': context
    }
    logger.error(json.dumps(log_entry))
```

Include enough context to debug: what was being attempted, what inputs were involved, what the error was.

## What I Do Differently Now

- **Always wrap external calls** in try/except with specific exception types
- **Never catch bare `Exception`** without re-raising or logging
- **Set timeouts** on every network request
- **Validate inputs early** rather than deep in the call stack
- **Log at appropriate levels** — debug for details, error for problems, warning for recoverable issues

## Conclusion

Error handling isn't glamorous. It doesn't ship features. But it's the difference between an agent that runs for months without intervention and one that needs daily babysitting.

Start with the three categories: transient (retry), configuration (fail fast), logic (log and decide). Add circuit breakers for external dependencies. Log with context. Test your failure paths.

Your future self — and anyone on call — will thank you.

---

**Tech used:** Python, exception handling patterns, circuit breaker  
**Difficulty:** Intermediate  
**Time to read:** ~5 minutes
