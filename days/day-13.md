# Day 13 — Customization, part 1 — the adaptation ladder

> [← Day 12](day-12.md) · [All days](README.md) · [Day 14 →](day-14.md)

**Module:** Customization  ·  **Time:** ~2.5 hrs

## About this module

### Chapter 6 — Customization: adapting a model to your task (without training one)

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

## Interview drill (self-test)
Real Gen-AI interview questions on today's material — *answer from memory, then check against what you just read.*

1. What is RLHF?
2. What is quantization?
3. What is QLoRA (or QLoRA)?
4. When to use LoRA vs QLoRA?
5. How do you fine-tune a model?
6. Did you fine-tune the model? How?
7. How do you do fine-tuning on an LLM?
8. How do you fine-tune BERT for NLP tasks?
9. What is fine-tuning and how does it work?
10. Describe your experience with fine-tuning.
11. What are the steps for fine-tuning an LLM?
12. What is fine-tuning in the context of LLMs?
13. What is prompt engineering and fine-tuning?
14. How to fine-tune models using Hugging Face?
15. What is the process of fine-tuning a model?

_(48 matched; 15 shown — more in [`ai-eng-questions.md`](../ai-eng-questions.md).)_

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
