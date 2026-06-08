# Day 11 — Agents, part 1 — the loop & tool design

> [← Day 10](day-10.md) · [All days](README.md) · [Day 12 →](day-12.md)

**Module:** Agents & Tool Use  ·  **Time:** ~2.5–3 hrs

## About this module

### Chapter 5 — Agents & Tool Use: from single call to autonomous loop

**Goal:** Build an **agent** — a model running in a *loop* that chooses tools, acts, observes the
results, and decides its own next step until a task is done. By the end you can build a multi-tool
agent from scratch (no framework), keep it from running away, trace every step, and — just as
important — judge *when an agent is the right tool and when it's over-engineering*.

**Why this matters:** Everything so far has been **one request → one response**. An agent is the
model **driving its own multi-step process**: it decides what to do, *your code* executes it, the
result feeds back into the context, and the model decides again — repeat until done. This is what
powers "just do this task for me" products. It's also where cost, latency, and failure modes
explode: a single bad call is one wrong answer; a bad *loop* is twenty wrong calls, a runaway bill,
and an action taken in the real world you can't undo. The skill on this day is building agents that
are *reliable and bounded*, not just impressive in a demo.

> **Setup assumed:** same as [Chapter 2](02-apis-and-integration.md), plus your [Chapter 4](04-rag.md) `retrieve()` function — we'll wrap it as a
> *tool* the agent can call. From-scratch only: we build the loop in plain Python with the SDKs so
> you can see exactly what every framework is doing under the hood. Frameworks (LangGraph, OpenAI
> Agents SDK, Claude Agent SDK) are named as pointers at the end, not taught.

---

## Part A — What an agent actually is (and isn't)

**1. An agent is a loop, not a clever prompt.**
The whole idea fits in five steps the model repeats on its own:
**think → act (call a tool) → observe (get the result) → think again → … → stop.**
The model never executes anything itself — on each turn it *requests* a tool call ("search the docs
for X"), **your code runs it** and feeds the result back, and the model uses that to decide its next
move. The loop ends when the model decides it's done (it answers instead of calling a tool) or you
cut it off (a limit). That "decide its own next step" is the entire difference from [Chapter 2](02-apis-and-integration.md): there,
*you* wrote the sequence of calls; here, *the model* chooses the sequence at runtime.
- *Build consequence:* An agent is mostly a `while` loop around the tool-calling you already learned
  in [Chapter 2](02-apis-and-integration.md). There's no magic — once you see the loop, every agent product is a variation on it.

**2. Agent vs. workflow vs. single call — the real distinction is *who decides the control flow*.**
- **Single call:** one prompt, one answer. *You* did all the thinking about steps. (Most tasks.)
- **Workflow (fixed pipeline):** a *predetermined* sequence of calls you wired up — e.g.
  "summarize → translate → format." The path is fixed; *you* control flow, the model just fills each
  step. Predictable, cheap, easy to test.
- **Agent:** the model chooses the path *at runtime* — which tool, in what order, how many steps,
  when to stop. Flexible, but non-deterministic and harder to bound.
- *Build consequence:* "Agent" is not a prestige upgrade — it's a tradeoff. You buy flexibility with
  unpredictability, cost, and latency. Reach for it only when the path genuinely *can't* be known
  ahead of time.

**3. When you actually need an agent — and when you don't.**
- **Use an agent** when the steps depend on intermediate results you can't predict: open-ended
  research ("find out X and tell me"), tasks whose length varies per input, or anything needing
  dynamic tool choice.
- **Use a fixed workflow** when the steps are knowable in advance — it's cheaper, faster,
  deterministic, and far easier to test. *Most "agent" use cases are really workflows in disguise.*
- **Use a single call** when one prompt does it. Don't build a loop to answer a question.
- *Build consequence:* The senior move is to **push work down the ladder**: agent → workflow →
  single call, choosing the *least* autonomous thing that solves the problem. Every step up the
  ladder costs you predictability and money. Start by asking "can this be a fixed pipeline?" and only
  build an agent when the honest answer is no.

---

## Part B — The agent loop, built from scratch

**4. Tool calling, recapped — now in a loop instead of one-shot.**
[Chapter 2](02-apis-and-integration.md) Part D taught the 5-step round trip: you declare tools → the model requests one → your code
runs it → you return the result → the model uses it. That was *one* round trip. An agent just
**keeps doing round trips until the model stops requesting tools.** The model's signal is the
stop/finish reason: if it returned a tool-use request, run the tool and loop; if it returned a
normal text answer, you're done.
- *Build consequence:* If [Chapter 2](02-apis-and-integration.md)'s tool calling is solid, the agent loop is a small step. If it's
  shaky, go re-do [Chapter 2](02-apis-and-integration.md) Part D first — the loop multiplies every weakness in your tool handling.

**5. The control loop in code — this is the whole engine.**
```python
# Anthropic — the agent loop. (OpenAI shape is identical; see note below.)
def run_agent(question, tools, tool_impls, max_steps=8):
    messages = [{"role": "user", "content": question}]
    for step in range(max_steps):                      # <-- the loop limit (concept 15)
        resp = client.messages.create(
            model="claude-sonnet-4-6", max_tokens=1024,
            system="You are a research assistant. Use tools to find facts before answering. "
                   "When you have enough information, answer directly and cite sources.",
            tools=tools,                               # the toolbox (concept 6)
            messages=messages,
        )
        messages.append({"role": "assistant", "content": resp.content})

        if resp.stop_reason != "tool_use":             # <-- STOP: model answered instead of acting
            return final_text(resp)

        # model requested one or more tools — run them, feed results back
        results = []
        for block in resp.content:
            if block.type == "tool_use":
                output = tool_impls[block.name](**block.input)   # YOUR code executes
                results.append({"type": "tool_result",
                                "tool_use_id": block.id, "content": str(output)})
        messages.append({"role": "user", "content": results})    # observation → next think
    return "Stopped: hit step limit without finishing."          # <-- bounded failure, not infinite
```
The three things that *are* the agent: the **toolbox** (`tools`), the **stop condition**
(`stop_reason != "tool_use"`), and the **limit** (`max_steps`). Everything else is plumbing.
> **OpenAI difference:** same loop; the model's tool requests come back as
> `resp.choices[0].message.tool_calls`, you append a
> `{"role":"tool", "tool_call_id":..., "content":...}` message per result, and the stop signal is
> `finish_reason == "tool_calls"`. The *shape* of the loop is identical — only the field names
> differ (the same [Chapter 2](02-apis-and-integration.md) Part A/D differences).

- *Build consequence:* Notice the context grows every iteration (you append the assistant turn *and*
  the tool results). That growth is the source of Session 2's cost and "lost the plot" problems — keep
  it in view from the first loop you write.

**6. Multi-tool: give the model a toolbox and let it choose.**
A real agent has several tools and decides *which* to use (and in what order) per step. You pass a
list of tool declarations; the model picks. For our deliverable, the toolbox is: your **`retrieve()`
from [Chapter 4](04-rag.md)** (search the docs), a **calculator** (models are unreliable at arithmetic — give them
a real one), and optionally a **web/lookup** tool. The model might search the docs, do a calculation
on what it found, then answer — choosing that sequence itself.
- *Build consequence:* The model's tool *choice* is only as good as your tool *descriptions* — which
  is exactly why [Part C](#part-c--designing-good-tools-the-real-determinant-of-agent-quality) is the part that actually determines whether your agent works.

---

## Part C — Designing good tools (the real determinant of agent quality)

**7. A tool is an API *written for the model* — its description is prompt engineering.**
The model decides whether and how to call a tool **entirely from the tool's name, description, and
parameter schema.** It can't read your implementation. So those fields *are* a prompt, and [Chapter 3](03-prompt-engineering.md)
rules apply: be specific, say what it does, when to use it, when *not* to, and document every
parameter. `search_docs(query: str) — "Search the internal knowledge base for relevant passages.
Use for any factual question about company policy. Returns the top passages with sources."` beats
`search(q)` every time.
- *Build consequence:* When an agent uses the wrong tool, calls it with bad arguments, or ignores a
  tool it should've used, **the bug is almost always a vague tool description**, not a dumb model.
  Fix the description before anything else — it's the highest-leverage knob in agent building (the
  way chunking was for RAG).

**8. Return errors the model can *recover from* — don't crash the loop.**
Tools fail: bad arguments, no results, an API timeout. The agent's superpower is that it can *react*
to a failure — but only if you hand it back a readable message instead of throwing. Return
`"Error: no documents matched 'X'. Try broader search terms."` and the model will rephrase and
retry. Let the exception bubble up and the whole loop dies.
- *Build consequence:* Treat tool error messages as *instructions to the model*, not log lines. A
  good error tells the model how to fix its next attempt. This single habit separates agents that
  self-correct from agents that fall over.

**9. Right-size your tools — not too granular, not too monolithic.**
- **Too granular** (`open_file`, `read_line`, `close_file`): the model wastes many steps — and many
  model calls — orchestrating trivia, and is more likely to lose its way.
- **Too monolithic** (`do_everything(request)`): you've just hidden the real logic inside one tool
  and removed the model's ability to compose — that's a workflow wearing an agent costume.
- **The sweet spot:** tools at the granularity of a *meaningful action* a competent human would name
  — `search_docs`, `calculate`, `send_email`. Each does one comprehensible thing.
- *Build consequence:* Fewer, well-chosen tools beat a sprawling toolbox. Every extra tool is another
  description competing for the model's attention and another way for it to choose wrong. Add tools
  when a capability is genuinely missing, not preemptively.

---

## Interview drill (self-test)
Real Gen-AI interview questions on today's material — *answer from memory, then check against what you just read.*

1. What is agentic AI?
2. What are AI agents?
3. What is ReAct prompting?
4. What are agent frameworks?
5. What are agentic workflows?
6. What are agentic frameworks?
7. What is function calling in LLMs?
8. Explain Cursor AI's agentic flow.
9. What is tool calling in LangChain?
10. What are agentic AI framework tools?
11. What is a multi-hop agentic workflow?
12. What are multiple agentic approaches?
13. What are the frameworks for agentic AI?
14. What agent is used in your architecture?
15. How long did it take to build this agent?

_(90 matched; 15 shown — more in [`ai-eng-questions.md`](../ai-eng-questions.md).)_

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
