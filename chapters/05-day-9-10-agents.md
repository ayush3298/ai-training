## Day 9–10 — Agents & Tool Use: from single call to autonomous loop

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

> **Setup assumed:** same as Day 3, plus your Day 7–8 `retrieve()` function — we'll wrap it as a
> *tool* the agent can call. From-scratch only: we build the loop in plain Python with the SDKs so
> you can see exactly what every framework is doing under the hood. Frameworks (LangGraph, OpenAI
> Agents SDK, Claude Agent SDK) are named as pointers at the end, not taught.

**Suggested split:** Day 9 = Parts A–C (what an agent is, the loop from scratch, designing tools —
build a working single-tool then multi-tool agent). Day 10 = Parts D–F (memory/context, planning
patterns, production reality — make it robust, bounded, and observable).

---

## Part A — What an agent actually is (and isn't)

**1. An agent is a loop, not a clever prompt.**
The whole idea fits in five steps the model repeats on its own:
**think → act (call a tool) → observe (get the result) → think again → … → stop.**
The model never executes anything itself — on each turn it *requests* a tool call ("search the docs
for X"), **your code runs it** and feeds the result back, and the model uses that to decide its next
move. The loop ends when the model decides it's done (it answers instead of calling a tool) or you
cut it off (a limit). That "decide its own next step" is the entire difference from Day 3: there,
*you* wrote the sequence of calls; here, *the model* chooses the sequence at runtime.
- *Build consequence:* An agent is mostly a `while` loop around the tool-calling you already learned
  on Day 3. There's no magic — once you see the loop, every agent product is a variation on it.

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
Day 3 Part D taught the 5-step round trip: you declare tools → the model requests one → your code
runs it → you return the result → the model uses it. That was *one* round trip. An agent just
**keeps doing round trips until the model stops requesting tools.** The model's signal is the
stop/finish reason: if it returned a tool-use request, run the tool and loop; if it returned a
normal text answer, you're done.
- *Build consequence:* If Day 3's tool calling is solid, the agent loop is a small step. If it's
  shaky, go re-do Day 3 Part D first — the loop multiplies every weakness in your tool handling.

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
> differ (the same Day 3 Part A/D differences).

- *Build consequence:* Notice the context grows every iteration (you append the assistant turn *and*
  the tool results). That growth is the source of Day 10's cost and "lost the plot" problems — keep
  it in view from the first loop you write.

**6. Multi-tool: give the model a toolbox and let it choose.**
A real agent has several tools and decides *which* to use (and in what order) per step. You pass a
list of tool declarations; the model picks. For our deliverable, the toolbox is: your **`retrieve()`
from Day 7–8** (search the docs), a **calculator** (models are unreliable at arithmetic — give them
a real one), and optionally a **web/lookup** tool. The model might search the docs, do a calculation
on what it found, then answer — choosing that sequence itself.
- *Build consequence:* The model's tool *choice* is only as good as your tool *descriptions* — which
  is exactly why Part C is the part that actually determines whether your agent works.

---

## Part C — Designing good tools (the real determinant of agent quality)

**7. A tool is an API *written for the model* — its description is prompt engineering.**
The model decides whether and how to call a tool **entirely from the tool's name, description, and
parameter schema.** It can't read your implementation. So those fields *are* a prompt, and Day 5–6
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

## Part D — Memory, context & state across steps

**10. Context grows every single step — and that's your main enemy in a long loop.**
Each iteration appends the model's turn *and* the tool result to `messages`, and you resend the
whole thing next step (Day 3's statelessness, now compounding). A 15-step agent can be resending
tens of thousands of tokens by the end. Two consequences, both from Day 2/Day 3: **cost climbs
super-linearly** across a run, and eventually you **approach the context-window ceiling** and must
trim.
- *Build consequence:* Cost and latency for an agent scale with *steps × growing-context*, not with
  one call. Always print per-run token usage and step count — an agent that quietly takes 20 steps is
  a quietly enormous bill.

**11. Short-term scratchpad vs. long-term memory.**
- **Short-term (working memory):** the `messages` list itself — the running transcript of this task.
  It *is* the agent's scratchpad; it's how it "remembers" what it already tried two steps ago.
- **Long-term memory:** anything that must persist *across* runs or outlive the window — past
  conversations, learned facts, user preferences. The standard implementation is… RAG (Day 7–8):
  write memories to a store, retrieve the relevant ones into context when needed. *Long-term agent
  memory is just retrieval pointed at the agent's own history.*
- *Build consequence:* You don't get memory for free — short-term is the message list *you* manage,
  long-term is a retrieval system *you* build. "The agent forgot" is always a bug in one of those
  two, never a model feature you forgot to enable.

**12. Compaction: summarize old steps so a long loop doesn't drown.**
When the transcript grows large, replace the oldest steps with a summary — "so far: searched docs
for X, found Y, computed Z" — keeping recent turns verbatim. This caps context growth and, as a
bonus, *refocuses* the model on what matters. It's the agent version of Day 3's "trim or summarize
older turns."
- *Build consequence:* For any agent that runs more than a handful of steps, plan for compaction from
  the start. Without it, long tasks get more expensive *and* less accurate as they go — exactly the
  wrong direction.

---

## Part E — Planning & multi-step reasoning patterns

**13. The patterns that make agents reliable — pick by task, don't cargo-cult.**
- **ReAct (reason + act):** the model writes its reasoning *then* acts, each step — "I need the
  refund window, so I'll search docs" → searches → "found it, now I'll answer." This is the natural
  default and basically what our loop already does. Interleaving thought and action keeps it grounded.
- **Plan-then-execute:** for complex tasks, have the model write the *whole plan* up front, then
  execute the steps. Gives you a plan you can inspect (or approve) before any action runs — valuable
  when steps have side effects.
- **Reflection / self-correction:** after producing a result, the model critiques its own work and
  revises ("does this actually answer the question? what's missing?"). Catches mistakes a single
  forward pass misses — at the cost of extra calls.
- **Decomposition:** break a big task into sub-tasks, solve each, combine. Often a *workflow* around
  small agent/LLM calls — and usually better than one mega-agent.
- *Build consequence:* These cost extra model calls, so add them where they earn their keep
  (reflection on high-stakes output, planning where side effects are dangerous) — not everywhere.

**14. Single agent vs. multi-agent — and why multi-agent is usually premature.**
The hype is "swarms of agents talking to each other." The reality: multi-agent systems multiply
cost, latency, and failure surface, and coordination is genuinely hard. **A single well-equipped
agent with good tools beats a committee of agents for almost everything you'll build early on.**
Multi-agent earns its place only when sub-tasks are truly independent and parallelizable, or need
genuinely different specialized contexts.
- *Build consequence:* Default to one agent with a good toolbox. Reach for multi-agent only when you
  can name the specific problem one agent *can't* solve — not because it sounds powerful.

---

## Part F — Production reality: cost, latency, safety, failure

**15. Runaway loops are the signature agent failure — bound everything.**
An agent can loop forever: call a tool, get a confusing result, call it again, repeat. Three
guardrails, always:
- **A hard step limit** (`max_steps`) — the backstop in our loop. Non-negotiable.
- **A token / cost budget** — track cumulative usage across the run and abort past a ceiling.
- **Loop detection** — if the model calls the same tool with the same arguments twice running,
  intervene (it's stuck).
- *Build consequence:* Never ship an agent without a hard upper bound on steps *and* spend. "It
  usually finishes in 3 steps" is not a bound; the one time it doesn't is a runaway bill or a hammered
  downstream API.

**16. Guardrails: the agent can take *actions*, so gate the dangerous ones.**
A read-only agent (search, calculate) is low-risk. The moment a tool *changes the world* — sends
email, writes a DB, spends money, deletes files — the stakes change:
- **Human-in-the-loop:** require explicit approval before consequential actions (the agent
  *proposes*, a human *confirms*). This mirrors your own working rule — show the draft, get
  confirmation before sending.
- **Least-privilege tools:** give each tool the *minimum* permission it needs; don't hand an agent
  broad write access "just in case."
- **Sandboxing & validation:** validate tool arguments before executing; run risky operations in a
  contained environment.
- *Build consequence:* Separate **read** tools (safe to let the agent fire freely) from
  **write/irreversible** tools (gate behind confirmation). The blast radius of an agent equals the
  power of its most dangerous tool — design as if it *will* eventually call that tool wrong.

**17. Observability: you cannot debug what you cannot see.**
An agent's behavior is a *trajectory* — the sequence of thoughts, tool calls, arguments, and results
across steps. When it does something wrong, "the output was bad" tells you nothing; you need the
*trace* to find which step went off the rails. Log every step: what the model decided, which tool it
called with what arguments, what came back.
- *Build consequence:* Build step tracing *before* you need it — it's the only way to debug a loop.
  The same trace is also how you'll *evaluate* the agent (Section 7): grade the trajectory, not just
  the final answer.

**18. Know when to *not* let the agent decide.**
The recurring theme of this whole day: autonomy is a cost, not a goal. If a step is predictable,
hard-code it. If an action is dangerous, gate it. If the whole task is knowable, it's a workflow, not
an agent. The best agent systems are *mostly* deterministic scaffolding with the model making
decisions only at the genuinely open points.
- *Build consequence:* Maximize the deterministic parts, minimize the autonomous ones. You're not
  building "an AI that does anything" — you're building a bounded system that uses a model's judgment
  exactly where judgment is actually required.

---

## Part G — Managed agent runtimes (cloud-native agents)

Everything so far had *you* own the loop: your `while`, your `messages` list, your `max_steps`, your
tracing. A managed agent runtime is the cloud running that exact loop **server-side** — you supply
the model, the tools, and (optionally) a knowledge base, and the provider executes think→act→observe
on its own infrastructure. You give up control of the loop in exchange for not operating it. This
Part is about *that trade* and how the **invocation** differs from the Day-9 hand-built loop. (The
broader platform/pricing/region picture lives in the Cloud Managed-GenAI Platforms extension
chapter; here we stay on the agent *runtime* decision.)

**19. When a managed runtime beats your hand-built loop.**
Your from-scratch loop wins when you need to *see and shape* every step — custom stop logic, bespoke
compaction, weird tool orchestration, full control of the trajectory. A managed runtime wins when the
loop itself is undifferentiated and the *hard parts are operational*: you need IAM-scoped tool
permissions, VPC-private tool calls, audit logs, autoscaling, session isolation, and long-running
executions — and you'd rather not build and run all of that. The agent's *logic* is the same loop
from Part B; what you're really choosing is **who operates it**.
- *Build consequence:* Pick managed when the value is in the *governance and ops* (auth, audit,
  scaling, isolation) and the loop is standard reason-act. Keep your own loop when the *loop itself*
  is the product — non-standard control flow, custom guardrails, or step-level behavior you can't
  express in a provider's schema.

**20. AWS Bedrock Agents — the loop runs server-side behind one call.**
You configure an agent from three declared pieces: a **foundation model**, **action groups** (your
tools, backed by a Lambda or an OpenAPI/API endpoint — the managed equivalent of concept 6's
toolbox), and a **Knowledge Base** (managed RAG over your data — Day 7–8's `retrieve()`, run by AWS).
AWS then executes the *entire* reason-act tool loop for you: the model picks an action group, Bedrock
invokes your Lambda, feeds the result back, queries the Knowledge Base when needed, and loops until
done. Your code shrinks to **one `InvokeAgent` call** — no `while`, no `messages` bookkeeping, no
stop-reason check. Above the turnkey "Agents" product sits the modular **AgentCore** tier:
separately-consumable primitives — **Runtime** (serverless execution, up to ~8-hour runs, per-session
isolation), **Memory**, **Gateway** (turns existing APIs into agent tools), **Identity**, and
**Observability** — that you compose into your own runtime. The choice: **managed Agents** when you
want the whole loop turnkey; **AgentCore** when you want the building blocks and your own assembly.
- *Build consequence:* With Bedrock Agents, concepts 5, 10, 12, and 17 (the loop, context growth,
  compaction, tracing) move *inside AWS* — convenient, but you observe the trajectory through
  Bedrock/CloudWatch traces, not your own logs, and you bound it with AWS config, not your `max_steps`.
  Reach for AgentCore the moment the turnkey loop's schema can't express a control or guardrail you
  need (concept 16).

**21. GCP Vertex Agent Engine — managed runtime, ADK as the deploy target.**
On Google Cloud the managed runtime is **Vertex AI Agent Engine**: it hosts your agent and provides
the operational layer — **autoscaling**, managed **sessions/memory**, and **long-running**
executions — so you don't run the serving loop yourself. You author the agent in a code-first
framework (Google's **ADK**) and **deploy it onto Agent Engine** as the target; Agent Engine is the
runtime, ADK is just how the agent is defined. (We don't teach orchestration frameworks here — ADK
matters only as the thing Agent Engine runs.)
- *Build consequence:* Same trade as Bedrock, different vendor: you hand sessions, memory, and scaling
  to Google and invoke a deployed endpoint instead of driving your own loop. Evaluate it on the *ops*
  it removes (sessions, autoscale, long-running) versus the lock-in of authoring against its runtime.

**22. Azure Copilot Studio — low/no-code agents with model-choice-per-use-case (BYOM).**
Copilot Studio is the **low/no-code** end of the spectrum: you build an agent in a visual builder —
topics, tools, knowledge — with little or no loop code at all. Its agent-building lever is **BYOM
("bring your own model")**: you wire any **Azure AI Foundry** model to a given prompt/agent, and you
can **swap the model without rebuilding the agent** — cheaper model for routine prompts, a stronger
one where it matters. That's *model choice per use case* expressed as configuration, not code.
- *Build consequence:* Copilot Studio trades the most control for the least code — right for
  business/internal agents and fast iteration, wrong when you need step-level control of the loop.
  BYOM means your Day-3 "pick the model per task" decision becomes a per-prompt setting you can change
  without touching the agent's wiring.

**23. The runtime tradeoff axis — managed vs. your own loop.**
Same axis that runs through this whole day, applied to *who owns the loop*:
- **Managed runtime (Bedrock Agents / Vertex Agent Engine / Copilot Studio):** far less code, and
  **IAM, VPC, audit, scaling, and session isolation are baked in**. You pay in **schema lock-in** (the
  agent must fit the provider's action-group/tool/knowledge model) and **less control** over the loop,
  stop logic, and trajectory.
- **Your own loop (Part B):** total control and portability across providers; you operate everything —
  bounds, tracing, memory, auth — yourself.
- **Invocation difference:** the Day-9 loop is *your process* running many model calls and tool
  executions locally; a managed runtime is a **single remote call** (`InvokeAgent` / a deployed Agent
  Engine endpoint / a Copilot Studio invocation) where the provider runs the steps and returns the
  result — your `while`, `stop_reason`, and tool execution all happen on their side.
- *Build consequence:* Choose by where your differentiation is. If it's governance and operating an
  agent safely at scale, managed wins. If it's the loop's behavior itself, keep Part B. Either way the
  *concepts* (tools as prompts, recoverable errors, bounds, observability, gating writes) are
  unchanged — managed runtimes implement them *for* you; they don't remove them.

---

**Resources**
- Anthropic — *Building Effective Agents* (the agent-vs-workflow framing, and why simpler is usually
  better); tool-use / agent docs.
- AWS — Bedrock Agents (action groups, Knowledge Bases, `InvokeAgent`) and AgentCore (Runtime / Memory
  / Gateway / Identity / Observability). GCP — Vertex AI Agent Engine (managed runtime) + ADK as the
  deploy target. Azure — Copilot Studio (low/no-code) + Azure AI Foundry models (BYOM).
- The **Cloud Managed-GenAI Platforms** extension chapter — the broader platform, cost, and region
  picture for all three clouds (this Part covers only the agent-*runtime* decision).
- OpenAI — function-calling guide and the Agents SDK overview (read for the loop concepts; we build
  raw).
- Your own Day 3 (Part D, tool calling) and Day 7–8 (`retrieve()`) — direct prerequisites; the
  deliverable reuses both.

**Hands-on tasks**
1. **One-tool loop:** wrap your Day 7–8 `retrieve()` as a `search_docs` tool and put it in the Part B
   loop. Ask a question that needs one search; print the messages list after each step and watch the
   loop run, then stop.
2. **Watch it stop:** add a `print(step, resp.stop_reason)` each iteration. Confirm exactly when
   `stop_reason` flips from `tool_use` to a final answer.
3. **Add a second tool:** add a `calculate(expr)` tool. Ask a question needing *both* (e.g. "what's
   our refund window in hours?") and confirm the agent searches *then* calculates — its own choice of
   order.
4. **Break a description:** rename `search_docs` to `search` and gut its description. Observe the
   agent misusing or ignoring it. Restore the good description; note the difference. (Concept 7, felt.)
5. **Recoverable error:** make `search_docs` return `"Error: no matches, try broader terms"` on a
   miss. Ask something not in the corpus and watch whether the agent retries or gives up.
6. **Bound it:** set `max_steps=3` and ask a question needing more. Confirm it stops cleanly with the
   bounded-failure message instead of looping. Then log cumulative token usage per run.
7. **Trace it:** log every step (decision, tool, args, result) to a list and print the full trajectory
   at the end. This is your debugging + eval artifact.
8. *(Stretch)* **Compaction:** after step 5, summarize the oldest steps into one message and continue.
   Compare total tokens with and without.
9. *(Stretch)* **Managed runtime, same agent:** re-implement your hand-built agent as a **Bedrock
   Agent** (or **Vertex Agent Engine**): define *one* tool (an action group / tool wrapping your
   `search_docs`) and a **Knowledge Base** over the Day 7–8 corpus, then invoke it with a single
   `InvokeAgent` (or deployed-endpoint) call on the same question. Measure three things against the
   Part B loop: **lines of code** (yours vs. the managed config + one call), **latency** (your local
   multi-call loop vs. the single remote call), and **flexibility** (what step-level control you lose).
   Write **3 bullets**: when you'd pick the hand-built loop, when managed Agents, and when AgentCore /
   the modular tier (concepts 19–23).

**Questions**

*Check understanding*
1. State the agent loop in its five steps. Who executes the tool — the model or your code?
2. What's the precise difference between an agent and a fixed workflow?
3. In the loop, what single signal tells you the agent is *done* vs. wants to act again?
4. Why does an agent's context grow every step, and what two problems does that cause?
5. Of name, description, and schema — why is the tool *description* called prompt engineering?
6. What's the difference between an agent's short-term and long-term memory, and how is long-term
   usually implemented?
7. Name the three guardrails that prevent a runaway loop.

*Apply it*
8. Your agent keeps picking the wrong tool. What do you fix *first*, and why?
9. A tool hits a timeout. Why is returning `"Error: timed out, retry"` better than letting the
   exception propagate?
10. You're asked to build "an agent" that always does: fetch order → check status → email the
    customer. Is an agent the right choice? What would you build instead, and why?
11. Your agent runs 18 steps and the bill is huge. Name two mechanisms to bring this under control.
12. The agent has a `delete_records` tool. What guardrail must wrap it, and how does that differ from
    how you treat `search_docs`?

*Stretch*
13. "We should use 5 specialized agents instead of 1." Argue the default position and name the
    conditions under which multi-agent is actually justified.
14. Your agent gives a wrong final answer. Walk through how you'd use the step trace to localize the
    failure, and why the final answer alone is not enough.
15. Explain the "push work down the ladder" principle (agent → workflow → single call) and why each
    step up costs you something. Give one task that belongs at each rung.

**Answer key**
1. think → act (call a tool) → observe (result) → think again → stop. *Your code* executes the tool;
   the model only *requests* it.
2. A workflow's control flow is fixed by *you* in advance; an agent's control flow (which tool, what
   order, how many steps, when to stop) is chosen by the *model at runtime*.
3. The stop/finish reason: `stop_reason != "tool_use"` (Anthropic) / `finish_reason != "tool_calls"`
   (OpenAI) — i.e. the model returned a text answer instead of a tool request.
4. Each step appends the assistant turn + tool result and the whole list is resent. Problems: cost
   climbs super-linearly across the run, and you approach the context-window ceiling.
5. The model decides whether/how to call a tool *solely* from its name/description/schema — it can't
   see your code — so those fields function exactly like a prompt and follow Day 5–6 rules.
6. Short-term = the running `messages` transcript for this task (the scratchpad); long-term =
   persistence across runs, usually implemented as RAG over the agent's own history/facts.
7. A hard step limit (`max_steps`), a token/cost budget with abort, and loop/repeat detection.
8. The tool description (concept 7) — wrong-tool / bad-argument behavior is almost always a vague
   description, not a weak model. Cheapest, highest-leverage fix.
9. A readable error lets the model *react* and retry with a fix; an uncaught exception kills the whole
   loop, wasting all prior steps. Errors are instructions to the model.
10. No — the steps are fully known in advance, so it's a *workflow*: a fixed three-call pipeline. It's
    cheaper, deterministic, and easy to test; an agent would add cost and unpredictability for zero
    benefit.
11. A hard `max_steps` limit and a cumulative token/cost budget that aborts the run (plus loop
    detection if it's repeating a call).
12. `delete_records` is irreversible → human-in-the-loop approval before it runs, least-privilege
    scope, and argument validation. `search_docs` is read-only/safe, so the agent may fire it freely.
13. Default to one well-equipped agent: multi-agent multiplies cost, latency, and coordination
    failure. It's justified only when sub-tasks are genuinely independent/parallelizable or need
    distinct specialized contexts you can't combine in one.
14. Replay the trajectory step by step: find the step where the decision, tool choice, arguments, or
    returned result first went wrong (e.g. retrieval returned junk at step 2). The final answer alone
    hides *which* of N steps failed; the trace localizes it — same "debug retrieval before generation"
    logic, generalized.
15. Use the *least* autonomous tool that solves the task; each rung up (single→workflow→agent) trades
    away predictability, cost, and testability for flexibility. Single call: "summarize this text."
    Workflow: "fetch → check → email" (fixed steps). Agent: "research this open question and report,"
    where steps depend on what's found.

**Deliverable:** a working **multi-tool RAG research agent**, built from scratch (no framework): the
Day 7–8 `retrieve()` wrapped as `search_docs`, plus a `calculate` tool, running in a bounded loop
(`max_steps` + token budget). It must (a) choose tools itself and answer a question needing both
search and calculation, (b) recover from a tool error instead of crashing, (c) stop cleanly at the
limit, and (d) emit a full step trace. Include a short writeup: for one run, *when and why* did the
agent decide to stop?

**Daily update (DM to Ayush):** one line — what you built/learned and any blocker (e.g. "multi-tool
agent works: searches docs then calculates, recovers from empty-result errors, capped at 8 steps;
trace logging in").

**Time:** ~2 days. Day 9: Parts A–C (loop + tool design, get a multi-tool agent running). Day 10:
Parts D–F (memory/compaction, planning patterns, bounds/guardrails/tracing).

---

