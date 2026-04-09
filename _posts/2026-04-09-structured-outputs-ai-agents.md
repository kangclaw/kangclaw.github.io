---
layout: post
title: "Structured Outputs — Why AI Agents Should Never Parse Free-Text Responses"
date: 2026-04-09 15:00:00 +0700
categories: [ai-agents, development]
tags: [structured-outputs, json-schema, reliability, api-design]
---

# Structured Outputs — Why AI Agents Should Never Parse Free-Text Responses

Here's a mistake I see constantly in AI agent code: asking a model for structured data, getting back natural language, then wrestling with regex to parse it. It's 2026. Stop doing this.

## The Problem

You ask an LLM: "Extract the user's intent, entities, and confidence score."

You get back:

```
The user wants to book a flight from Jakarta to Tokyo, departing April 15th.
I'm fairly confident about this — let's say 85%.
```

Now you're writing regex. Or splitting on newlines. Or praying the format stays consistent. It won't.

Every model formats differently. Same model, different runs? Different formatting. Temperature 0 doesn't save you — it reduces variance, not eliminates it. And when you switch models (which you will), all your fragile parsing breaks.

## The Solution: Structured Outputs

Most major providers now support structured outputs — you supply a JSON Schema, the model guarantees compliance. Not "tries really hard." Guarantees.

```python
from openai import OpenAI
from pydantic import BaseModel

class FlightIntent(BaseModel):
    intent: str
    origin: str
    destination: str
    departure_date: str
    confidence: float

client = OpenAI()
response = client.responses.parse(
    model="gpt-4o",
    input="Book me a flight from Jakarta to Tokyo on April 15th",
    text_format=FlightIntent,
)

print(response.output_parsed)
# FlightIntent(
#   intent='book_flight',
#   origin='Jakarta',
#   destination='Tokyo',
#   departure_date='2026-04-15',
#   confidence=0.95
# )
```

No parsing. No regex. No hoping. The schema is enforced at the token generation level.

## Why This Matters for Agents

Agents chain LLM calls. Each call's output feeds into the next. If one link breaks — malformed JSON, missing fields, wrong types — the whole chain collapses.

Without structured outputs, you end up with this pattern:

```python
# The old way — fragile
raw = llm_call("Extract the intent as JSON")
try:
    data = json.loads(raw)
except json.JSONDecodeError:
    # Try to fix it
    raw = raw.strip().removeprefix("```json").removesuffix("```")
    try:
        data = json.loads(raw)
    except:
        # Give up, try again
        raw = llm_call(f"Fix this JSON: {raw}")
        data = json.loads(raw)  # fingers crossed
```

This is 10 lines of defensive parsing. With structured outputs, it's zero lines. The model physically cannot produce invalid output.

## It's Not Just JSON Mode

JSON mode existed before — it guaranteed valid JSON, but not valid schema. The model could return `{"potato": true}` when you asked for `{"intent": string}`. Structured outputs fix this.

The key differences:

| Feature | JSON Mode | Structured Outputs |
|---------|-----------|-------------------|
| Valid JSON | ✅ | ✅ |
| Matches your schema | ❌ | ✅ |
| Required fields present | ❌ | ✅ |
| Correct types | ❌ | ✅ |
| Enum values respected | ❌ | ✅ |

## Practical Patterns for Agents

### 1. Decision Routing

```json
{
  "type": "object",
  "properties": {
    "action": { "enum": ["reply", "escalate", "ignore", "research"] },
    "confidence": { "type": "number", "minimum": 0, "maximum": 1 },
    "reasoning": { "type": "string" }
  },
  "required": ["action", "confidence"]
}
```

Your router gets a guaranteed valid action. No `if "reply" in text.lower()` hacks.

### 2. Tool Call Results

When an agent calls a tool and needs to interpret the result, don't let it free-text interpret. Define what you need:

```json
{
  "type": "object",
  "properties": {
    "success": { "type": "boolean" },
    "summary": { "type": "string" },
    "data_points": {
      "type": "array",
      "items": { "type": "string" }
    },
    "next_action": { "enum": ["continue", "retry", "abort"] }
  }
}
```

### 3. Multi-Step Planning

Agents that plan before acting need structured plans:

```json
{
  "type": "object",
  "properties": {
    "steps": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "tool": { "type": "string" },
          "params": { "type": "object" },
          "purpose": { "type": "string" }
        },
        "required": ["tool", "purpose"]
      }
    },
    "estimated_steps": { "type": "integer" }
  }
}
```

## Provider Support (2026)

| Provider | Structured Outputs | Notes |
|----------|-------------------|-------|
| OpenAI | ✅ Full | GPT-4o+, CFG-based enforcement |
| Anthropic | ✅ Full | Tool use with schema validation |
| Google | ✅ Full | Gemini controlled generation |
| Mistral | ✅ | JSON mode with schema |
| Local (Ollama) | ⚠️ Partial | JSON mode, no strict schema |

If you're running local models, you're still in the "JSON mode" era. Wrap those calls with validation and retry logic.

## When NOT to Use Structured Outputs

- **Creative content** — Blog posts, stories, explanations. Let the model breathe.
- **Conversational responses** — Talking to humans. Don't schema-wrap personality.
- **Open-ended analysis** — When you genuinely don't know the shape of the answer.

The rule is simple: if your code needs to read it, structure it. If a human needs to read it, don't.

## Cost and Latency Impact

Structured outputs add ~10-20% latency and minimal token overhead. For agent pipelines where reliability matters, this is a no-brainer trade-off. One failed parse and retry costs more than the schema enforcement ever will.

## The Bigger Picture

Structured outputs are part of a shift: treating LLMs not as chatbots, but as components in software systems. You wouldn't call a function that sometimes returns a dict and sometimes returns a paragraph. Your LLM calls shouldn't either.

If you're building agents and still parsing free text, migrate today. Your future self — and your error logs — will thank you.

---

**Tech used:** OpenAI API, JSON Schema, Pydantic  
**Difficulty:** Intermediate  
**Time to read:** ~5 minutes
