---
layout: post
title: "Prompt Caching for AI Agents — Save Tokens, Cut Costs, Speed Up Responses"
date: 2026-03-12 15:00:00 +0700
categories: [ai, optimization]
tags: [prompt-caching, llm, cost-optimization, api, anthropic, ai-agents]
---

# Prompt Caching for AI Agents — Save Tokens, Cut Costs, Speed Up Responses

If you're running an AI agent that makes repeated API calls, you're probably burning tokens on the same context over and over. Prompt caching fixes that — and it's one of the most underrated cost optimization techniques for LLM applications.

Here's what I've learned from using it daily.

## What Is Prompt Caching?

When you send a prompt to an LLM API, the entire context gets processed every time. That includes:

- System instructions
- Conversation history
- Tool definitions
- Long documents you're analyzing

If you're making multiple calls with overlapping context, you're paying to process the same tokens repeatedly.

Prompt caching stores the processed prefix of your prompt on the provider's side. When you make another request with the same prefix, the cached portion skips reprocessing — saving tokens and latency.

## How It Works

Most major providers now support some form of caching:

**Anthropic (Claude):**
```python
# Mark content as cacheable with cache_control
message = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    system=[
        {
            "type": "text",
            "text": SYSTEM_PROMPT,  # Your long system prompt
            "cache_control": {"type": "ephemeral"}
        }
    ],
    messages=[...],
    max_tokens=1024
)
```

**OpenAI (GPT-4):**
```python
# Automatic for repeated prefixes — no code changes needed
# Just structure your prompts consistently
response = client.chat.completions.create(
    model="gpt-4-turbo",
    messages=[
        {"role": "system", "content": SYSTEM_PROMPT},  # Gets cached
        {"role": "user", "content": user_message}
    ]
)
```

The key insight: **structure matters**. Put stable content at the beginning of your prompt, dynamic content at the end.

## Where Caching Shines

### 1. Long System Prompts

If your agent has a 2000-token system prompt (instructions, personality, rules), that gets reprocessed on every call. With caching:

- First call: pay for 2000 tokens
- Subsequent calls: pay for 0 tokens (cached)

At 100 calls/day, that's 200,000 tokens saved daily.

### 2. Tool Definitions

Agents with many tools have massive schema definitions. These rarely change between calls:

```python
tools = [
    {"name": "read_file", "schema": {...}},
    {"name": "write_file", "schema": {...}},
    {"name": "execute_command", "schema": {...}},
    # ... 20 more tools
]

# All of this can be cached if placed consistently
```

### 3. Few-Shot Examples

If you include example outputs in your prompt, cache them:

```python
examples = """
Example 1:
Input: "Schedule a meeting"
Output: {"action": "create_event", "params": {...}}

Example 2:
Input: "Send an email"
Output: {"action": "send_email", "params": {...}}
"""
```

### 4. Document Analysis

Processing the same document multiple times? Cache it:

```python
# First call: analyze document structure
# Subsequent calls: ask different questions about same document
message = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "document",
                    "source": {"type": "text", "text": LONG_DOCUMENT},
                    "cache_control": {"type": "ephemeral"}
                },
                {"type": "text", "text": "What's the main topic?"}
            ]
        }
    ]
)
```

## Cache Hit Rates

Not every call will hit cache. Here's what affects hit rate:

| Factor | Impact |
|--------|--------|
| Consistent prompt structure | High |
| Same model version | High |
| Time between calls | Medium (caches expire) |
| Cache prefix length | Medium (min tokens required) |

**Anthropic** requires minimum 1024 tokens in the cached prefix. **OpenAI** has similar thresholds.

If your prefix is too short, caching won't activate.

## Real-World Savings

For an agent making 500 calls/day with a 3000-token system prompt:

| Metric | Without Cache | With Cache |
|--------|---------------|------------|
| Input tokens/call | 3,500 | 500 |
| Daily input tokens | 1,750,000 | 250,000 |
| Cost reduction | — | ~85% |

Cache writes cost a small premium (~25% more), but the savings on reads more than compensate.

## Best Practices

### Structure for Cacheability

```python
# GOOD: Stable content first
messages = [
    {"role": "system", "content": SYSTEM_PROMPT},      # Cached
    {"role": "system", "content": TOOL_DEFINITIONS},   # Cached
    {"role": "system", "content": FEW_SHOT_EXAMPLES},  # Cached
    {"role": "user", "content": user_message},         # Dynamic
]

# BAD: Dynamic content breaks cache
messages = [
    {"role": "user", "content": f"Time: {datetime.now()}"},  # Changes every call!
    {"role": "system", "content": SYSTEM_PROMPT},
]
```

### Monitor Cache Performance

```python
# Anthropic returns cache stats in response
usage = response.usage
cache_read = usage.cache_read_input_tokens  # Tokens saved
cache_write = usage.cache_write_input_tokens  # New cache writes

hit_rate = cache_read / (cache_read + cache_write + usage.input_tokens)
```

### Don't Cache Everything

Some content changes too frequently:

- Timestamps
- Session-specific IDs
- Real-time data

Put these at the end of your prompt, after the cacheable prefix.

## When Caching Doesn't Help

- **One-off requests** — no repeated context
- **Short prompts** — below minimum token threshold
- **Highly dynamic prompts** — prefix changes every call
- **Long idle periods** — cache expires between sessions

## Implementation Checklist

- [ ] Identify stable prompt content (system instructions, tools, examples)
- [ ] Move stable content to prompt beginning
- [ ] Add `cache_control` markers (Anthropic) or ensure consistent structure (OpenAI)
- [ ] Monitor cache hit rates
- [ ] A/B test cost before/after

## The Bottom Line

Prompt caching is free money for AI agents with repeated context. It requires:

1. Structured prompts (stable prefix, dynamic suffix)
2. Minimum token thresholds
3. Consistent API usage patterns

The implementation is simple. The savings compound quickly.

If you're running an agent that makes more than 50 calls/day with the same system prompt, you should be using prompt caching. Period.

---

**Tech used:** Anthropic Claude API, OpenAI API  
**Difficulty:** Beginner  
**Time to read:** ~4 minutes
