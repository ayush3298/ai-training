## LLM Application Architecture & System Design: composing reliable systems, not one big prompt

**Goal:** By the end you can take a vague product request ("build an AI feature that does X") and **draw its system architecture**: decide which parts are deterministic *workflow* versus autonomous *agent*, pick the orchestration pattern for each step (chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer), place the cross-cutting layers — a provider-gateway boundary, caching tiers, guardrails, the context/data plane, streaming, observability — as **named components** rather than inline code, and justify every box by the failure it prevents or the cost/latency it buys. You produce one architecture diagram plus a thin "compound system" skeleton that wires your earlier-chapter artifacts (`retrieve()`, the agent loop, `tracked_call`) behind clean interfaces.

**Why this matters:** Each earlier chapter handed you one capability in isolation — a prompt ([Chapter 3](03-prompt-engineering.md)), a retriever ([Chapter 4](04-rag.md)), an agent ([Chapter 5](05-agents.md)), an endpoint ([Chapter 8](08-deployment.md)). Real products are none of those alone; they're a **compound system** of several models, retrievers, tools, and validators wired together. The dominant failure at this stage is "I built one giant prompt (or one all-powerful agent) to do everything" — a thing that is unpredictable, expensive, untestable, and impossible to debug because you can't tell *which sub-task* failed. This is the highest-leverage chapter for shipping real systems: where you learn to decompose the problem, push each piece down to the *least-autonomous* component that solves it, draw the trust and control boundaries, and make the architecture decisions *on paper before* you write the operational code in [Chapter 8](08-deployment.md). The cost of not knowing this is the demo that won't survive contact with the second use case.

> **Setup assumed:** same as before, plus three artifacts you already built: your [Chapter 4](04-rag.md) `retrieve(question) -> chunks`, your [Chapter 5](05-agents.md) agent loop, and the [Chapter 8](08-deployment.md) `tracked_call` logging wrapper. This chapter wires them behind clean interfaces; it does not re-teach them. The work is mostly **boxes-and-arrows on paper** plus thin Python skeletons that name components and their signatures — no new infrastructure beyond what earlier chapters already used (Docker for any local store, secrets from the environment).

**Suggested split:** two sessions. Session 1 = Parts A–C (decompose the system into components; the workflow-vs-agent decision at the *system* level; the five orchestration patterns — the heart of the chapter). Session 2 = Parts D–F (the provider-gateway boundary; the cross-cutting caching / guardrails / data-plane layers; streaming and observability) plus the deliverable.

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

## Part D — The gateway / provider-abstraction boundary

So far every box in the spine that calls a model — the CLASSIFIER, the ANSWERER, the VALIDATOR — has reached out to a provider *directly*. That's fine on the diagram, where "model call" is one arrow. It stops being fine the moment you look at the code those arrows hide, because each one names a specific provider's SDK, response shape, and field names. This Part slips **one model-agnostic seam under every model box** so that "which provider, which model" stops being code and becomes configuration. We **build on** the call-shape differences you already met in [Chapter 2](02-apis-and-integration.md) and the multi-provider-fallback need from [Chapter 8](08-deployment.md); we do **not** re-cover key/secret handling (that stays [Chapter 8](08-deployment.md)) or the *operational* depth of retries and circuit breakers (that lives in [Chapter 8](08-deployment.md) Part I, the Resilience graft). This Part is the *abstraction seam*; those Parts operate on it.

**10. Put one model-agnostic boundary — a gateway — between your application and every provider, so "which provider/model" is configuration, not code.**
You felt this crack already in [Chapter 2](02-apis-and-integration.md): the *same* call is shaped differently per provider — the system prompt is a top-level `system=` on one and a `{"role":"system"}` message on the other; the reply lives at `resp.content[0].text` on one and `resp.choices[0].message.content` on the other; usage is `input_tokens`/`output_tokens` here and `prompt_tokens`/`completion_tokens` there; even the stop reason is spelled `end_turn` vs `stop`. If each of your three model boxes calls a provider SDK directly, every one of those differences is **hard-coded in three places**, and "let's try the other provider for the answerer" becomes a code change in your application logic — the exact thing architecture is supposed to prevent.

The fix is the same move a senior makes with any volatile dependency: **hide it behind one stable interface and let the variation live behind that interface.** Think of a **universal power adapter** — the travel plug you carry so your laptop doesn't care whether the wall socket is British, German, or Indian. Your appliance exposes one plug shape; the adapter absorbs the per-country difference; you change *countries* without changing *appliances*. The **gateway** (also called a **provider-abstraction boundary**, or an "LLM proxy" in product form) is that adapter for model calls: your whole application calls *one* function — `call(messages, model)`-shaped — and behind it sit thin **provider adapters** that translate to and from each provider's native shape. "Which provider/model" is now a string argument and a config table, not a branch in your business logic.

Concretely, the boundary is a single function whose job is to take a provider-neutral request, dispatch to the right adapter, and normalize the response back to **one shape your app agrees on**:

```python
# ONE signature the whole app calls. Behind it, per-provider adapters normalize to one response shape.
from dataclasses import dataclass

@dataclass
class LLMResponse:                 # the ONE shape every caller sees, regardless of provider
    text: str
    input_tokens: int
    output_tokens: int
    stop_reason: str               # normalized: "stop" | "length" | "tool_use"

def llm_call(messages, model, *, system=None, max_tokens=1024, temperature=0.2) -> LLMResponse:
    provider = MODEL_TABLE[model]["provider"]      # "which provider" is a config lookup, not a branch
    if provider == "anthropic":
        r = anthropic_client.messages.create(
            model=MODEL_TABLE[model]["id"], system=system or "",     # system is a TOP-LEVEL param here
            max_tokens=max_tokens, temperature=temperature, messages=messages,
        )
        return LLMResponse(r.content[0].text, r.usage.input_tokens, r.usage.output_tokens,
                           {"end_turn": "stop", "max_tokens": "length"}.get(r.stop_reason, r.stop_reason))
    if provider == "openai":
        msgs = ([{"role": "system", "content": system}] if system else []) + messages  # system is a MESSAGE here
        r = openai_client.chat.completions.create(
            model=MODEL_TABLE[model]["id"], max_tokens=max_tokens, temperature=temperature, messages=msgs,
        )
        u = r.usage
        return LLMResponse(r.choices[0].message.content, u.prompt_tokens, u.completion_tokens,
                           r.choices[0].finish_reason)
    raise ValueError(f"unknown provider for model {model!r}")
```

That's ~15 lines doing one thing: absorbing the [Chapter 2](02-apis-and-integration.md) differences so no caller upstream ever sees them. The CLASSIFIER calls `llm_call(msgs, "fast")`, the ANSWERER calls `llm_call(msgs, "best")`, the VALIDATOR calls `llm_call(msgs, "fast")` — and *none of them names a provider*. The model boxes on the spine are unchanged in responsibility; they just no longer reach past the boundary.
- *Build consequence:* Draw the gateway as a thin horizontal band *under* the three model boxes of [Part A](#part-a--compound-ai-systems-the-system-is-the-unit-not-the-prompt)'s diagram — every model arrow now passes through it. The win is testability and swappability of the *seam itself*: you can point the whole system at a stub `llm_call` in tests (no API key), and you can change a model in `MODEL_TABLE` without grepping your application. If any box still imports a provider SDK directly, the boundary has a hole and the next concept's unlocks won't hold.

**11. The boundary is what makes fallback chains, model tiering, and build-vs-buy *configuration decisions* instead of rewrites.**
A seam is only worth drawing if it *buys* something. Once every model call goes through one function, three things that were painful become config-level, because there's exactly one place they live:

- **Fallback chains — primary fails, secondary answers, app never notices.** A **fallback chain** is an ordered list of providers/models tried in sequence: call the primary; if it times out, rate-limits, or errors, transparently retry the request on a secondary. With direct SDK calls this logic would be smeared across every call site; behind the gateway it's *one* wrapper around `llm_call`. This is the [Chapter 8](08-deployment.md) concept-11 "second path when the primary fails" made into a property of the boundary rather than a thing you bolt onto each feature — and the *full operational treatment* (timeout → backoff retry → circuit breaker → fallback, in order, with a forced-failure test) is **[Chapter 8](08-deployment.md) Part I / the Resilience graft**; the gateway is simply *where it attaches*.
- **Model tiering / routing as config.** **Model tiering** (a.k.a. routing) is sending each task to the cheapest model that passes its eval — the [Part A](#part-a--compound-ai-systems-the-system-is-the-unit-not-the-prompt) insight that the classifier wants a small fast model and the answerer wants your best one. Without the boundary, that per-box choice is hard-coded in each box. With it, the whole policy is the `MODEL_TABLE` — `"fast" → haiku-class`, `"best" → opus-class` — so re-tiering a task (because a cheaper model now passes [Chapter 7](07-evaluation.md)'s eval, or because [Chapter 8](08-deployment.md) cost pressure demands it) is a one-line config edit, reviewed and rolled back like any deploy. Note this is the *placement* of routing-as-config; the *routing pattern* that picks a path by classifying the request is [Part C](#part-c--the-five-orchestration-patterns-the-workflow-vocabulary)'s job — they compose (the router decides the path, the gateway decides the model).
- **Build-vs-buy: your thin adapter vs a managed gateway.** Your ~15-line `llm_call` *is* a gateway; the market also sells **managed gateways / LLM proxies** (LiteLLM, OpenRouter, and similar) that ship the same boundary plus extras — a unified API across dozens of providers, usage/cost accounting, key management, sometimes caching and routing built in. The decision is ordinary build-vs-buy: hand-roll the thin adapter while you support two providers and want to understand the seam from the inside; reach for a managed proxy when you're juggling many providers or want its accounting/routing for free. Either way the *architectural point is identical* — there is **one boundary**, and your app talks only to it. (Don't mistake the product tour for the lesson: the lesson is *why the seam exists and what it buys*, not which proxy wins.)
- *Build consequence:* These three unlocks are the *return* on drawing the boundary, and they only exist because there's a single seam to attach them to. When someone asks "can we add a fallback provider / route easy traffic to a cheaper model / try a managed proxy?", the answer with a gateway is "yes, it's config"; without one it's "yes, after we touch every call site." Build the seam first; the unlocks are then cheap. Secrets to reach each provider still come from the environment and stay server-side — that's [Chapter 8](08-deployment.md), unchanged; the gateway only abstracts the *call shape*, not the *key handling*.

**Hands-on ([Part D](#part-d--the-gateway--provider-abstraction-boundary)):** Write a single `llm_call(messages, model, **opts)` that normalizes Anthropic and OpenAI responses to one `LLMResponse` shape (text, input/output tokens, normalized stop reason) — the seam from concept 10, with `model` resolved through a `MODEL_TABLE` so the caller never names a provider. Then wrap it in a **fallback chain**: `call_with_fallback(messages, chain=["best", "best_alt"])` that calls the primary and, on a raised error *or* a timeout, transparently retries on the secondary model/provider and returns the *same* `LLMResponse` — the caller can't tell which leg answered. Finally, point your [Part C](#part-c--the-five-orchestration-patterns-the-workflow-vocabulary) router at the gateway: the router decides *which path* a request takes, but every model call on every path goes through `llm_call`, so **no downstream code names a provider directly**. Verify by grepping your feature code for `anthropic`/`openai` imports — they should appear *only* inside the gateway module.

---

## Part E — Cross-cutting layers: caching tiers, guardrails, the context/data plane

[Part D](#part-d--the-gateway--provider-abstraction-boundary) slipped a seam *under* the model boxes. This Part wraps three named layers *around the whole diagram* — caching, guardrails, and the context/data plane — each a band you draw on top of the [Part A](#part-a--compound-ai-systems-the-system-is-the-unit-not-the-prompt) spine, not code you sprinkle inline. We **build on** the prompt-caching and semantic-caching material from [Chapter 8](08-deployment.md) (concepts 8, 9), the guardrail sandwich from [Chapter 7](07-evaluation.md) (concepts 22–25), and the ingestion/chunk/embed/index pipeline from [Chapter 4](04-rag.md); we **place** them here as architecture and keep the *operational* depth where it already lives — deep guardrail engineering is the Security chapter, cache TTL/freshness tuning is [Chapter 8](08-deployment.md). Placed here, operated there.

**12. Caching is a four-tier safety ladder — cheapest-and-safest first, most-powerful-and-most-dangerous last.**
"Add a cache" is not one decision; it's four, and they differ by *how much they can be wrong*. Order them as a ladder you climb only as far as you need, because each rung up saves more but risks more. The image to hold: the bottom rungs **can't serve a wrong answer** (they only ever return something *byte-identical* to what you'd have computed); the top rung **can serve a near-miss** (it returns the answer to a *similar* question, which is where the danger lives).

1. **Prompt-prefix caching** *(free correctness).* The provider caches a stable prompt *prefix* ([Chapter 3](03-prompt-engineering.md)/[Chapter 8](08-deployment.md)) — same system prompt + instructions + retrieved context billed far cheaper on repeat. It **cannot change the answer**: you still make the model call, you just pay less for the unchanged front of it. Structure prompts as stable-prefix → variable-suffix and turn it on. Zero correctness risk; pure win.
2. **Embedding cache** *(don't re-embed identical text).* Embedding the same string always yields the same vector, so caching `text → vector` removes redundant embedding calls on the read path *and* during ingestion. Like tier 1, it's **deterministic** — a cache hit is byte-identical to a recompute, so it's risk-free.
3. **Response cache (exact-match memoize).** Key on the *exact* normalized request (same messages, same model) → store the response; an identical request returns it with **no model call at all**. Still safe, because "exact match" means the input truly is the same. The ceiling: it only fires on byte-identical repeats, so its hit rate is modest.
4. **Semantic cache** *(powerful — and the one that can be confidently wrong).* A **semantic cache** embeds the incoming question and serves a stored answer when a *past* question is close enough in cosine similarity ([Chapter 4](04-rag.md)) — so "how do I reset my password?" and "I forgot my password, what now?" share one cached answer, deflecting a large fraction of traffic with no model call. This is the most powerful tier and the only one that can serve a **near-miss**: set the threshold too loose and you return the refund-policy answer to a *subtly different* question. Re-state the [Chapter 8](08-deployment.md) concept-9 trap plainly: **a wrong cache hit is a confidently wrong answer** — same instrument, but unlike the lower tiers it can be *wrong*, so cache only where answers are stable (a policy, not "what's my order status"), tune the threshold, and set a TTL.

Small worked example — one request, "*how do I reset my password?*", flowing down the ladder:

```
request ─▶ [4] semantic cache  : embed question; nearest stored question cosine 0.93 ≥ 0.92 threshold?
                                  └─ HIT → return stored answer, ZERO model calls.  (done)
           (miss / disabled) ─▶ [3] response cache : exact normalized request seen before?
                                  └─ HIT → return stored response, zero model calls.
           (miss) ──────────▶ build prompt; [2] embedding cache serves the question vector for retrieval;
                                  [1] prompt-prefix cache makes the model call cheaper → MODEL CALL → answer.
```

Read top-down, the request tries to be answered as *cheaply and safely* as possible: tier 4 deflects it entirely if a near-equivalent exists and you've accepted the risk; failing that, tier 3 catches exact repeats; failing that, you do the real work but tiers 1–2 shave its cost. The thousand-a-day "reset my password" question hits tier 4 (or 3) and never reaches the model; a genuinely novel question falls all the way through and pays full price *once*.
- *Build consequence:* Draw caching as the outermost band wrapping the [Part A](#part-a--compound-ai-systems-the-system-is-the-unit-not-the-prompt) diagram, and adopt the tiers **bottom-up**: tiers 1–2 are free correctness — turn them on always; tier 3 is safe and cheap; tier 4 is the one that needs a *risk decision* (threshold, TTL, an allowlist of cacheable question classes) before it ships. The anti-pattern is treating the semantic cache as "free like the prompt cache" — it is not; it's the one rung where a hit can be *wrong*.

**13. Guardrails are an architectural layer — an input station and an output station bolted to the diagram, failing safe by placement.**
[Chapter 7](07-evaluation.md) gave you the **sandwich**: input guardrails → model → output guardrails (concepts 22–25). At the system level that sandwich becomes a *placement rule on the spine*, not a per-call afterthought. Draw two explicit stations:

- an **input station** in front of the model boxes — injection-delimiting of untrusted text (user message, retrieved chunks, tool output), PII handling, abuse/moderation screening *before* a model call is spent;
- an **output station** after the model boxes — structural validation, the groundedness check, output moderation, refusal handling *before* anything reaches the user or a downstream system.

On the spine, the VALIDATOR box you already drew *is* the output station for the answer path; the input station is the new band in front of the CLASSIFIER. The architectural decision here is just one: where the stations sit and which direction they fail. Reuse [Chapter 7](07-evaluation.md)'s rule as a **placement rule** — **fail-safe, not fail-open**: when a guardrail (or the model) errors, the *default* is to block/withhold, not to let traffic through. ("Fail-open" = let it pass when the check breaks; "fail-safe"/"fail-closed" = withhold when the check breaks.) That choice is a property of *where you put the station and what it does on error*, decided on paper, not discovered in an incident.
- *Build consequence:* Draw the two stations as bands bracketing the model boxes and label each with its fail direction (default fail-safe; fail-open only for a consciously low-stakes path). Keep the *depth* — which detector, how to engineer each check, the injection-defense ladder — scoped to the **Security chapter**; here you only commit to the *placement* (two stations, brackets around the model, fail-safe by default). The architecture's job is to guarantee there *is* a station on each side; the Security chapter fills in what runs inside it.

**14. The context/data plane is two different paths — a per-request read path and a batch write/maintenance path — and conflating them is the bug.**
The last band wraps the system's *information supply*, and it splits cleanly in two. One is the **context window as a managed budget**; the other is the **data pipeline that fills the store the retriever reads**. Keep them separate or you'll rebuild the index on every request.

- **The context plane (a budget, not a bucket).** The context window is [Chapter 1](01-foundations.md)'s working memory and [Chapter 5](05-agents.md)'s context-growth problem made into a design constraint: a fixed token budget you *allocate* deliberately — system prompt + retrieved chunks + history — rather than dump everything into. **Context engineering** (deciding what earns a slot in that budget per request) is the read-time discipline; it's the difference between "retrieve the top-4 tightest chunks" and "paste the whole document and hope."
- **The data plane (a separate pipeline service with its own lifecycle).** The [Chapter 4](04-rag.md) sequence **ingestion → chunk → embed → index → reindex** is *not* something that happens inline when a user asks a question. It's a **batch service** that runs on its **own trigger** — new documents arrive, or the embedding model changes — and whose *output* is the index the retriever queries. The naming to fix in your head: **ingestion** = turning raw source docs into clean chunks; **(re)indexing** = embedding those chunks and (re)building the searchable store.

The disambiguation that bites people: the **read path** and the **write/maintenance path** are different services on different clocks.

```
WRITE / MAINTENANCE PATH  (batch service, own trigger: new docs OR embedding-model change)
   raw docs ─▶ ingest ─▶ chunk ─▶ embed ─▶ index/reindex ─▶ ┌──────────┐
                                                             │  INDEX   │   (the shared store)
READ PATH  (per request, hot, latency-critical)              └────┬─────┘
   user question ─▶ RETRIEVER.retrieve(question) ──────────────────┘  (queries the existing index)
```

The retriever at request time only ever **reads** an index that *already exists*; building that index is a separate, scheduled job. The classic bug is collapsing the two — running `ingest → chunk → embed → index` *inside* the request handler — which re-embeds the entire corpus on every single question: catastrophically slow, expensive, and pointless, since the corpus didn't change between two users' questions.
- *Build consequence:* Draw the data plane as a **separate box feeding the retriever's index from the side**, with its trigger labeled (new docs / model change) — explicitly *off* the request path. The retriever's `retrieve(question) -> chunks` only reads; ingestion and reindex are a batch service with their own schedule and their own monitoring. If your index rebuild lives in the request handler, you've merged two planes that must stay separate — split them, and the read path gets fast and cheap for free.

**Hands-on ([Part E](#part-e--cross-cutting-layers-caching-tiers-guardrails-the-contextdata-plane)):** Add a **two-tier cache** to your [Part D](#part-d--the-gateway--provider-abstraction-boundary) gateway. Tier 3: an **exact-match response cache** keyed on the normalized request — identical request returns the stored `LLMResponse`, no model call. Tier 4: a **semantic cache** — embed the question (reuse [Chapter 4](04-rag.md) embeddings), cosine-match it against past questions, and serve the stored answer when similarity clears a threshold you pick. Run a **repeated-question workload** (a list with deliberate paraphrases — "reset my password" / "I forgot my password") through the gateway and **measure the hit rate** of each tier. Then **name one question class you'd refuse to semantically cache** and say why (e.g. "what's my order status" / "is my account locked" — answers are *per-user and time-varying*, so a near-miss hit is a confidently wrong, possibly leaking answer). Finally, **draw (don't build)** the ingestion pipeline as a *separate service box* — `raw docs → ingest → chunk → embed → index` — feeding your retriever's index from the side, and **label its trigger** (new docs / embedding-model change). Confirm on the drawing that this box is **not** on the request path: the request path only does `question → retrieve(question) → answer`, reading the index the batch box produced.

## Part F — Streaming architecture & end-to-end observability

Two concerns left, and both refuse to live inside any single box. Look back at the spine — [CLASSIFIER → RETRIEVER → ANSWERER → VALIDATOR](#part-a--compound-ai-systems-the-system-is-the-unit-not-the-prompt). Everything so far decided what a box *is* (a single call, a function, an agent — [Part B](#part-b--workflows-vs-agents-at-the-system-level-the-orchestration-decision)), how the boxes are *wired* ([Part C](#part-c--the-five-orchestration-patterns-the-workflow-vocabulary)), and what gets slipped *under* and *around* them (a gateway — [Part D](#part-d--the-gateway--provider-abstraction-boundary); caching, guardrails, the data plane — [Part E](#part-e--cross-cutting-layers-caching-tiers-guardrails-the-contextdata-plane)). Streaming and observability are different in kind: they're properties of the *whole request as it travels the spine*. Streaming is how the answer leaves the system and reaches a screen; observability is how you see the request move through every box after it has left. Each threads across all four boxes at once, so each is an architectural decision, not a box-local one.

This Part **deepens** two ideas [Chapter 8](08-deployment.md) introduced for a *single endpoint* and lifts them to a *compound system*: streaming as a perceived-latency win ([Chapter 8](08-deployment.md) Part B), and parent/child spans for one request ([Chapter 8](08-deployment.md) Part G). It does **not** re-cover the *operating-over-time* loop those feed — drift detection, online evaluation, turning feedback into eval cases, incident triage — that lives in the **Monitoring, Drift & Continuous-Improvement** extension chapter, and concept 16 hands it off explicitly. Here we get the traces *emitted* across the system and the stream *wired* through it; that chapter operates on them.

**15. Streaming is a transport decision, not a feature toggle — it changes the contract between your system and the client.**
[Chapter 8](08-deployment.md) sold streaming as the cheapest perceived-latency win: a 6-second answer that *starts* appearing immediately feels faster than a 3-second one behind a spinner, because the user reads as it generates. True — but that framing makes streaming sound like a switch you flip on the answerer. At the system level it's bigger: streaming decides **how the answer physically leaves your system and travels to the client**, and that's a *transport* choice — which wire protocol carries the tokens, and what the client is allowed to do while they arrive. Think of it the way you'd think about shipping. "What's in the box" is the answer; **streaming is choosing the delivery vehicle and the contract with the recipient** — do they just sign for it, or can they phone the driver mid-route and redirect? Pick the wrong vehicle and the payload is fine but the experience (and the bill) is wrong. Two questions fall out of that, and you answer them at design time, on the diagram, not in the answerer's prompt.

**The first question — which protocol — has exactly one discriminator: does the client only *receive*, or does it also *send* mid-stream?** A protocol here is just the rule for how bytes move over a held-open connection. Two options cover almost everything:

- **SSE (Server-Sent Events)** — a one-way push. The client opens *one* HTTP connection and the server streams tokens down it as they're generated; the client cannot speak back over that same channel, only listen. It rides on ordinary HTTP, so your existing gateway, load balancer, and auth ([Part D](#part-d--the-gateway--provider-abstraction-boundary)) all just work. This is the right default for the support assistant: the ANSWERER produces tokens, the client renders them, the client has nothing to say until the answer is done. **One-way token push → SSE.**
- **WebSocket** — a two-way pipe. Both ends can send at any time over a single persistent connection. You need this only when the client must talk *during* the stream: interactive turn-taking where the user interjects mid-answer, collaborative cursors, or a **voice agent** where audio frames flow up *while* synthesized speech flows down (the realtime-voice budget in the Beyond-Text chapter lives here). **Interactive / bidirectional or voice → WebSocket.**

Don't reach for WebSocket because it sounds more capable — it's a heavier, statefuller connection that complicates load-balancing and reconnection. Start at SSE; step up to WebSocket only when you can name the mid-stream message the client must send. That's the same defaults-first ladder you've applied to every box: the least machinery that does the job.

**The second question — cancellation — is where streaming stops being cosmetic and starts touching the bill.** *Cancellation* means propagating a "stop now" signal from the client all the way back through your system to the provider call, so generation actually halts. Here's why it's load-bearing and not a nicety. In [Chapter 5](05-agents.md) you learned that an agent's cost is *per step* — each loop iteration is a full model call, and a long run quietly stacks them. Now picture the user hitting **Stop** on a long agent answer halfway through. If your transport only *hides* the rest of the stream from the screen but the request keeps running server-side, the agent keeps looping, the gateway keeps calling the provider, and **you keep paying for every token the user already told you they don't want.** The architecture has to carry the cancel: client closes the stream → your handler observes the disconnect → it cancels the in-flight task → the gateway aborts the provider request → the agent loop stops issuing steps. A stopped stream the user can't see is a UX bug; a stopped stream that's still *spending* is a cost bug, and at scale it's the expensive one.

The third piece is the **client contract for incomplete output**. **Partial rendering** is the agreement about what the client may safely show before the answer is complete — because a stream hands the client a half-finished payload by definition. Streaming free-flowing prose is easy: append tokens, they read along. Streaming *structured* output is the trap — half a JSON object is not valid JSON, half a markdown table renders as garbage, half a tool-call argument is undefined. The contract says: render prose incrementally, but **buffer structured fragments until they parse**, and only then commit them to the screen. And the moment you render model tokens as anything richer than plain text, you're back at [Chapter 8](08-deployment.md)'s rule that *model output is untrusted UGC* — a streamed `<script>` or `javascript:` link is exactly as dangerous arriving token-by-token as it is in one blob. The sanitization depth — HTML-escape, link/scheme allowlist — is [Chapter 8](08-deployment.md)'s safe-render material (planned graft); don't duplicate it, just know the streamed boundary is the *same* boundary and inherits the *same* contract.

- *Build consequence:* Decide streaming on the diagram, not in a prompt. Pick the transport by one question — *does the client send mid-stream?* (no → SSE, yes/voice → WebSocket) — and default to SSE. Wire **cancellation end-to-end** so a client disconnect aborts the in-flight provider call and stops any agent loop; treat "the stream stopped but the spend didn't" as a bug to test for, because that's real money on a long run ([Chapter 5](05-agents.md)). Define the **partial-rendering contract** explicitly: stream prose, buffer structured fragments until they parse, and run every streamed token through the same UGC sanitizer you'd apply to a non-streamed answer ([Chapter 8](08-deployment.md)).

**16. End-to-end observability: one request is a span *tree* across the whole system, and every span carries who it was for.**
[Chapter 8](08-deployment.md) Part G taught the move for a single endpoint: promote a flat log into a **trace**. A *span* is one timed, attributed unit of work — a stopwatch with a label and metadata. The **parent span** is the whole request; each sub-step — a retrieval, a model call, a tool call — becomes a **child span** nested under it, so when a request is slow or wrong you open the trace and read *which child span* burned the latency or returned the junk, instead of knowing only that "the request" was bad. [Chapter 8](08-deployment.md) also told you *not* to invent a schema: emit the **OTel GenAI semantic conventions** — OpenTelemetry's vendor-neutral standard for LLM traces — so any backend can read them, with span types like `inference`, `embeddings`, `retrieval`, `execute_tool` and attributes like `gen_ai.request.model`, `gen_ai.usage.input_tokens` / `output_tokens`, `gen_ai.response.finish_reasons`.

That was *one endpoint*. The compound system generalizes it directly: **a single user request fans across router → retriever → gateway → workers, and each becomes a child span under one parent.** Walk the support assistant and the tree writes itself — the parent span is the inbound request, and under it sit a `CLASSIFIER` inference span, a `RETRIEVER` retrieval span, an `ANSWERER` inference span (whose model and token attributes come straight off the [gateway](#part-d--the-gateway--provider-abstraction-boundary), which is the natural place to emit them), and a `VALIDATOR` span — and if the answerer is an orchestrator-workers agent ([Part C](#part-c--the-five-orchestration-patterns-the-workflow-vocabulary)), each worker call is its own child span too. Nothing new to *log*; you already had these fields. The shift is **structural** — the same attributes hung on a tree that mirrors your architecture instead of flattened into one row. When a support answer is wrong, you don't ask "was the prompt bad?"; you open the trace and see the classifier mis-routed, *or* retrieval returned the wrong chunks, *or* the answerer ignored good chunks, *or* the validator over-blocked. The four cracks from [Part A](#part-a--compound-ai-systems-the-system-is-the-unit-not-the-prompt) — you couldn't tell which box failed — close here, in the trace.

The genuinely new architectural idea is **per-tenant / per-feature cost attribution: tag every span with `tenant_id` so the bill becomes sliceable.** A compound system serving real users has a single AWS-style invoice from each provider, but that number can't answer the questions the business actually asks: *which customer is expensive? which feature loses money? did that one tenant's traffic spike cause last month's bill?* Each span already records its own token usage (it's a GenAI attribute); add one attribute — `tenant_id` (and `feature` if you want) — to **every** span, and the cost rollup falls out of a query: *sum input+output tokens grouped by `tenant_id`, priced per model.* This is the system property that turns "we spent $4,000 on the API" into "tenant A cost $2,900, tenant B cost $40, and the new summarize feature is 60% of the bill" — the difference between a number you stare at and a number you can act on (rate-limit a tenant, reprice a feature, kill a money-losing path). It's nearly free *if* you decide it at design time, because the tag has to be set when the parent span opens and **propagated to every child**; retrofitting tenant tags onto an already-instrumented system is the painful path, so do it now.

Closing the loop — but only halfway, by design. These spans, plus a captured user signal (a 👍/👎, a thumbs, a correction), are the raw material of the feedback flywheel: production traffic becomes the source of your next eval cases. That's a real *system property* — the system is built to observe itself and feed its own improvement. But the *operating* of that loop over time — detecting drift, running online evaluation, triaging a 👎 down to the broken box, promoting human-verified cases (never the model's wrong output) into the eval set — is deliberately **out of scope here** and covered in the **Monitoring, Drift & Continuous-Improvement** extension chapter (and the [Chapter 7](07-evaluation.md) feedback-loop graft). This Part's job ends at *emit the tree and tag it*; that chapter's job is to run the loop the tree makes possible. The architecture's responsibility is to make the request *observable and attributable*; what you do with that observability over weeks is a different chapter's craft.

- *Build consequence:* Emit **one parent span per request with a child span per box** — router, retrieval, gateway call, each worker — using OTel GenAI attribute names so any backend reads them ([Chapter 8](08-deployment.md)). Set `tenant_id` (and `feature`) **on the parent and propagate it to every child** from day one, so a per-tenant / per-feature cost rollup is a `GROUP BY` over spans, not a forensic reconstruction. Treat the trace-plus-signal as the *input* to the improvement loop, and **forward the loop itself** — drift, online eval, triage — to the Monitoring/Drift chapter rather than building it here.

**Hands-on ([Part F](#part-f--streaming-architecture--end-to-end-observability)):** Take the compound system you wired in Parts C/D and make it observable, then make one path streamable and cancellable.

1. **Emit the span tree.** Wrap each request in a **parent span**, and emit a **child span** for the router/classifier, the retrieval, each gateway model call, and any worker calls. Use the [Chapter 8](08-deployment.md) OTel GenAI attribute names (`gen_ai.request.model`, `gen_ai.usage.input_tokens`/`output_tokens`, `gen_ai.response.finish_reasons`), or run **Langfuse** or **Phoenix** self-hosted in Docker and use its SDK. The gateway from [Part D](#part-d--the-gateway--provider-abstraction-boundary) is the natural place to emit the per-call token attributes once for both providers.
2. **Tag for tenancy.** Set a fake `tenant_id` on the parent span and propagate it to every child. Drive a **mixed workload** — e.g. 50 requests across `tenant_id ∈ {acme, globex, initech}` with different question mixes (some cache hits, some agent paths).
3. **Roll up the cost.** From the captured spans, produce a **per-tenant cost rollup**: sum `input_tokens` + `output_tokens` per span, price per model, `GROUP BY tenant_id`. Confirm the three tenants have different totals and that they sum to the whole-system spend — that's a number you can act on.
4. **Stream and cancel.** Convert one endpoint (the answerer path) to **stream over SSE**. Then trigger a **client-side cancel** mid-stream (close the connection / abort the fetch) and prove, *from the spans*, that **no further model calls fire** after the cancel — the in-flight provider request aborts and any agent loop stops issuing steps. Compare token usage for a cancelled run vs. a run allowed to finish; the gap is the spend cancellation saved you ([Chapter 5](05-agents.md)).

**Resources**

- Anthropic — [Building effective agents](https://www.anthropic.com/research/building-effective-agents) — the canonical write-up of workflows vs agents and the five orchestration patterns.
- [The compound AI systems shift (BAIR)](https://bair.berkeley.edu/blog/2024/02/18/compound-ai-systems/) — why the system, not the model, is the unit of design.
- Gateways / LLM proxies: [LiteLLM](https://docs.litellm.ai/) and [OpenRouter](https://openrouter.ai/) — managed versions of the Part D boundary.
- Observability: [OpenTelemetry GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/), [Langfuse](https://langfuse.com/), [Arize Phoenix](https://docs.arize.com/phoenix) — the span tree + per-tenant cost from Part F (run in Docker).

**Questions**

*Check your understanding*

1. What is the system-level default: is a real system mostly workflow with autonomous islands, or mostly agent with deterministic bits?
2. Define "agent-washing" and name its three added costs over a workflow.
3. Match each symptom to a pattern: (a) one prompt handles wildly different inputs badly; (b) the first draft isn't good enough; (c) independent sub-tasks run slowly in series.
4. What single thing does a gateway / provider-abstraction boundary turn "which provider/model" into?
5. Name the four caching tiers in safety order, cheapest/safest first.
6. In the guardrail sandwich placed as architecture, what are the two stations and the default fail direction?
7. What single question decides SSE vs. WebSocket as the streaming transport, and which answer maps to which?

*Apply it*

8. On the spine, why does the CLASSIFIER box become the *routing* pattern, and which model tier should it run on?
9. A teammate builds "fetch order → check status → email customer" as an agent. What's the diagnosis and the fix?
10. Your three model boxes each import a provider SDK directly. What breaks, and what does adding the gateway buy?
11. Why is the semantic cache the one tier that needs a risk decision before shipping, unlike prompt-prefix caching?
12. A teammate's RAG endpoint runs ingest → chunk → embed → index inside the request handler. What's wrong and what's the fix?
13. A user hits "Stop" on a long agent answer and the stream disappears from the screen, but next month's bill is unchanged for those stopped requests. What's wrong and how does the architecture fix it?
14. You have one provider invoice of $4,000 but can't tell which customer or feature drove it. What one change to your spans makes the bill sliceable, and why must it be designed in early?

*Stretch*

15. Orchestrator-workers vs a full agent vs multi-agent — state the distinguishing axis and where each lands on it.
16. You add an evaluator-optimizer to a path. How do you keep it from becoming an agent, and why is the evaluator "free" to build?
17. Where does routing-as-config (the gateway) end and the routing *pattern* (Part C) begin — and how do they compose?
18. The gateway "unlocks" fallback chains, but the chapter says the deep treatment lives elsewhere. What belongs here vs there?
19. Ch8 already gave you parent/child spans for one endpoint. What does generalizing to a compound system actually add, and where do you draw the line on what belongs in *this* chapter vs. the Monitoring/Drift chapter?

**Answer key** — *peek only after attempting.*

1. Mostly **deterministic workflow** — you own the control flow end to end — with at most a few **autonomous islands** where the path can't be predetermined. Autonomy is the justified exception, never the substrate.
2. Wrapping a *knowable* pipeline in an agent loop because it sounds advanced. Over a workflow it adds **non-determinism** (path varies per run), **untestability** (no fixed stage to probe), and **runaway spend** (an unbounded loop needs step/cost guardrails you only need because you added autonomy you didn't need).
3. (a) **Routing**; (b) **Evaluator-optimizer**; (c) **Parallelization — sectioning**.
4. Configuration behind one stable `call(messages, model)`-shaped interface — a string arg + a model table — instead of provider SDK calls hard-coded across your app.
5. Prompt-prefix cache → embedding cache → exact-match response cache → semantic cache; the first three are deterministic (a hit equals a recompute), only the semantic cache can serve a near-miss.
6. An input station before the model boxes and an output station after them; default fail-safe (block/withhold on error), fail-open only for a consciously low-stakes path.
7. Does the client only *receive*, or also *send* mid-stream? One-way token push → SSE (one-way HTTP, rides existing gateway/auth); interactive/bidirectional or voice (client sends during the stream) → WebSocket. Default to SSE; step up only when you can name the mid-stream message.
8. The CLASSIFIER turns a message into a routed intent, then your code dispatches to a specialized downstream prompt — that *is* routing (classify-then-dispatch). Labelling is easy, so it runs on the **cheap/fast tier** ([Ch 8](08-deployment.md) model tiering); only the hard downstream answer pays for the capable tier.
9. **Agent-washing** — the three steps are knowable in advance, so it's a **workflow** (a fixed three-call chain). Fix: build it as a deterministic chain; you drop the non-determinism, get testable stages, and shed the runaway-spend exposure (no step/cost guardrails needed).
10. Every Ch2 call-shape difference (system placement, reply path, usage names, stop reason) is hard-coded in three places, so swapping a provider is a code change. The gateway makes fallback chains, model tiering, and build-vs-buy all config-level edits attached to one seam.
11. Prompt-prefix/embedding/response caches only ever return byte-identical results, so a hit can't be wrong; the semantic cache serves the answer to a *similar* question, so a loose threshold returns a confidently wrong answer to a subtly different question — it needs a threshold, TTL, and a cacheable-class allowlist.
12. It conflates the write/maintenance path with the read path, re-embedding the whole corpus on every question — slow, expensive, pointless. Split the data plane into a batch service triggered by new docs / model change that builds the index off the request path; the request handler only calls `retrieve(question)` against the existing index.
13. The transport hid the remaining tokens but didn't *cancel* server-side, so the agent kept looping and the gateway kept calling the provider — you paid for tokens the user rejected (per-step cost, Ch5). Fix: propagate cancellation end-to-end — client disconnect → handler observes it → cancels the in-flight task → gateway aborts the provider call → agent loop stops. A stopped-but-still-spending stream is a cost bug to test for.
14. Tag every span with `tenant_id` (and `feature`) and propagate from parent to all children; each span already carries its own token usage, so a per-tenant cost rollup is just sum(input+output tokens) priced per model, `GROUP BY tenant_id`. Design it early because the tag is set when the parent opens and must flow to every child — retrofitting tenant tags onto an already-instrumented system is the painful path.
15. Axis: **who owns control flow, and is the loop bounded?** Orchestrator-workers — *you* own the loop, workers are **bounded single calls** (a fixed `for` over a planned split). Full agent — the **model** owns an **open-ended** loop (needs step/cost guardrails). Multi-agent — several such model-owned loops; premature until sub-tasks are truly independent and one agent provably can't do it.
16. **Bound the revise loop** (one or two passes, never open-ended) — you own the loop and cap it, so the model never decides when to stop. The evaluator is "free" because it's [Chapter 7](07-evaluation.md)'s **LLM-as-judge** (the same faithfulness judge) used **inline** in the request path instead of offline on a test set — no new mechanism.
17. The Part C router classifies the request and picks *which path/box* handles it; the gateway picks *which model* each box's call uses, via the model table. They stack: the router decides the path, every model call on that path goes through `llm_call`, so no downstream code names a provider — routing of work and routing of model are two seams that compose cleanly.
18. Here (architecture): *that* there is one seam under every model box, and a fallback chain attaches to it so primary→secondary is one wrapper. There (Ch8 Part I / Resilience graft): the *operated* stack — timeout → backoff+jitter retry → circuit breaker → fallback in order, plus the forced-failure test that proves it. Placed here, operated there.
19. It adds: (1) the tree spans the *whole architecture* — router→retriever→gateway→workers, each box a child span mirroring the diagram, so "which box failed" (the Part-A cracks) is answered in the trace; (2) per-tenant/per-feature cost attribution as a system property. The line: this chapter's job is to *emit* the tagged tree and capture the user signal — i.e. make the request observable and attributable. *Operating* that data over time — drift detection, online eval, triaging a 👎 to a box, promoting human-verified eval cases — is the Monitoring/Drift extension chapter (and the Ch7 feedback-loop graft), not here.

**Deliverable:** one **architecture diagram** for a real feature (start from the Part A support-assistant spine, or your own) — every box labelled deterministic-workflow or autonomous-agent, each orchestration step tagged with its pattern (Part C), a gateway under every model box (Part D), the caching / guardrail / data-plane layers drawn as named components (Part E), and the streaming + span-tree threaded through (Part F) — with every box justified in one line by the failure it prevents or the cost/latency it buys. **Plus** a thin **compound-system skeleton in code** that wires your earlier-chapter artifacts (`retrieve()`, the agent, `tracked_call`) behind clean interfaces, with a router (Part C) calling everything through your gateway (Part D).

**Daily update:** one line — what you designed/wired and any open question (e.g. "drew the support-assistant architecture: classifier+retriever+answerer+validator as a workflow, zero agents; router → gateway with an OpenAI fallback; two-tier cache (exact + semantic); per-tenant spans rolling up cost — open question: where the ingestion pipeline's reindex trigger should live").

**Time:** two sessions. Session 1: Parts A–C (decompose into components; the workflow-vs-agent decision; the five orchestration patterns). Session 2: Parts D–F (the gateway boundary; caching/guardrails/data-plane; streaming + observability) plus the architecture diagram + skeleton deliverable.
