---

layout:  post

title:  "Building a session history manager for LLM agents"

categories:  AI

author:  "Aalok Singh"

---

How to build a production-grade session history manager for LLM agents

---

## Where This Comes From

I've been building an agentic solution that involves multiple sub-agents orchestrated together - think a main planning agent that delegates to specialised sub-agents for RAG retrieval, tool execution, and code generation. Each sub-agent adds its own system prompt, tool schemas, retrieved documents, and generated code blocks to the conversation. A single user request can fan out into dozens of internal messages before a final answer surfaces.

It worked beautifully in demos. Then real users started having *actual* conversations.

Within 15–20 turns, the context window was full. Users started seeing this in production:

> `context limit exceeded`

It was because every tool call, every chunk of RAG context, every block of generated code, and every sub-agent handoff was silently piling up in the session history.

I took a step back and built a dedicated context management engine to solve this for good. This post captures the key learnings, common pitfalls, and practical patterns I discovered.

---

## The Problem (and Why "400K Context" Is Misleading)

Let's take the model we're actually using: **GPT-5.3-codex**. It advertises a **400K total context length**. Sounds massive but here's the breakdown:

| Spec | Tokens |
|---|---|
| **Input** context limit | 272,000 |
| **Output** (max completion) | 128,000 |
| **Total** (input + output) | 400,000 |

That 400K headline number is **not** what you get to play with. The output budget is reserved for the model's response. **Your session history, system prompt, tool schemas, RAG context - all of it must fit inside the 272K *input* limit.** That's the number that matters for session management.

And 272K still sounds like a lot - until you're running a multi-agent pipeline. Here's what a typical turn looks like under the hood:

| Message | Role | Approx. tokens |
|---|---|---|
| User asks a question | `user` | ~50 |
| Planning agent reasons about which sub-agent to call | `assistant` | ~200 |
| RAG sub-agent retrieves 3 document chunks | `tool` | ~2,000 |
| Code-gen sub-agent produces a solution | `assistant` | ~1,500 |
| Tool call to execute/validate the code | `tool` | ~500 |
| Final synthesised answer | `assistant` | ~300 |

That's **~4,500 tokens for a single turn**. Multiply by 40-50 turns in an extended working session and you've blown past 200K tokens without the user typing more than a few sentences. The context doesn't grow linearly with conversation length; it **balloons** because of the compound effect of sub-agents, RAG payloads, and generated code.

---

## Pitfall #1: Ignoring the Hidden Token Consumers

Most developers look at the spec sheet - "400K context!" - and assume they have that much room for chat. **You don't.** The input limit is 272K, and even that is eroded by things you never see in the chat:

| Hidden consumer | Typical cost |
|---|---|
| System prompt | 1000 - 1500 tokens |
| Tool/function schemas (multiple agents) | 1,000 - 5,000 tokens |
| Reserved output tokens | up to 128,000 tokens |

With GPT-5.3-codex, if you reserve even a modest 16K for output and burn 3K on tool schemas across your sub-agents, **your real budget is ~252K**, not 272K.

```python
# The ACTUAL budget formula
input_context_limit = 272_000           
system_tokens       = count_tokens(system_prompt)
reserved            = max_output_tokens + tool_schema_overhead
remaining           = input_context_limit - system_tokens - reserved
budget              = int(remaining * safety_fraction)  # 0.80 recommended
```

**Lesson:** Always compute your budget against the **input** context limit, not the total. And never assume the full input window is yours either - subtract system prompts, tool schemas, and output reservations first.

---

## Pitfall #2: Trimming at the Wrong Boundary

The naive approach is to just drop the oldest messages until you're under budget. But chat histories aren't a flat list of independent messages - they contain **logical groups** that must stay together:

```
user: "What's the weather in Paris?"
assistant: [tool_call: get_weather("Paris")]    ← function call
tool: "22°C and sunny"                         ← function result
assistant: "It's 22°C and sunny in Paris!"     ← final reply
```

If you trim between the tool call and the tool result, the model sees an orphaned function call with no response - and it hallucinates or errors out.

**The fix: align your trim boundary to the next `role == "user"` message.** This guarantees you never split a tool-call sequence:

```python
# Walk forward to the next "user" message
aligned = None
for j in range(keep_from, len(items)):
    if items[j].get("role") == "user":
        aligned = j
        break

if aligned is None:
    # No safe boundary exists - nuke the session
    await session.clear_session()
    return

keep_from = aligned
```

**Lesson:** Never trim mid-turn. Always snap to a role boundary.

---

## Pitfall #3: Losing Context Silently

Trimming old messages solves the token problem - but destroys context. If the user mentioned their name, a project requirement, or a design decision 40 messages ago, that information is just *gone*.

The fix is to **summarize before you trim**:

```python
conversation_text = "\n".join(
    f"{item['role']}: {item['content']}" for item in trimmed_items
)

response = await client.chat.completions.create(
    model=summary_model,
    messages=[{
        "role": "user",
        "content": (
            "Summarise this conversation excerpt in 2-4 sentences, "
            "preserving all key facts, decisions, and information "
            "the user may refer back to.\n\n"
            f"{conversation_text}"
        ),
    }],
    max_tokens=300,
    temperature=0.3,
)

summary = response.choices[0].message.content.strip()
```

Then inject the summary back into the session as the very first message:

```python
await session.clear_session()

new_items = [{
    "role": "user",
    "content": f"[Previous conversation summary]: {summary}",
}]
new_items.extend(kept_items)

await session.add_items(new_items)
```

Your system prompt should explicitly tell the model to trust these summaries:

```
When you see a '[Previous conversation summary]' message,
treat it as a reliable recap of the earlier conversation.
Use it to maintain continuity.
```

**Lesson:** Trim the tokens, but compress the knowledge. A 3-sentence summary is worth 10,000 trimmed tokens.

---

## Pitfall #4: Letting Summarization Failures Kill the Session

Summarization uses an LLM call, and LLM calls can fail - network issues, rate limits, content filters. If your summarization crashes and you haven't trimmed, the *next* agent call will hit the context limit anyway.

**Always trim first, summarize second, and treat summarization as best-effort:**

```python
summary_text = None
try:
    summary_text = await summarize(trimmed_items)
except Exception:
    logger.exception("Summarization failed - trimming without summary")

# Proceed with or without summary
await rewrite_session(summary_text, kept_items)
```

**Lesson:** Graceful degradation > fragile correctness.

---

## Good Practice: Make the Budget Visible

Once I added a React-based session visualizer connected to the backend via WebSocket, debugging became 10x easier. I could literally *watch* the token budget shrink in real time:

```python
# In your context manager, emit events via a callback:
if event_callback:
    await event_callback({
        "type": "budget_info",
        "context_window": context_window,
        "history_tokens": history_tokens,
        "budget": budget,
    })
```

I'd strongly recommend building a lightweight dashboard when developing agents. Watching the numbers change as you chat makes budget problems obvious *before* they become runtime errors.

---

## Good Practice: Apply a Safety Fraction

Even with precise token counting, there's always a margin of error - tiktoken counts don't perfectly match the model's internal tokenizer, and API overhead adds a few tokens per message.

A simple **0.80× safety fraction** on the remaining budget gives you a comfortable 10% buffer:

```python
budget = int(remaining * 0.80)
```

This single line has prevented more crashes than any other piece of code in the project.

---

## The Complete Algorithm

Here's the full strategy distilled into 9 steps:

1. **Count tokens** for system prompt + full session history
2. **Reserve** output tokens + tool-schema overhead
3. **Apply safety fraction** (0.80) to the remaining budget
4. **Keep the newest messages** that fit within budget
5. **Align trim boundary** forward to the next `role == "user"` message
6. **Summarize** the trimmed messages using an LLM (best-effort)
7. **Inject the summary** as the first message in the rewritten session
8. **Clear session entirely** if budget ≤ 0 or no safe boundary exists
9. **Continue trimming** even if summarization fails

Run this function **before every agent turn**, and your agent will handle conversations of unlimited length without ever hitting the context ceiling.

---

## TL;DR

| Pitfall | Fix |
|---|---|
| Assuming the full context window is available | Subtract system prompt, tool schemas, and output reservation |
| Trimming mid-turn (splitting tool calls) | Snap trim boundary to `role == "user"` |
| Losing important context when trimming | Summarize trimmed messages before discarding |
| Summarization failure crashing the session | Treat summarization as best-effort; always trim |
| Debugging budget issues blindly | Build a real-time visualizer dashboard |

---

If you're building with the OpenAI Agents SDK (or any LLM framework), these patterns will save you from the most common - and most frustrating - production failures.

The full POC code (Python backend + React visualizer) is available on my [GitHub](https://github.com/aalok05/AgentSessionManager).


---

