# Day 20 — Architecture, part 1 — compound systems & orchestration patterns

> [← Day 19](day-19.md) · [All days](README.md) · [Day 21 →](day-21.md)

**Module:** Architecture & System Design  ·  **Time:** ~2.5–3 hrs

## About this module

### LLM Application Architecture & System Design: composing reliable systems, not one big prompt

**Goal:** By the end you can take a vague product request ("build an AI feature that does X") and **draw its system architecture**: decide which parts are deterministic *workflow* versus autonomous *agent*, pick the orchestration pattern for each step (chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer), place the cross-cutting layers — a provider-gateway boundary, caching tiers, guardrails, the context/data plane, streaming, observability — as **named components** rather than inline code, and justify every box by the failure it prevents or the cost/latency it buys. You produce one architecture diagram plus a thin "compound system" skeleton that wires your earlier-chapter artifacts (`retrieve()`, the agent loop, `tracked_call`) behind clean interfaces.

**Why this matters:** Each earlier chapter handed you one capability in isolation — a prompt ([Chapter 3](03-prompt-engineering.md)), a retriever ([Chapter 4](04-rag.md)), an agent ([Chapter 5](05-agents.md)), an endpoint ([Chapter 8](08-deployment.md)). Real products are none of those alone; they're a **compound system** of several models, retrievers, tools, and validators wired together. The dominant failure at this stage is "I built one giant prompt (or one all-powerful agent) to do everything" — a thing that is unpredictable, expensive, untestable, and impossible to debug because you can't tell *which sub-task* failed. This is the highest-leverage chapter for shipping real systems: where you learn to decompose the problem, push each piece down to the *least-autonomous* component that solves it, draw the trust and control boundaries, and make the architecture decisions *on paper before* you write the operational code in [Chapter 8](08-deployment.md). The cost of not knowing this is the demo that won't survive contact with the second use case.

> **Setup assumed:** same as before, plus three artifacts you already built: your [Chapter 4](04-rag.md) `retrieve(question) -> chunks`, your [Chapter 5](05-agents.md) agent loop, and the [Chapter 8](08-deployment.md) `tracked_call` logging wrapper. This chapter wires them behind clean interfaces; it does not re-teach them. The work is mostly **boxes-and-arrows on paper** plus thin Python skeletons that name components and their signatures — no new infrastructure beyond what earlier chapters already used (Docker for any local store, secrets from the environment).

---

## Part A — Compound AI systems: the system is the unit, not the prompt

**1. A "compound AI system" is the real unit of design — one big prompt is a 2000-line function with no subroutines.**
Picture the worst function you've ever inherited: two thousand lines, no sub-routines, every concern tangled into one body, untestable because there's no seam to test *at*. That is exactly what "one prompt (or one agent) that classifies *and* retrieves *and* answers *and* checks its own work" is — just expressed in natural language instead of code, which makes it *look* innocent. A **compound AI system** is the same logic *factored*: the identical end-to-end behavior, but broken into named, single-responsibility units — a classifier, a retriever, an answerer, a validator — each with a clean interface, each independently testable, swappable, and observable. Nothing about the *capability* changes; what changes is that the system is now made of parts you can reason about one at a time. The mental shift this chapter asks for is the same one that separates a junior who writes one giant function from a senior who factors it: **you stop designing prompts and start designing systems** — components and the edges between them.
- *Build consequence:* When someone hands you "build an AI feature that does X," your first artifact is not a prompt — it's a **diagram of boxes and arrows**. If your whole design is one prompt, you have a single 2000-line function and every problem below (can't test, can't cache, can't swap, can't tell what failed) is already baked in. Factor first; write code into the boxes second.

**2. Decompose by the cracks: each thing the monolith *can't* do names a box.**
Don't decompose by taste — derive the boxes from where the monolith cracks. Take a concrete, realistic **support assistant**: a user writes in, and the feature is supposed to read the message, find the relevant help-center articles, write a grounded answer, and make sure that answer is safe and on-policy before it goes out. The tempting build is *one* prompt that does all four: *"You are a support agent. Classify the request, search the docs, write an answer grounded in them, and don't say anything off-policy."* It demos beautifully. Then it meets the second use case, and you discover four cracks, each of which is really a missing component:

- **You can't *test* it.** When an answer is wrong, was the failure the classification, the retrieval, the drafting, or the safety check? A single prompt gives you one output and no way to attribute the failure to a stage — exactly [Chapter 7](07-evaluation.md)'s silent-failure problem with *no seam to put a probe on*. (Crack → you need stages you can evaluate independently.)
- **You can't *cache* it.** "How do I reset my password?" arrives a thousand times a day. With one prompt every hit pays full price for classification + retrieval + a long generation. There's no boundary at which to say "this sub-result is reusable," because there are no sub-results. (Crack → you need named stages whose outputs can be cached.)
- **You can't *swap a model into* it.** Classification is a cheap, easy job a small fast model nails; the grounded *answer* may want your best (most expensive) model. Fused into one prompt, the whole thing runs on one model — you overpay for the easy parts and can't independently upgrade the hard one. (Crack → you need per-stage model choice.)
- **You can't tell *which sub-task failed*.** A bad answer is just "the prompt was bad," so you tune the one giant instruction and pray — every edit risks regressing a part that was fine. (Crack → you need components that fail, and are fixed, in isolation.)

Each crack is the system telling you a boundary belongs there. Four cracks, four boxes.
- *Build consequence:* The decomposition isn't arbitrary or aesthetic — **every box earns its place by a capability the monolith lacked** (testability, caching, model choice, attributable failure). When you defend an architecture, you defend it box by box: *"this is a separate component because otherwise I can't ___."* If a box can't name the failure it prevents, merge it back; if a stage of your monolith hits one of these cracks, split it out.

Redraw the support assistant as four boxes with the arrows between them. This is the **spine of the chapter** — every later Part adds a layer *onto this same diagram* (Part B decides which box, if any, is an agent; Part C picks the orchestration pattern wiring the boxes; Part D slips a gateway under every model box; Part E wraps the whole thing in caching, guardrails, and the data plane; Part F threads streaming and a span tree through it). Memorize these four boxes:

```
        ┌─────────────────────────────────────────────────────────────────────┐
        │                       SUPPORT ASSISTANT (compound system)            │
        │                                                                       │
  user  │   ┌────────────┐     ┌────────────┐     ┌────────────┐     ┌────────┐ │
 msg ───┼──▶│ CLASSIFIER │────▶│ RETRIEVER  │────▶│  ANSWERER  │────▶│VALIDATOR│─┼──▶ reply
        │   │  (model)   │     │ retrieve() │     │  (model)   │     │ (checks)│ │
        │   └────────────┘     └────────────┘     └────────────┘     └────────┘ │
        │    intent +           top-k help          grounded          safe /    │
        │    route              articles            draft answer      on-policy?│
        │                                                              │        │
        │                                              fail ◀──────────┘        │
        │                                       (revise / refuse / escalate)    │
        └─────────────────────────────────────────────────────────────────────┘
```

Read the diagram as four single-responsibility components and the edges that carry data between them:

- **CLASSIFIER** *(a model call)* — one job: turn the raw message into a routed intent (e.g. `billing` / `how-to` / `bug-report` / `abuse`). Cheap, fast, small model. Interface: `classify(message) -> intent`.
- **RETRIEVER** *(your [Chapter 4](04-rag.md) `retrieve()`, unchanged)* — one job: given the (intent-scoped) query, return the top-k help-center chunks. Interface: `retrieve(question) -> chunks`. This is a *drop-in reuse* of an artifact you already own — the whole point of clean interfaces.
- **ANSWERER** *(a model call)* — one job: write an answer **grounded only in** the retrieved chunks. Can be your most capable (most expensive) model, since this is the hard part. Interface: `answer(question, chunks) -> draft`.
- **VALIDATOR** *(checks — a cheap model call and/or deterministic rules)* — one job: decide whether the draft is safe, on-policy, and actually grounded in the chunks, and if not, route it back to revise / refuse / escalate to a human. Interface: `validate(draft, chunks) -> ok | failure`.

Now re-read the four cracks against the four boxes and watch each one close: you can **test** each box on its own inputs (feed the classifier 50 labelled messages and score it; feed the retriever the [Chapter 7](07-evaluation.md) probe set); you can **cache** the classifier's intent and the retriever's chunks for repeated questions (Part E makes this concrete); you can **swap a model into** any box independently — small/cheap on the classifier, best/expensive on the answerer; and when an answer is wrong you can read the trace and see **which box failed** ([Chapter 8](08-deployment.md)'s span tree, generalized in Part F) instead of blaming "the prompt." Same capability as the monolith; four seams where there were none.
- *Build consequence:* This four-box diagram is what you draw *before* writing any code, and it is the artifact every later Part decorates. The boxes are **stable interfaces** — `classify`, `retrieve`, `answer`, `validate` — so each can be built, tested, and replaced without touching its neighbours, exactly the way [Chapter 4](04-rag.md)'s `retrieve(question) -> chunks` let you swap brute-force search for pgvector without changing a caller. Architecture means *these components and these edges*; it does not mean "more code" or "adopt a framework."

**Generalizing Chapter 5's ladder from one agent to the whole system.** [Chapter 5](05-agents.md) taught the move *within* a single feature: **push work down the ladder** — agent → workflow → single call — and pick the *least* autonomous thing that solves the task, because every rung up costs predictability, money, and latency. At the *system* level that same ladder applies **per box, not once for the whole product.** Look at the four boxes through it: the CLASSIFIER is a *single call* (one prompt in, one label out — never make this an agent); the RETRIEVER is *deterministic code* (the lowest rung of all — it's a function, no model deciding control flow); the ANSWERER is a *single call* (chunks + question in, grounded draft out); the VALIDATOR is a *single call or pure rules*. The whole support assistant — a real, shippable product — needs **zero agents**: it's a fixed four-step *workflow*, the cheapest, most testable, most predictable shape there is. The senior instinct generalizes: don't ask "should this product be an agent?"; ask, **for each box, what is the least-autonomous component that does this box's one job?** Most boxes in most systems answer "a single call" or "plain code," and an agent, when one is justified at all, is *one island inside the diagram* — never the diagram itself. (Part B makes that decision rigorous and names the anti-pattern of reaching for an agent by default.)
- *Build consequence:* Run the [Chapter 5](05-agents.md) ladder over **every box** in your diagram, independently. A system where every box sits on the lowest rung that works is cheap, fast, and testable; a system that's "one big agent" is the monolith from concept 1 wearing an autonomy costume — every crack from concept 2, now also non-deterministic. Decompose first, then right-size each box down the ladder.

**Hands-on ([Part A](#part-a--compound-ai-systems-the-system-is-the-unit-not-the-prompt)):** Take your [Chapter 4](04-rag.md) RAG bot — the one that today is, honestly, one `retrieve()` call feeding one answer prompt — and, **on paper, redraw it as a compound system.** No code yet; this is a boxes-and-arrows exercise.
1. List every component the bot really contains, even the ones currently fused into one step: an **embedder** (turns the question into a vector), the **retriever** (`retrieve()` — top-k chunks), a **prompt-assembler** (stitches question + chunks + instructions into the final prompt), the **answerer model** (writes the grounded answer), and an **optional validator** (checks the answer is grounded in the chunks before it ships — [Chapter 7](07-evaluation.md)'s faithfulness check as a box).
2. Draw the arrows: `question → embedder → retriever → prompt-assembler → answerer → (validator) → answer`, with the validator's failure edge looping back to "revise / refuse."
3. For **each box**, write two things: its **single responsibility** in one sentence ("the embedder's only job is question → vector") and its **interface** — the signature of what goes in and what comes out (`embed(text) -> vector`, `retrieve(question) -> chunks`, `assemble(question, chunks) -> prompt`, `answer(prompt) -> draft`, `validate(draft, chunks) -> ok | failure`).
4. For each box, note in the margin the one failure it prevents *or* the cost/latency it buys (e.g. "validator: prevents an ungrounded answer reaching the user"; "separate embedder: lets me swap the embedding model without touching retrieval"). If a box can't name one, you've found a box to merge away.

Keep this diagram. Later Parts pick *which* of these boxes (if any) needs an agent, *which* orchestration pattern wires them, and *which* cross-cutting layers wrap them — all drawn onto this same picture.

## Part B — Workflows vs agents at the system level: the orchestration decision

[Chapter 5](05-agents.md) taught the **agent loop** (a model that thinks → acts → observes → repeats, choosing its own path at runtime) and the **workflow-vs-agent** distinction *for one feature*: push work down the ladder, pick the least-autonomous component that solves the task. **This Part does not re-teach that loop or re-argue the ladder.** It promotes the same decision from "which component is this one feature?" to the **top-level orchestration decision for a whole system** — applied per box across the [Part A](#part-a--compound-ai-systems-the-system-is-the-unit-not-the-prompt) diagram — and then [Part C](#part-c--the-five-orchestration-patterns-the-workflow-vocabulary) adds the *named patterns* Chapter 5 never enumerated. Quick vocabulary refresh so the rest reads cleanly: a **workflow** is a fixed pipeline whose control flow *you* wrote in advance (you decide which call runs next); an **agent** is a model that *owns its own control flow* at runtime (it decides which call runs next, and when to stop). The axis is the same one Chapter 5 named — **who decides the next step** — only now we're deciding it for every box at once.

**3. A real system is a deterministic workflow with autonomous islands — not an agent with deterministic bits.**
The instinct, once you've built one agent, is to picture the whole product as "an agent that occasionally does fixed things." Flip that picture: a shippable system is a **fixed pipeline** — you own the control flow end to end — with at most a few **autonomous islands** dropped in only where the path genuinely can't be known ahead of time. The default is determinism; autonomy is the exception you justify, box by box, never the substrate. Think of an assembly line that you, the engineer, laid out: every station is in a fixed order *you* chose, and the parts move predictably from one to the next. One station might be a skilled worker improvising against a part that arrives in an unpredictable state — but you don't tear out the line and replace it with a crowd of improvisers because *one* station needed judgement. The line stays; the improviser is one station on it.

Run the [Part A](#part-a--compound-ai-systems-the-system-is-the-unit-not-the-prompt) spine through this lens, box by box, asking only **"can I write the path for this box in advance?"**:

```
  user ──▶ CLASSIFIER ──▶ RETRIEVER ──▶ ANSWERER ──▶ VALIDATOR ──▶ reply
           workflow       workflow       workflow      workflow
           (1 call)       (code)         (1 call)      (call/rules)
                                            │
                              ┌─────────────┘
                              ▼
                   open-ended troubleshooting?
                   ── the ONLY candidate island ──
                   agent (model owns the path)
```

- **CLASSIFIER** — one message in, one label out. The path is `classify(message) -> intent`; there *is* no branching to discover. Single call. Workflow.
- **RETRIEVER** — `retrieve(question) -> chunks`. Deterministic code, no model deciding anything. The lowest rung on the ladder.
- **ANSWERER** — `answer(question, chunks) -> draft`. One prompt in, one grounded draft out. Single call. Workflow.
- **VALIDATOR** — `validate(draft, chunks) -> ok | failure`, a cheap call or pure rules with a fixed fail-edge. Workflow.

Four of four boxes are deterministic. The whole support assistant — a real product — ships with **zero agents**. The *only* place an agent could earn its keep is a fifth path that isn't even drawn yet: **open-ended troubleshooting** — "the user's problem isn't in the docs, walk them through diagnosing it," where the next question genuinely depends on the last answer and you cannot pre-write the sequence. That single box is the candidate island; everything routing, retrieval, and formatting around it stays a fixed pipeline. Roughly 90% of the boxes are deterministic, and the autonomous one is surrounded by — not in charge of — the workflow.
- *Build consequence:* When you draw a system, your default pen draws **workflow**. An agent box is something you add only after writing the sentence *"the path through this box can't be predetermined because ___"* and meaning it. If you can't finish that sentence for a box, it is a workflow — downgrade it. The architecture is a pipeline with islands; if your diagram is an agent with a few fixed bits, you've inverted the default and bought every cost in concept 4 for boxes that never needed it.

**4. Agent-washing — wrapping a knowable pipeline in a loop because it sounds advanced — is the system-level monolith from [Part A](#part-a--compound-ai-systems-the-system-is-the-unit-not-the-prompt), now also non-deterministic.**
**Agent-washing** is reaching for an agent when a workflow would do: taking a path you *could* write out as fixed steps and instead handing it to a model in a loop, because "agentic" reads as more sophisticated than "pipeline." It is the exact mistake [Chapter 5](05-agents.md) named with the **fetch-order → check-status → email-customer** example — three steps fully known in advance, so it's a workflow; building it as an agent is pure over-engineering. Here the same mistake operates at *system* scale, and it's worth seeing it is precisely the [Part A](#part-a--compound-ai-systems-the-system-is-the-unit-not-the-prompt) monolith wearing an autonomy costume: one undifferentiated thing doing everything, except now the control flow is non-deterministic too, so you've kept every crack from Part A (can't test, can't cache, can't swap a model in, can't tell what failed) and *added* three more:

- **Non-determinism.** A workflow runs the same path every time; an agent may take a different path on identical input, so the same ticket can resolve two different ways. You can't reason about a system whose control flow changes per run.
- **Untestability.** A fixed pipeline has stages you can probe one at a time ([Part A](#part-a--compound-ai-systems-the-system-is-the-unit-not-the-prompt)'s seams; [Chapter 7](07-evaluation.md)'s independent eval per box). A loop that decides its own steps has no fixed stage to attach a probe to — you're back to "the agent was bad."
- **Runaway spend.** [Chapter 5](05-agents.md)'s signature agent failure: a loop with no upper bound on steps can fetch → get a confusing result → fetch again, forever — a runaway bill and a hammered downstream API. A workflow makes exactly N calls; an agent makes "however many it decides," and the one time it decides wrong is an incident. A four-call workflow that you agent-washed into a loop now needs a step limit, a cost budget, and loop detection to be safe — *guardrails you only need because you added autonomy you didn't need*.

The tell is simple: if you can name the steps in order before any input arrives, it's a workflow, and an agent there is agent-washing.
- *Build consequence:* Before any box becomes an agent, try to write its steps as a fixed list. If you can — it's a workflow; building it as an agent buys you non-determinism, untestability, and runaway-spend exposure in exchange for nothing. The decision ladder is unchanged from [Chapter 5](05-agents.md) (single call → workflow → agent, least-autonomous that works), but now you run it **per box across the whole diagram**, and the system-level default is: pipeline everywhere, agent only on the island you can defend in one sentence.

**Hands-on ([Part B](#part-b--workflows-vs-agents-at-the-system-level-the-orchestration-decision)):** Take your [Part A](#part-a--compound-ai-systems-the-system-is-the-unit-not-the-prompt) diagram and label **every box** either `deterministic` (workflow — you own the control flow) or `autonomous (agent)` (the model owns the control flow). For each box you mark autonomous, write **one sentence** naming why the path through it can't be predetermined ("the next diagnostic question depends on the user's previous answer, which I can't enumerate in advance"). The rule is strict: **if you can't write that sentence, downgrade the box to a workflow.** Then add up the boxes — you should find the overwhelming majority are deterministic and at most one is a genuine island. If more than one box came out autonomous, re-examine each: are its steps really unknowable, or did you just reach for a loop because it sounded advanced (agent-washing)? Keep the labelled diagram; [Part C](#part-c--the-five-orchestration-patterns-the-workflow-vocabulary) names the pattern wiring the deterministic boxes together.

---

## Part C — The five orchestration patterns (the workflow vocabulary)

[Part B](#part-b--workflows-vs-agents-at-the-system-level-the-orchestration-decision) settled that most of a system is deterministic workflow. But "deterministic workflow" is not formless — there are exactly **five reusable shapes** that cover almost every fixed pipeline you'll build, and knowing their names turns "wire some calls together" into "pick the pattern that repairs my specific symptom." Each is taught the same way: a one-sentence **claim**, a tiny **diagram**, the **crack in the previous pattern it repairs** (so you see *why* it exists, not just *that* it does), a short **code sketch** (Anthropic first, OpenAI noted where the surface differs — the patterns are identical across providers, only the call shape changes), and a `*Build consequence:*`. They build on each other: chaining is the baseline, and each later pattern fixes a way the baseline breaks.

> **Setup assumed:** an Anthropic client (`client = anthropic.Anthropic()`) or OpenAI client (`client = OpenAI()`), key from the environment; a one-line helper `call(model, prompt) -> str` wrapping `client.messages.create(...).content[0].text` (Anthropic) or `client.chat.completions.create(...).choices[0].message.content` (OpenAI), so the sketches stay about *shape*, not boilerplate. Model IDs: a cheap tier (`claude-haiku-4-5` / `gpt-4o-mini`) and a capable tier (`claude-sonnet-4-6` / `gpt-4.1`).

**5. Prompt chaining — do A, then feed A's output into B — is the baseline workflow: decompose one hard call into a fixed sequence of easier ones.**
**Prompt chaining** means breaking a task into an ordered series of model calls where each call's output is the next call's input — `extract → translate → format`, a fixed line. It exists because one prompt asked to do three things at once does each of them worse (and gives you no seam to inspect the middle); splitting them lets each call do one job well and lets you check the intermediate result before paying for the next step.

```
  in ──▶ [call A] ──▶ A_out ──▶ [call B] ──▶ B_out ──▶ out
```

```python
# Anthropic. Each step's output is the next step's input — a fixed, ordered chain.
outline = call("claude-haiku-4-5",  f"Outline a help-doc answer for: {question}")
draft   = call("claude-sonnet-4-6", f"Write the answer from this outline:\n{outline}")
final   = call("claude-haiku-4-5",  f"Tighten to 3 sentences, keep all facts:\n{draft}")
# OpenAI: identical shape, swap the model IDs (gpt-4o-mini / gpt-4.1) and the call helper.
```

- *Build consequence:* This is the default shape of the [Part A](#part-a--compound-ai-systems-the-system-is-the-unit-not-the-prompt) spine — `CLASSIFIER → RETRIEVER → ANSWERER → VALIDATOR` *is* a chain. Reach for it whenever one prompt is trying to do too much; the cost is added latency (calls run in series), which the next patterns address.

**6. Routing — an LLM-as-classifier picks the path, then dispatches — repairs the chain that has to handle wildly different inputs with one prompt.**
The crack in a plain chain: a single `ANSWERER` prompt asked to handle refunds *and* deep technical bugs *and* billing disputes is mediocre at all three, because one instruction can't be tuned for inputs that need different tones, tools, and knowledge. **Routing** repairs it by classifying the input first and **dispatching** to a specialized downstream prompt — the model acts as a *one-shot classifier* that names a category, and *your code* uses that label to choose the handler. This **is the spine's CLASSIFIER box**.

```
                   ┌─▶ refund handler
  in ──▶ [classify]┼─▶ technical handler
                   └─▶ billing handler
```

```python
# Anthropic. A CHEAP model classifies; YOUR code dispatches to a specialized prompt.
intent = call("claude-haiku-4-5",
              f"Classify as one of refund|technical|billing. Reply with one word.\n{msg}").strip()
handler = {"refund": refund_prompt, "technical": tech_prompt, "billing": billing_prompt}[intent]
answer  = call("claude-sonnet-4-6", handler(msg))   # specialized downstream prompt
# OpenAI: same — gpt-4o-mini classifies, your dict dispatches, gpt-4.1 answers.
```

The classifier is a textbook fit for [Chapter 8](08-deployment.md)'s **model tiering**: labelling refund-vs-technical-vs-billing is easy, so it runs on the cheap fast tier (`claude-haiku-4-5` / `gpt-4o-mini`), and only the hard downstream answer pays for the capable tier. Most traffic is routed by a model that costs a fraction of the one that answers.
- *Build consequence:* Routing lets you keep each downstream prompt simple *and* keep the cheap model on the easy classify step — the highest-leverage cost win in most systems. **Don't confuse routing with an agent:** routing is a *one-shot classify-then-dispatch* — the model emits one label and *your code* owns the branch; it never loops or decides the next step. An agent owns the control flow; a router hands control straight back to you.

**7. Parallelization — run independent work concurrently (sectioning) or the same work redundantly (voting) — repairs the chain that pays for in-series latency or single-shot unreliability.**
A chain runs strictly in series, so a task with three *independent* sub-parts pays `t1 + t2 + t3` for no reason, and a single call on a flaky judgement is one roll of the die. **Parallelization** has two flavours. **Sectioning** splits a task into independent sub-tasks and fans them out concurrently, collapsing wall-clock to the slowest leg — this is exactly [Chapter 8](08-deployment.md)'s parallel-fan-out (`asyncio.gather` / thread pool), now used as an *orchestration* shape, not just a latency tweak. **Voting** runs the *same* task N times and aggregates (majority vote) for reliability — independent samples cancel out one-off errors.

```
            ┌▶[task A]┐                     ┌▶[run #1]┐
  in ──▶ split│▶[task B]│▶ aggregate ──▶ out    in ──▶│▶[run #2]│▶ majority ──▶ out
            └▶[task C]┘                     └▶[run #3]┘
            (sectioning)                        (voting)
```

```python
# Sectioning (Anthropic, async): independent sub-tasks at once → wall-clock = slowest leg.
summary, sentiment, pii = await asyncio.gather(
    acall("claude-haiku-4-5", f"Summarize:\n{msg}"),
    acall("claude-haiku-4-5", f"Sentiment (pos/neg/neutral):\n{msg}"),
    acall("claude-haiku-4-5", f"Does this contain PII? yes/no:\n{msg}"))

# Voting: run the same risky check 3x, take the majority — cancels one-off errors.
votes = [call("claude-haiku-4-5", f"Is this abusive? yes/no:\n{msg}").strip() for _ in range(3)]
verdict = max(set(votes), key=votes.count)   # e.g. ["yes","no","yes"] -> "yes"
# OpenAI: asyncio.gather over client.chat.completions.create; voting loop identical.
```

- *Build consequence:* Use **sectioning** when sub-tasks don't depend on each other (only independent calls can be fanned out — a step needing the previous step's output stays serial). Use **voting** to buy reliability on a single high-stakes judgement at the cost of N× spend — so reserve it for the few checks that matter, on the cheap tier.

**8. Orchestrator-workers — one model dynamically plans the split, then spawns bounded worker calls — repairs parallelization's assumption that you knew the sub-tasks in advance.**
Sectioning assumes *you* can enumerate the sub-tasks when you write the code. But sometimes the split depends on the input — "research this question across however many sub-topics it turns out to have" — and you can't hard-code three fixed legs. **Orchestrator-workers** repairs that: an **orchestrator** model reads the input and *decides the breakdown at runtime*, then dispatches a **worker** call per piece and synthesizes the results. The split is dynamic; each worker is still a **single bounded call**, not a free loop.

```
  in ──▶ [orchestrator: plan subtasks] ──▶ {worker, worker, worker} ──▶ [synthesize] ──▶ out
                  (dynamic split)            (bounded calls)
```

```python
# Anthropic. Orchestrator decides the split; each worker is ONE bounded call; you own the loop.
subtasks = json.loads(call("claude-sonnet-4-6",
    f'Break this into independent sub-questions. Return a JSON list.\n{question}'))
results  = [call("claude-haiku-4-5", f"Answer concisely:\n{s}") for s in subtasks]  # bounded
final    = call("claude-sonnet-4-6", f"Synthesize one answer from:\n{results}")
# OpenAI: same three stages; swap model IDs and the call helper.
```

- *Build consequence:* Use this only when the sub-tasks genuinely can't be enumerated ahead of time — otherwise it's sectioning (concept 7) and cheaper. **Don't confuse orchestrator-workers with multi-agent (or with a full agent):** the orchestrator *plans the split*, but the workers are **bounded single calls** and **you still own the loop** — there's no model running free, choosing tools, and deciding when to stop. State the axis explicitly: *who owns control flow, and is the loop bounded?* Here **you** own it and it's bounded (a fixed `for` over a planned list); an agent owns it and the loop is open-ended. That one line is the whole difference.

**9. Evaluator-optimizer — pair a generator with a critic that scores and sends it back to revise — repairs the chain whose first draft just isn't good enough.**
The last crack: a single forward pass produces a draft, and for hard outputs the first draft is often *almost* right — but a plain chain has no step that *catches* that and improves it. **Evaluator-optimizer** repairs it with a **generate → critique → revise** pair: one call produces a draft, a second call (the **evaluator**) scores it against a rubric, and if it fails you feed the critique back for a bounded revise. This is [Chapter 5](05-agents.md)'s **reflection** pattern (the model critiques and revises its own work) made into a *named two-component loop* — and the evaluator is exactly [Chapter 7](07-evaluation.md)'s **LLM-as-judge**, the same faithfulness judge, used **inline** in the request path instead of offline on a test set. It **is the spine's VALIDATOR loop**, generalized: validate → fail → revise.

```
  in ──▶ [generate] ──▶ draft ──▶ [evaluate vs rubric] ──fail──▶ (feed critique back) ─┐
                                          │ pass                                       │
                                          ▼                                    [generate] ◀┘
                                         out                                   (bounded: ≤1 revise)
```

```python
# Anthropic. Generate -> judge scores -> revise ONCE if it fails. The judge is ch07's, used inline.
draft = call("claude-sonnet-4-6", f"Answer, grounded only in:\n{chunks}\nQ: {question}")
score, critique = judge(question, chunks, draft)   # ch07 faithfulness judge: 1..3 + reason
if score < 3:                                      # failed the bar -> one bounded revision
    draft = call("claude-sonnet-4-6",
                 f"Revise to fix this critique:\n{critique}\nDraft:\n{draft}\nContext:\n{chunks}")
# OpenAI: identical; the judge is just another client.chat.completions.create call.
```

- *Build consequence:* Use it where there's a *clear quality bar a critic can check* (grounded-in-the-chunks, matches-the-schema, on-policy) and the first pass sometimes misses — it buys reliability for the price of one or two extra calls. **Bound the revise loop** (one or two passes, never open-ended) or you've reinvented an agent: the loop is yours and capped, the model doesn't decide when to stop. Same judge as your offline eval, pointed inline.

**One menu, not a list to memorize — match the symptom to the pattern:**

| Symptom you have | Pattern | Who owns control flow | Loop? |
|---|---|---|---|
| One prompt is doing too many things; I want inspectable middle steps | **Prompt chaining** (5) | You (fixed order) | No |
| One prompt handles wildly different inputs badly | **Routing** (6) | You (dispatch on a label) | No |
| Independent sub-tasks waste time run in series | **Parallelization — sectioning** (7) | You (fan-out) | No |
| One call is unreliable on a high-stakes judgement | **Parallelization — voting** (7) | You (aggregate N runs) | No |
| I can't enumerate the sub-tasks until I see the input | **Orchestrator-workers** (8) | You (orchestrator plans, workers bounded) | Bounded |
| The first draft isn't good enough | **Evaluator-optimizer** (9) | You (capped revise) | Bounded |
| The path genuinely can't be known until runtime | **Agent** ([Ch 5](05-agents.md)) | The model | Open-ended (guard it) |

Read the last two columns top to bottom: every pattern above the agent row leaves **you** owning control flow, with the loop either absent or **bounded**. The single thing that distinguishes an agent from all five workflow patterns is that the model owns the loop and the loop is open-ended — which is why an agent is the *one* row that needs [Chapter 5](05-agents.md)'s step-and-cost guardrails. Pick the lowest row that fixes your symptom.

**Hands-on ([Part C](#part-c--the-five-orchestration-patterns-the-workflow-vocabulary)):** Build a **router from scratch**, then wrap one path in an **evaluator-optimizer**, and measure each pattern's cost.
1. **Router (concept 6).** Write a cheap-model classifier that tags an incoming question as `factual` / `computational` / `chit-chat` (one Haiku/`gpt-4o-mini` call, reply-with-one-word). Dispatch on the label in your own code: `factual → your retrieve() path` ([Part A](#part-a--compound-ai-systems-the-system-is-the-unit-not-the-prompt)'s RETRIEVER → ANSWERER chain), `computational → a calculator function` (no model — let code do math), `chit-chat → a single direct answer`. Confirm three different inputs take three different paths.
2. **Evaluator-optimizer (concept 9).** On the `factual` path, wrap the answer in generate → judge → revise: produce the grounded draft, score it with your [Chapter 7](07-evaluation.md) faithfulness judge, and **revise exactly once** if it scores below the bar. Prove it: feed a question whose first draft drifts off the chunks and watch the second pass pull it back.
3. **Measure.** For each pattern, record **added latency** (router adds one cheap classify call ≈ the Haiku round-trip; evaluator-optimizer adds a judge call always plus a revise call only on failure) and **added cost** (extra calls × their tier's price). You should be able to say, in numbers, *"routing cost me one cheap call and saved an expensive mis-answer; evaluator-optimizer cost me ~1.3 calls on average and lifted faithfulness from X to Y."* That sentence — cost paid vs failure prevented — is how you defend every box in the diagram.

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
