## Chapter 6 — Customization: adapting a model to your task (without training one)

**Goal:** Learn the full ladder of ways to make a model better at *your* task, and — given a real
problem — pick the right rung. By the end you can look at "the model isn't good enough" and
confidently say *which* fix it needs: a better prompt, retrieval, fine-tuning, or nothing at all.
The deliverable is a **decision**, not a trained model.

**Why this matters:** "The model isn't good enough at our task" is one complaint with many fixes
that differ in cost by **orders of magnitude** — from a five-minute prompt edit to a multi-week
data-and-training project. The expensive, classic mistake is reaching for fine-tuning *first*. This
section is the framework that stops you from spending a month training a model to do something a
better system prompt or a RAG pipeline would have done in an afternoon. Consistent with this whole
program: we're building *on top of* models, so we learn fine-tuning **deep enough to decide and to
brief a vendor — not to run training**.

> **Setup assumed:** no new keys or infrastructure. This block is mostly judgment plus a small
> data-prep exercise. We'll *look at* the shape of fine-tuning data and the provider APIs
> conceptually, but **run no training job** — the skill is choosing correctly, not operating a
> trainer.

**Suggested split:** Session 1 = Parts A–C (the adaptation ladder, what fine-tuning really is, the
modern landscape); Session 2 = Parts D–E (data — the thing that actually decides success — and the
decision/measurement framework), plus the deliverable.

---

## Part A — The adaptation ladder (the decision framework)

**1. There's a ladder of adaptation techniques, cheapest to costliest — always climb from the
bottom.**
Four rungs, in the order you should *try* them:
1. **Prompt engineering** ([Chapter 3](03-prompt-engineering.md)) — minutes, no data, instant iteration. Change the instructions.
2. **RAG** ([Chapter 4](04-rag.md)) — hours to days. Put the right *facts* in context at runtime.
3. **Fine-tuning** — days to weeks: collect data, train, host, version, evaluate. Change the model's
   *default behavior*.
4. **Pretraining a foundation model** — months and millions of dollars. **You will never do this.**
   It's on the ladder only so you know where the top is and why everything below it exists.
- *Build consequence:* Start at rung 1 and only climb when the cheaper rung genuinely can't solve it.
  Most "we need to fine-tune" instincts are solved one or two rungs down for a fraction of the cost.
  The discipline is *earning* your way up the ladder, not jumping to the top.

**2. The split that governs everything: RAG adds *knowledge*; fine-tuning changes *behavior*.**
This is the single most important distinction in the section, and the one people get wrong:
- **A knowledge gap** — "the model doesn't *know* our refund policy / this customer's data / last
  week's release notes." → **RAG.** Fine-tuning is the wrong tool: it's bad at absorbing facts, and
  any fact you bake in is frozen and instantly stale.
- **A behavior gap** — "the model knows plenty, but it won't *consistently* answer in our house tone
  / always emit exactly this JSON shape / reliably classify into our 8 categories." → **fine-tuning
  is a candidate** (after you've tried prompting). It shifts the model's *defaults*.
- *Build consequence:* Diagnose the gap before picking a tool. Ask: "Does the model lack
  *information*, or lack *the right behavior*?" Information → RAG. Behavior/style/format → prompt
  first, fine-tune if prompting plateaus. Picking the wrong axis wastes the entire effort.

**3. Why fine-tuning is rarely the *first* answer.**
Even when the gap is genuinely behavioral, prompting usually gets you most of the way: a sharp
system prompt + a few good few-shot examples ([Chapter 3](03-prompt-engineering.md)) often matches what people *assume* requires
fine-tuning — with zero data collection, zero training, and instant iteration. Fine-tuning earns its
place only when (a) prompting has clearly plateaued, (b) you need the behavior at a consistency or
scale prompting can't hit, or (c) you want to bake the behavior in so you can *drop* the long prompt
and save tokens on every call.
- *Build consequence:* Treat fine-tuning as an *optimization you graduate to*, with evidence that the
  cheaper rungs fell short — not a default. "Have you tried a better prompt and a few examples?"
  should be answered *yes, and here's the eval showing it wasn't enough* before anyone trains
  anything.

---

## Part B — What fine-tuning actually is (build-deep, not train-deep)

**4. Fine-tuning = continue training a pretrained model on *your* labeled examples to shift its
defaults.**
You don't start from scratch (that's pretraining). You take an existing model that already knows
language and the world, and show it a few hundred-plus examples of *your* task done the way you want
— input → ideal output, repeated — so its default behavior moves toward your examples. After tuning,
it produces your preferred style/format/decision *without* being told in the prompt every time.
- *Build consequence:* Think of it as "teaching by example at the weights level" rather than
  "teaching by instruction in the prompt." The product is a *new model version that behaves
  differently by default* — which is exactly why it's a heavier commitment than editing a string.

**5. What fine-tuning is genuinely good at — and what it can't do.**
- **Good at:** a *consistent narrow behavior* — a fixed output format/schema, a specific tone or
  brand voice, a well-defined classification or extraction task, following a niche instruction style
  reliably. Tasks where you can show "for inputs like this, produce outputs like that," over and over.
- **Bad at / can't do:** injecting *fresh or factual knowledge*. Fine-tuning doesn't reliably
  memorize facts, and whatever it does absorb is frozen at training time and goes stale — you'd have
  to retrain to update it. **That's RAG's job, not fine-tuning's.** It also won't make a model
  fundamentally smarter or fix a capability the base model lacks.
- *Build consequence:* If your wish is "know our latest docs," fine-tuning will disappoint and decay;
  use RAG. If your wish is "always behave *this way*," fine-tuning can deliver. Match the tool to the
  kind of gap (concept 2), restated where it bites hardest.

**6. The cost reality — why it's a commitment, not a tweak.**
Fine-tuning isn't one action; it's a small project with ongoing weight:
- **Data prep** ([Part D](#part-d--data-the-thing-that-actually-determines-success)) — collecting, cleaning, formatting, and splitting examples. This is most of
  the work.
- **The training run** — time and compute cost (managed by the provider, but not free or instant).
- **Hosting & versioning** — you now own a *custom model artifact*: deploy it, track which version is
  in prod, re-evaluate it when the base model updates, retrain when your needs drift.
- **Evaluation** — you can't know it helped without an eval set ([Part E](#part-e--deciding--measuring)).
- *Build consequence:* Budget for the *lifecycle*, not just the training run. A fine-tuned model is
  something you maintain. That maintenance cost is a big part of why "try the cheaper rung first" is
  the rule.

---

## Part C — PEFT & the modern landscape (so the words mean something)

**7. Full fine-tuning vs. PEFT/LoRA — why almost everyone uses the efficient kind.**
- **Full fine-tuning** updates *all* of the model's weights. Maximally flexible, but expensive to
  train and store — you end up with a whole separate copy of a giant model.
- **PEFT (parameter-efficient fine-tuning), of which LoRA is the popular kind** freezes the base
  model and trains a *small* set of extra parameters layered on top. You get most of the benefit at a
  fraction of the compute and storage, and you can keep many small task-specific adapters around one
  shared base.
- *(That's the depth you need:* full = move everything; PEFT/LoRA = train a small add-on. You don't
  need the internal math to make the decision or talk to a vendor.)*
- *Build consequence:* When a fine-tuning product or vendor says "LoRA / adapters / PEFT," hear "the
  cheap, practical kind of fine-tuning." It's why fine-tuning is far more accessible than it was —
  and what you'll almost always actually use.

**8. What a provider fine-tuning API gives you — conceptually.**
The hosted path (e.g. OpenAI's fine-tuning API; Anthropic offers tuning for select models/enterprise)
hides the machinery: you **upload a dataset** (JSONL of example conversations, [Part D](#part-d--data-the-thing-that-actually-determines-success)), **start a
tuning job**, and get back a **new model id** you call exactly like any other model. You don't manage
GPUs or training loops. (Key/billing handling is out of scope here, per the program.)
- *Build consequence:* For most teams, fine-tuning is "prepare a good dataset, submit it, get a model
  id back." That means **your leverage is almost entirely in the data** — which is why [Part D](#part-d--data-the-thing-that-actually-determines-success) is the
  real work and the rest is plumbing.

**9. Distillation — the cost play worth knowing by name.**
A common production pattern: use a big, expensive model to generate high-quality outputs, then
**fine-tune a smaller, cheaper model to imitate them.** You get most of the big model's quality on
your specific task at a fraction of the per-call cost and latency. This is "distilling" the big
model's behavior into a small one.
- *Build consequence:* When a task is narrow, high-volume, and cost-sensitive, distillation is often
  the winning move: prototype with the big model, then fine-tune a small one to do that one job
  cheaply at scale. Know it exists so you can reach for it when the bill matters.

---

## Part D — Data: the thing that actually determines success

**10. Fine-tuning is only as good as its examples — data quality *is* the project.**
The model learns to imitate exactly what you show it, so the dataset is the spec. Examples are
typically formatted as **conversations in JSONL** (one example per line): the input (and any system
prompt) plus the *ideal* assistant output, in the same shape you'll use at inference. Garbage,
inconsistent, or contradictory examples produce a garbage model — faithfully.
```jsonl
{"messages":[{"role":"system","content":"You are a support classifier."},{"role":"user","content":"My card was charged twice"},{"role":"assistant","content":"{\"category\":\"billing\",\"urgency\":\"high\"}"}]}
{"messages":[{"role":"system","content":"You are a support classifier."},{"role":"user","content":"How do I change my avatar?"},{"role":"assistant","content":"{\"category\":\"account\",\"urgency\":\"low\"}"}]}
```
- *Build consequence:* Every assistant message in your dataset is a *vote* for how the model should
  behave. Inconsistency between examples (two near-identical inputs with different output styles)
  actively teaches the model to be inconsistent. Curate ruthlessly.

**11. Quantity: a few hundred *great* examples beat thousands of sloppy ones.**
You usually need far fewer examples than people expect — often on the order of **50–few hundred**
high-quality, consistent examples to shift a narrow behavior, not tens of thousands. Quality and
consistency dominate raw count: a small, clean, on-spec dataset outperforms a large noisy one. Past a
point, more examples give diminishing returns.
- *Build consequence:* Don't let "we don't have a huge dataset" block you — and don't let "we have
  tons of data" tempt you into dumping it in unfiltered. Invest in a few hundred examples you'd be
  proud of, not a giant pile you haven't read.

**12. The train/eval split — never grade on the examples you trained on.**
Hold back a portion of your examples as an **eval set the model never trains on**. After tuning, you
measure performance on that held-out set to see if it actually learned the behavior *generally* vs.
just memorized the training examples. This is non-negotiable and the same discipline you'll formalize
in [Chapter 7](07-evaluation.md).
- *Build consequence:* Without a held-out eval set you can't distinguish "the model learned the task"
  from "the model memorized the answer key." Build the split *before* you train, and treat the eval
  set as sacred — don't peek, don't train on it.

---

## Part E — Deciding & measuring

**13. The decision tree — run it before anyone trains anything.**
1. **Is it a knowledge gap?** (model lacks information) → **RAG.** Stop.
2. **Is it a behavior/format/style gap?** → first, **engineer the prompt + few-shot** ([Chapter 3](03-prompt-engineering.md)). Did
   that solve it? → **Stop, you're done.**
3. **Did prompting plateau** *and* do you have (or can you build) a few hundred consistent examples
   *and* is the task narrow and stable enough to be worth maintaining a custom model? → **fine-tune.**
4. **Is it a one-off or low-volume task?** → just prompt; the fine-tuning lifecycle isn't worth it.
- *Build consequence:* Most paths terminate *before* fine-tuning. Fine-tuning is the answer only at
  the specific intersection of "behavioral gap + prompting wasn't enough + good data exists + the task
  justifies ongoing maintenance." If any of those is missing, climb back down the ladder.

**14. You cannot tell if fine-tuning helped without an eval set.**
"It seems better" is not a result. You need the same `(input, expected)` eval harness from [Chapter 4](04-rag.md) /
[Chapter 7](07-evaluation.md): score the *base model with your best prompt* against the *fine-tuned model* on the
held-out set, with a real metric. Often the honest finding is "the well-prompted base model was
already good enough" — which saves you the entire fine-tuning lifecycle. That's a *win*, not a
failure.
- *Build consequence:* The eval set is what turns "should we fine-tune?" from an argument into a
  measurement. Build it first; it both *justifies* the decision and *proves* the result. This is the
  direct bridge into **[Chapter 7](07-evaluation.md)**.

**15. The common production combo: fine-tune for *behavior*, RAG for *facts*.**
These aren't competitors — the strongest systems often use both. Fine-tune a model so it *reliably
behaves* the way you need (tone, format, decision style), and use RAG to feed it *current facts* at
runtime. Each handles the gap it's actually good at: tuning owns behavior, retrieval owns knowledge.
- *Build consequence:* "RAG vs. fine-tuning" is usually a false choice. The senior answer to many
  real problems is "fine-tune the *how*, retrieve the *what*" — combine them along the
  knowledge/behavior split from concept 2.

---

**Resources**
- OpenAI — fine-tuning guide (the dataset JSONL format, when-to-use guidance) and the distillation
  docs.
- Anthropic — guidance on prompting-before-fine-tuning and where tuning fits; the RAG-vs-fine-tuning
  framing.
- A short read on **LoRA/PEFT** at the conceptual level (what an adapter is, why it's cheap) — enough
  to follow vendor docs, not to implement it.
- Your own [Chapter 3](03-prompt-engineering.md) (prompting), [Chapter 4](04-rag.md) (RAG) — the two rungs you must rule out before fine-tuning.

**Hands-on tasks**
1. **Diagnose the gap (×4):** for each scenario, label it *knowledge* vs *behavior* and name the rung
   you'd try first — (a) "the bot doesn't know our 2024 pricing"; (b) "the bot's answers don't match
   our terse brand voice"; (c) "we need every reply as strict JSON for our pipeline"; (d) "the bot
   can't answer questions about a customer's specific account."
2. **Climb the ladder:** take scenario (b) or (c) and write the *prompt-engineering* attempt first
   (system prompt + 2–3 few-shot examples). Argue whether it plausibly solves it without any tuning.
3. **Feel the data shape:** hand-write a **10-line JSONL** fine-tuning dataset for a small
   classification or formatting task (use the concept 10 shape). Make the examples *consistent* with
   each other on purpose.
4. **Break consistency:** in your 10-line set, deliberately make two near-identical inputs have
   differently-styled outputs. Write one sentence on what that would teach the model.
5. **Split it:** take your dataset and mark a held-out eval portion. State how you'd compare "base +
   good prompt" vs "fine-tuned" on it.
6. *(Stretch)* **Distillation plan:** for a high-volume narrow task, sketch how you'd use a big model
   to generate training data for a small fine-tuned model, and estimate where the cost savings come
   from.

**Questions**

*Check understanding*
1. List the four rungs of the adaptation ladder, cheapest to costliest. Which one will you never do,
   and why is it on the list?
2. State the core split in one sentence: what does RAG add, and what does fine-tuning change?
3. Give one thing fine-tuning is good at and one thing it genuinely can't do.
4. Why is fine-tuning rarely the *first* fix for a behavioral gap?
5. In one line each: full fine-tuning vs. PEFT/LoRA.
6. What does a hosted fine-tuning API take in and hand back?
7. Why is a held-out eval set non-negotiable when fine-tuning?

*Apply it*
8. "Our support bot doesn't know last week's policy change." RAG or fine-tune? Why?
9. "Our bot's tone is inconsistent and we've already tried three prompts and few-shot." What's the
   next step, and what do you need before taking it?
10. A teammate wants to fine-tune a model to "know all our documentation." Explain why this will
    disappoint and what to do instead.
11. You have a narrow, high-volume classification task and the per-call cost of the big model hurts.
    What technique fits, and where do the savings come from?
12. Someone says "we have 100,000 logged conversations, let's fine-tune on all of them." What's your
    concern, and what would you do instead?

*Stretch*
13. Walk the full decision tree for: "We need replies in our brand voice *and* grounded in our current
    pricing." What's the right architecture and why isn't it one technique?
14. After fine-tuning, your eval shows the base model with a good prompt was just as good. Is the
    project a failure? What do you ship, and what did you gain?
15. Explain why "RAG vs. fine-tuning" is usually a false dichotomy, using the knowledge/behavior split.

**Answer key**
1. Prompt engineering → RAG → fine-tuning → pretraining. You'll never pretrain (months, millions of
   dollars, needs a foundation-model effort); it's on the list to mark the top of the ladder and show
   why the cheaper rungs exist.
2. RAG adds *knowledge* (facts injected into context at runtime); fine-tuning changes
   *behavior/style/format* (the model's defaults).
3. Good at: a consistent narrow behavior — fixed format/schema, tone, a defined
   classification/extraction task. Can't do: reliably inject fresh/factual knowledge (frozen + stale →
   that's RAG).
4. Because a sharp prompt + few-shot usually gets most of the way with zero data, zero training, and
   instant iteration; fine-tuning is the slower, heavier optimization you graduate to only after
   prompting plateaus.
5. Full = update *all* the model's weights (expensive, a full model copy). PEFT/LoRA = freeze the
   base, train a small add-on/adapter (cheap, most of the benefit).
6. In: a dataset (JSONL of example conversations with ideal outputs). Out: a new model id you call
   like any other model. It hides the GPUs/training loop.
7. Without held-out examples you can't tell "learned the behavior generally" from "memorized the
   training set"; the eval set is the only way to measure real improvement (and to compare against the
   prompted base).
8. RAG — it's a knowledge gap and the fact changes over time; fine-tuning would bake in a stale fact
   and need retraining to update.
9. Fine-tuning is now a candidate — but first build a few hundred consistent examples *and* an eval
   set, and confirm the task is narrow/stable enough to justify maintaining a custom model.
10. Fine-tuning doesn't reliably memorize facts and freezes whatever it learns (instantly stale); use
    RAG to retrieve current docs at runtime instead.
11. Distillation — use the big model to generate training data, fine-tune a small cheap model to
    imitate it; savings come from far lower per-call cost/latency of the small model at high volume.
12. Quantity isn't the goal and unfiltered logs are inconsistent (teaching inconsistency); curate a
    few hundred clean, consistent, on-spec examples and hold out an eval set rather than dumping all
    100k.
13. Knowledge gap (current pricing) → RAG; behavior gap (brand voice) → prompt first, fine-tune if it
    plateaus. The right architecture is *both*: fine-tune the *how* (voice/format), retrieve the *what*
    (pricing) — they address different axes (concept 2), so no single technique covers it.
14. Not a failure — it's a measured result that saves you the entire fine-tuning lifecycle (training,
    hosting, versioning, maintenance). Ship the well-prompted base model; you gained the *evidence*
    that you don't need a custom model.
15. Because they fix different gaps: RAG supplies knowledge, fine-tuning shapes behavior. Many real
    problems have *both* gaps, so the strong answer combines them (fine-tune behavior + retrieve facts)
    rather than choosing one.

**Deliverable:** a written **adaptation decision memo** for 3–4 realistic scenarios from your own
work: for each, state the gap type (knowledge vs behavior), the rung you'd choose and *why*, what (if
anything) you'd try first, what data you'd need, and how you'd measure success. **Plus** one small
hand-written JSONL dataset (~10 consistent examples) for a chosen task, with a held-out eval split
marked — to show you understand the data shape. **No training run required;** the artifact is the
reasoning.

**Daily update:** one line — the decision you reached for your scenarios and any open
question (e.g. "mapped 4 use cases to the ladder: 3 are RAG/prompt, 1 is a fine-tune candidate
pending an eval set; drafted a 10-line JSONL for the classifier").

**Time:** two sessions. Session 1: Parts A–C (the ladder, what fine-tuning is, the landscape). Session 2:
Parts D–E (data + the decision/measurement framework) and the deliverable memo.

---

