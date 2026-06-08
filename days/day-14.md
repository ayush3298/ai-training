# Day 14 — Customization, part 2 — data & deciding

> [← Day 13](day-13.md) · [All days](README.md) · [Day 15 →](day-15.md)

**Module:** Customization  ·  **Time:** ~2.5 hrs

## Where we are

_Continues **Customization**. Earlier days covered Parts A, B, C; today picks up where they left off._

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

---

## Module wrap-up — hands-on, questions & deliverable

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

---

## Interview drill (self-test)
Real Gen-AI interview questions on today's material — *answer from memory, then check against what you just read.*

1. What is QLoRA?
2. What is transfer learning?
3. How do you fine-tune an LLM?
4. When do we use LoRA and QLoRA?
5. What does fine-tuning models mean?
6. How do you version datasets and models?
7. How do you handle an imbalanced dataset?
8. What is PEFT? How do LoRA and QLoRA work?
9. What are synthetic datasets / synthetic DB?
10. How would you handle outliers in a dataset?
11. Explain fine-tuning and prompt engineering.
12. How to identify overfitting on training data?
13. What approaches exist for fine-tuning an LLM?
14. How to identify underfitting on training data?
15. If a dataset has missing values, what will you do?

_(48 matched; 15 shown — more in [`ai-eng-questions.md`](../ai-eng-questions.md).)_

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
