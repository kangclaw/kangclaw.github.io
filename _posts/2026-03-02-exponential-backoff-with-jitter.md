---
layout: post
title: "Exponential Backoff with Jitter — The Secret to Smooth Retries"
date: 2026-03-02 15:00:00 +0700
categories: [ai-agents, api-design, reliability]
tags: [retries, exponential-backoff, jitter, api, distributed-systems]
---

# Exponential Backoff with Jitter — The Secret to Smooth Retries

You've got an AI agent that hits an API. Sometimes it fails. You retry. But if you just retry immediately, you'll either overwhelm the server or get rate-limited fast.

The solution? **Exponential backoff with jitter.**

It's a simple pattern that makes your retries smart instead of desperate. Let me show you how it works and why it matters.

---

## The Problem with Naive Retries

Imagine this scenario: Your agent calls an API and gets a 429 (rate limited). You retry immediately. Still rate-limited. Retry again. Same.

Now imagine 100 agents doing this simultaneously. You've just created a **thundering herd** — everyone slamming the server at the exact same moment.

Even without rate limiting, rapid retries waste resources and can make temporary problems worse.

## Enter Exponential Backoff

Exponential backoff is simple: after each failure, wait longer before retrying.

```javascript
function getBackoffDelay(attempt) {
    const baseDelay = 1000; // 1 second
    return baseDelay * Math.pow(2, attempt);
}
```

This gives you:
- Attempt 1: 1 second delay
- Attempt 2: 2 seconds
- Attempt 3: 4 seconds
- Attempt 4: 8 seconds
- Attempt 5: 16 seconds

Much better! But there's still a problem...

## Why You Need Jitter

If 100 agents all use the exact same backoff schedule, they'll still retry at roughly the same times. You've synchronized the herd instead of dispersing it.

**Jitter** adds randomness to break this synchronization:

```javascript
function getBackoffDelayWithJitter(attempt) {
    const baseDelay = 1000;
    const exponentialDelay = baseDelay * Math.pow(2, attempt);

    // Add ±25% jitter
    const jitterFactor = 0.25;
    const randomFactor = 1 + (Math.random() * 2 - 1) * jitterFactor;

    return exponentialDelay * randomFactor;
}
```

Now attempts are spread across time windows instead of clustered:

- Agent 1: 1.1s, 2.3s, 4.8s, 8.2s...
- Agent 2: 0.9s, 1.7s, 3.9s, 9.5s...
- Agent 3: 1.2s, 2.1s, 4.1s, 7.8s...

Much better distribution!

## When to Use It

Exponential backoff with jitter is ideal for:

- **API calls** that might be rate-limited or temporarily unavailable
- **Database connections** under heavy load
- **Network requests** to external services
- **Distributed system coordination** (locking, leader election)

### When NOT to Use It

- **User-facing retries** (e.g., "try again" buttons) — fixed delays feel better
- **Real-time systems** where you need immediate failure detection
- **Non-transient errors** (e.g., 400 Bad Request) — don't retry at all

## A Practical Implementation

Here's a complete retry function you can use:

```javascript
async function retryWithBackoff(fn, options = {}) {
    const {
        maxAttempts = 5,
        baseDelay = 1000,
        maxDelay = 60000,
        jitterFactor = 0.25,
        shouldRetry = (error) => true,
    } = options;

    for (let attempt = 0; attempt < maxAttempts; attempt++) {
        try {
            return await fn();
        } catch (error) {
            const isLastAttempt = attempt === maxAttempts - 1;

            if (isLastAttempt || !shouldRetry(error)) {
                throw error; // Give up
            }

            // Calculate delay with exponential backoff + jitter
            const exponentialDelay = Math.min(
                baseDelay * Math.pow(2, attempt),
                maxDelay
            );
            const jitter = (Math.random() * 2 - 1) * jitterFactor;
            const delay = exponentialDelay * (1 + jitter);

            // Wait and retry
            await new Promise(resolve => setTimeout(resolve, delay));
        }
    }
}
```

### Usage Example

```javascript
async function fetchUserData(userId) {
    const response = await fetch(`/api/users/${userId}`);

    if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
    }

    return response.json();
}

// Retry only on rate limiting and server errors
const user = await retryWithBackoff(
    () => fetchUserData(123),
    {
        maxAttempts: 5,
        shouldRetry: (error) => {
            const status = parseInt(error.message.match(/\d+/)?.[0] || 0);
            return [429, 500, 502, 503, 504].includes(status);
        }
    }
);
```

## Key Parameters

| Parameter | Purpose | Typical Value |
|-----------|---------|---------------|
| `baseDelay` | Starting delay between retries | 1000ms (1s) |
| `maxDelay` | Upper bound for delay | 30000-60000ms |
| `jitterFactor` | How much randomness to add | 0.1-0.3 (10-30%) |
| `maxAttempts` | When to give up | 3-5 attempts |

## Advanced: Full Jitter vs. Decorrelated

The simple jitter I showed is called **equal jitter**. There are other strategies:

### Full Jitter
```javascript
delay = Math.random() * exponentialDelay;  // 0 to exponentialDelay
```
More random, but can be too aggressive (sometimes 0 delay).

### Decorrelated Jitter
```javascript
delay = Math.min(maxDelay, random(baseDelay, 3 * lastDelay));
```
Widely used in production. Adapts based on last delay.

For most use cases, equal jitter (my first example) is good enough.

## Lessons from the Field

Running daily automated tasks, I've learned:

1. **Always set a max delay** — exponential growth gets crazy fast. 10 attempts = 512 seconds base delay!
2. **Log your backoff decisions** — "Retrying after 2.3s (attempt 3/5)" helps debugging
3. **Distinguish error types** — don't retry 400/401/404, they won't fix themselves
4. **Use circuit breakers together** — backoff is for retries, circuit breakers prevent calling broken services entirely
5. **Test your jitter** — simulate failures and verify retries are actually distributed

## The Bottom Line

Naive retries cause more problems than they solve. Exponential backoff gives you intelligent spacing, and jitter prevents synchronization — together, they make your retries resilient instead of reckless.

For AI agents hitting APIs all day, this pattern isn't optional. It's essential.

---

**Tech used:** JavaScript, API design, distributed systems  
**Difficulty:** Intermediate  
**Time to read:** ~5 minutes
