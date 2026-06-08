# Day 15 — Evaluation, part 1 — metrics & LLM-as-judge

> [← Day 14](day-14.md) · [All days](README.md) · [Day 16 →](day-16.md)

**Module:** Evaluation & Safety  ·  **Time:** ~2.5–3 hrs

## About this module

### Chapter 7 — Evaluation, Safety & Reliability: shipping with confidence, not vibes

**Goal:** Build the discipline that makes an LLM feature trustworthy. By the end you can assemble a
real eval set, *choose and compute the right metric for each task type* (and know what each one
actually measures, from first principles), automate grading including LLM-as-judge, catch
regressions before users do, wrap the system in input/output safety guardrails, and answer the only
question that matters before shipping: *"is this good enough, and how do you **know**?"*

**Why this matters:** LLMs are non-deterministic and **fail silently** — the same prompt gives a
great answer today and a confidently wrong one tomorrow, with nothing flashing red. "It worked when
I tried it" is exactly how broken features reach production. Every earlier chapter ended on the same
unfinished sentence — *you can't tell if retrieval helped ([Chapter 4](04-rag.md)), if the agent went off the rails
([Chapter 5](05-agents.md)), or if fine-tuning was worth it ([Chapter 6](06-customization.md))* — **without an eval set and the right
metric**. This is that discipline. It's the single skill separating engineers who *hope* their
system works from those who can *prove* it with a number, defend a change, and sleep after a deploy.

> **Setup assumed:** same as before. We evaluate things you already built — your [Chapter 4](04-rag.md) RAG system
> and/or your [Chapter 5](05-agents.md) agent. Metrics are mostly plain arithmetic (we compute several by hand); the
> judge is just another model call ([Chapter 2](02-apis-and-integration.md)). No new infrastructure.

---

## Part A — Why eval is the core skill (and why "it looks good" fails)

**1. Non-determinism + silent failure = you cannot eyeball quality at scale.**
Two properties make LLM features uniquely hard to trust. **Non-determinism:** identical input can
produce different output run to run ([Chapter 1](01-foundations.md)'s sampling), so "it worked once" proves nothing about the
next call — you're measuring a *distribution* of behavior, not a fixed function. **Silent failure:**
a wrong answer looks exactly like a right one — fluent, confident, well-formatted ([Chapter 1](01-foundations.md)'s
hallucination, [Chapter 4](04-rag.md)'s retrieval miss). Normal code throws when it breaks; an LLM just *says
something wrong* with a straight face. You cannot catch that by skimming a handful of outputs, and
your eye is biased toward the cases you happened to try.
- *Build consequence:* Manual spot-checking neither scales nor catches the failures that matter. You
  need a *systematic, repeatable, quantified* way to measure quality — the same reason you write
  automated tests for code you can't verify by reading.

**2. The eval set is the unit of truth — it's a test suite for behavior.**
An eval set is a curated collection of **`(input, expected)` pairs** — representative inputs paired
with what a good output looks like (or a rule for judging one). Running your system over the set and
scoring the results produces a **number** that stands in for "how good is this." It's the LLM
analogue of a unit-test suite: the thing you run to know whether a change helped or hurt *before* a
user finds out. The score is only as honest as (a) the cases you chose and (b) the metric you scored
them with — which is why Parts B and C exist.
- *Build consequence:* "Do we have an eval set?" is the first question to ask of any LLM feature
  headed for production. No eval set = shipping on vibes, and every "improvement" is a guess.
  Building it *is* the engineering, not overhead around it.

**3. Offline vs. online eval — two stages, both needed.**
- **Offline eval:** run your system over a fixed dataset *before* shipping, in development/CI. Fast,
  repeatable, controlled — where you compare versions and catch regressions.
- **Online eval:** measure quality on *real* production traffic after shipping — sampling live
  outputs, collecting user signals (thumbs, edits, escalations), watching error/refusal rates.
  Catches the real-world inputs your offline set never imagined.
- *Build consequence:* Offline eval gates the deploy; online eval reveals what offline missed and
  *feeds back into* the offline set ([Part F](#part-f--reliability-in-production)). They're a loop, not a choice — build offline first,
  then close the loop with online once live.

---

## Part B — The metrics catalog, from first principles

A metric is a function that turns `(output, expected)` into a number you can average. But *which*
function? Every metric below exists because a simpler idea broke. We derive each one by finding the
crack in the thing before it.

**The root question: "is the output good?" — and why it's not one question.** The only thing you
actually want to know is *is the system producing good output?* The trouble: "good" means different
things depending on what the output **is**, and each shape of output breaks a different naive metric.
So the first act of first-principles thinking is **refusing to answer "is it good?" in general** and
instead asking "what *kind* of output is this, and what would *wrong* look like?" The whole catalog
falls out of that one split.

**4. Exact match — the metric you get for free, and the moment it breaks.**
*First principle:* if there is exactly *one* correct output and you know it, then "good" = "equals
the correct output." Compare strings, score 1 or 0, average over the set. This is the baseline every
other metric is a *repair* of — perfect for a label, an ID, a number, a JSON shape. **Where it
cracks:** the instant there's more than one acceptable output. `"Paris."` vs `"Paris"` vs `"The
capital is Paris"` are all correct but fail exact match. First repair is cheap — **normalize**
(lowercase, strip punctuation/whitespace) before comparing. The *second* crack is fatal: for
open-ended text there is no single correct string at all, so equality is the wrong tool entirely.
- *Build consequence:* Always start here. If your output reduces to "there's a right answer and I can
  check it," you're done — exact/structural checks are exact, instant, free. Every fancier metric is
  a tax you pay only because exact match *can't* apply.

**5. Classification: why "how often is it right?" (accuracy) is a trap.**
Now the output is a **category** ("spam / not spam", "fraud / legit", "billing / account /
technical"). The naive metric writes itself: **accuracy = fraction of cases it got right.** It feels
obviously correct — and it's the single most misleading number in evaluation. *The crack:* accuracy
treats every error as equally bad and every case as equally common; both are usually false. If **1
in 100** transactions is fraud, a model that always says "legit" is **right 99% of the time** —
99% accuracy, zero fraud caught. The number is high precisely *because* it ignores the rare thing you
built the system to find. Accuracy **collapses two genuinely different mistakes into one number, then
lets the common case dominate.** To fix it we must *stop collapsing* — which forces the confusion
matrix.

**6. The confusion matrix — the foundational object all classification is built on.**
*First principle:* there are only four things that can happen when a yes/no system meets a yes/no
reality. Cross "what the model said" with "what's true":

|                | model says YES | model says NO |
|----------------|----------------|---------------|
| **truly YES**  | **TP** — caught it | **FN** — missed it |
| **truly NO**   | **FP** — false alarm | **TN** — correctly ignored |

These four counts are the *complete* description of a classifier — every classification metric is
just a different ratio of them, chosen to answer a specific question. The two that matter most are
the two ways to be wrong: **FP (false alarm)** — shouted "yes" when truth was "no"; **FN (miss)** —
stayed silent when truth was "yes." These are *different real-world costs* (blocking a legit customer
≠ letting fraud through), which is exactly why accuracy (which adds them together) destroys the
information you need.

**7. Precision — "when it says yes, can I believe it?"**
*First principle:* if acting on a positive is *expensive* (every "yes" blocks a transaction, pages a
human, auto-rejects), the question is: **of all the times it cried "yes," how often was it right?**

$$\text{precision} = \frac{TP}{TP + FP} = \frac{\text{correct alarms}}{\text{all alarms}}$$

Read it as **trustworthiness of a positive prediction.** Precision 0.67 → a third of its alarms were
false. Note what it *ignores*: nothing about missed positives (FN absent). A model that flags one
transaction and is right has **perfect precision** while missing 99 frauds — a deliberate blind spot;
precision answers one question and leaves the other to recall.
- *When it's the metric:* false alarms are costly or erode trust — spam filters, content moderation,
  anything that auto-acts on a "yes."

**8. Recall — "of everything I should have caught, how much did I catch?"**
*First principle:* if *missing* a positive is the expensive thing (undetected fraud, an overlooked
tumor, a threat slipping through), the question flips: **of all the real positives, how many did the
system find?**

$$\text{recall} = \frac{TP}{TP + FN} = \frac{\text{caught}}{\text{all that were really there}}$$

Read it as **coverage of the real positives.** Recall 0.80 → caught 80%, let 20% through. Mirror blind
spot of precision: it ignores false alarms (FP absent). **Perfect recall** by flagging *everything* —
catch every fraud because you flagged every transaction — at the cost of drowning in false alarms.
- *When it's the metric:* misses are costly — disease screening, fraud/threat detection, "did we
  surface the relevant doc at all."

**9. The precision–recall tradeoff — why they fight, derived.**
Not an accident; structural. Most classifiers output a **score** and you pick a **threshold** above
which you say "yes." Slide it: **lower** (flag eagerly) → catch more true positives (**recall ↑**) but
sweep in junk (**FP ↑ → precision ↓**); **raise** (flag only when sure) → fewer false alarms
(**precision ↑**) but borderline real cases slip (**FN ↑ → recall ↓**). So precision and recall are
two ends of one lever — improving one *via the threshold* costs the other. (Pushing *both* up at once
requires a genuinely better model that moves the whole curve — which your eval is there to detect.)
This is *why* you decide, from the real cost of FP vs FN, **which error you'd rather make**, and set
the threshold there on purpose.

**10. F1 — collapsing two numbers into one, without letting either cheat.**
*First principle:* sometimes you need a *single* number (to rank experiments, gate CI), high only
when **both** precision and recall are high. Naive move: plain average `(P+R)/2`. **It breaks:**
precision 1.0, recall 0.0 (flags one item, right, misses everything) → `(1.0+0.0)/2 = 0.5`, a
middling score for a useless model. The arithmetic mean lets one great axis paper over a catastrophe.
Fix: the **harmonic mean**, dominated by the *smaller* value:

$$F_1 = 2 \cdot \frac{P \cdot R}{P + R}$$

Same disaster → `2(1.0·0.0)/(1.0+0.0) = 0`. The harmonic mean *refuses* to be fooled — high F1
requires being good at **both**. That's the entire reason it's harmonic, not arithmetic.

*Full worked example:* 100 transactions, 10 truly fraud. Model flags 12; 8 are real fraud → TP=8,
FP=4, FN=2, TN=86. Precision = 8/12 = **0.67**; Recall = 8/10 = **0.80**; F1 =
2(0.67·0.80)/(0.67+0.80) = **0.73**; Accuracy = 94/100 = **0.94**. Accuracy hides that it misses 1 in
5 frauds *and* a third of its alarms are false — a 0.94 that should not ship.
- *Build consequence:* For classification, **report precision and recall, not accuracy** — and decide
  *which error is worse for your use case* before tuning. "94% accurate" is often a number that lets
  a bad classifier ship.

**11. Retrieval metrics — the new first principle is *order*.**
RAG retrieval ([Chapter 4](04-rag.md)) outputs a **ranked list** of k chunks, so the question changes from "right or
wrong" to **"is the right thing in the list, and how near the top?"** Position matters: the model
attends best to the top of the context, and every slot costs tokens.
- **Hit-rate@k / Recall@k — the existence question.** Did a correct chunk make the top-k *at all*? If
  not, the generator never had a chance. $\text{hit-rate@}k = \frac{\#\text{ questions with a correct
  chunk in top-}k}{\#\text{ questions}}$ (per question 1 or 0, averaged). Your **primary** retrieval
  metric — it measures the precondition for a good answer. *Blind spot:* ignores *where* in the
  top-k it landed.
- **Precision@k — the noise question.** Of the k chunks returned, how many were relevant?
  `relevant_in_topk / k`. Every irrelevant chunk wastes tokens and distracts ([Chapter 4](04-rag.md)'s "lost in the
  middle").
- **MRR (Mean Reciprocal Rank) — the position question.** A correct chunk at rank 1 beats the same at
  rank 5, and value should fall with rank. Cleanest such function: `1/rank` of the *first* correct
  chunk (rank 1 → 1.0, 2 → 0.5, 3 → 0.33, 4 → 0.25), averaged:
  $\text{MRR} = \frac{1}{N}\sum_i \frac{1}{\text{rank of first correct chunk}_i}$. *Worked:* positions
  1, 3, 2 across three questions → (1 + 0.333 + 0.5)/3 = **0.61**. Use when there's ~one right chunk
  and getting it *high* matters.
- **NDCG — graded relevance, discounted by position, normalized.** MRR only sees the first hit and
  treats relevance as binary. NDCG builds on two principles: **gain** (each chunk graded 0/1/2/3) and
  **discount** (contribution divided by `log2(rank+1)`, so deep results count less). Sum →
  **DCG**, then normalize by the ideal ordering (**IDCG**) → `NDCG = DCG/IDCG` (0–1, comparable
  across queries). Use when relevance is a spectrum and overall ordering quality matters.
- *Build consequence:* pick by **how much position matters** — don't care → hit-rate@k; care about the
  top slot → MRR; whole-ranking with graded relevance → NDCG. These isolate the *retrieval* bug from
  the *generation* bug — the most useful split in RAG debugging.

**12. Text-similarity metrics — the hard problem of "two texts, same meaning?"**
Output is **free text** with a reference "gold" answer. You want "does the output match the
reference?" — but *how*? Word-for-word fails (many valid phrasings), so we approximate, and each
approximation is a known compromise.
- **BLEU — overlap as precision.** From translation. A good translation mostly contains word-sequences
  (**n-grams**) the reference also contains → BLEU measures **n-gram precision** (of the output's
  n-grams, what fraction are in the reference; plus a brevity penalty). Punishes *adding* content the
  reference lacks. Good for tight tasks (translation).
- **ROUGE — overlap as recall.** From summarization. A good summary should *cover* the reference →
  **n-gram recall** (of the reference's n-grams, what fraction are in the output). Punishes *missing*
  content. (The precision/recall split from classification, reappearing at the word level.)
- **Embedding similarity — overlap of *meaning*.** Both above count word overlap, so "get my money
  back" and "issue a refund" — same meaning, no shared words — score ~0. Fix: embed output and
  reference ([Chapter 4](04-rag.md)), take **cosine similarity** — different words, same meaning now score high. But
  you're trusting the embedding model's fuzzy notion of "similar."
- **The shared, unfixable limit.** All three compare against **one reference**, but open-ended tasks
  have *many* equally-good answers. A brilliant answer phrased unlike your reference scores low; a
  mediocre answer parroting reference words scores high. The problem is "one reference can't represent
  the space of good answers" — exactly the gap that forces LLM-as-judge ([Part C](#part-c--llm-as-judge-grading-open-ended-quality-at-scale)).
- *Build consequence:* use BLEU/ROUGE/embedding as cheap rough signals, never the sole gate for
  open-ended quality — their foundational assumption (one reference ≈ all good answers) is false there.

**13. RAG quality metrics — decomposing "good answer" into measurable atoms.**
"Is it good?" is too coarse to act on (concept 5's lesson). A RAG answer fails in **independent ways**,
and one score can't separate them — so decompose "good" into orthogonal dimensions, each measurable on
its own (each typically judged by an LLM, [Part C](#part-c--llm-as-judge-grading-open-ended-quality-at-scale)):
- **Faithfulness / groundedness** — *is every claim supported by the retrieved context?* Isolates
  **hallucination**, the worst failure. Extract the answer's claims, check each against context, score
  = supported / total. A confident answer with one invented number scores low even if it reads well.
- **Answer relevance** — *does it actually address the question?* A faithful answer can still answer
  the *wrong* question; this isolates on-topic-ness, independent of truth.
- **Context precision** — *of the chunks retrieved, how many were useful?* (precision@k's idea at the
  generation step: noise indicator.)
- **Context recall** — *did retrieval fetch everything needed?* Coverage: if context lacked a needed
  fact, even a perfect generator can't be complete.

**Why decompose (the payoff):** the four numbers *localize the broken component*, which one number
never can — low **context recall** → retrieval/chunking missed info (fix retrieval); good context but
low **faithfulness** → generator invented things (fix prompt/grounding); low **answer relevance** → it
misread the question (fix prompt/query understanding); low **context precision** → too much noise (fix
chunking/retrieval). [Chapter 4](04-rag.md)'s "debug retrieval before generation," turned into named, trackable numbers
— the metrics *are* the debugger.

**The thread tying the catalog together.** Every metric is the same move: **name the question
precisely, find what the naive metric throws away, build the ratio that keeps it.** Accuracy throws
away which-error-type → the confusion matrix keeps it. Plain average throws away the worse axis →
harmonic mean keeps it. Hit-rate throws away position → MRR keeps it. Word-overlap throws away meaning
→ embeddings keep it. One score throws away *where* RAG broke → decomposition keeps it.
- *Build consequence:* Don't memorize formulas — memorize the *question each metric protects*. Match
  the metric to the failure you most need to catch, prefer the cheapest one that still catches it, and
  report the pair (precision+recall, hit-rate+MRR, the four RAG dimensions) whenever a single number
  would hide a real error.

---

## Part C — LLM-as-judge (grading open-ended quality at scale)

**14. Use a model to grade model output — the only thing that scales for open-ended quality.**
For "is this answer good?" there's no string to match (concept 12's limit), and human grading can't
keep up with hundreds of cases per code change. **LLM-as-judge** uses a (usually strong) model to
score outputs against a rubric — fast, cheap, consistent enough to run on every change. It's what
makes evaluating the things you most care about (helpfulness, faithfulness, tone) tractable at all.
- *Build consequence:* This unlocks the metrics in concept 13. But a judge is itself an LLM —
  non-deterministic and biased — so it must be *engineered and validated* (concepts 16–17), never
  blindly trusted.

**15. Writing a good judge prompt is prompt engineering ([Chapter 3](03-prompt-engineering.md)) pointed at grading.**
A reliable judge needs: **a specific rubric** ("faithful = every factual claim is supported by the
context; no invented details") and a **scale** — *prefer coarse* (binary, or 1–3) over 1–10, since
models score far more consistently on coarse distinctions; **one dimension at a time** (faithfulness,
relevance, completeness separately, not one fuzzy "quality"); **few-shot anchors** (show a top score
and a low score, [Chapter 3](03-prompt-engineering.md)); **reasoning before the score** (CoT makes the verdict more reliable and
auditable); and **structured output** ([Chapter 2](02-apis-and-integration.md)) so you can aggregate.
```python
JUDGE = """You grade an AI answer for FAITHFULNESS to the provided context.
Faithful = every factual claim is directly supported by the context; no invented details.
Score: 1 (fabricated/contradicted), 2 (partly supported), 3 (fully grounded).
First give your reasoning, then the score. Return JSON: {"reasoning": "...", "score": <1-3>}."""

def judge(question, context, answer):
    resp = client.messages.create(
        model="claude-sonnet-4-6", max_tokens=400, system=JUDGE, temperature=0,
        messages=[{"role":"user","content":
            f"<context>{context}</context>\n<question>{question}</question>\n<answer>{answer}</answer>"}],
    )
    return json.loads(resp.content[0].text)   # {"reasoning":..., "score":...}
```
- *Build consequence:* A vague judge ("rate 1–10") yields noisy, unrepeatable numbers you can't
  defend. The judge prompt deserves the same care as your production prompt — it's the instrument
  measuring everything else.

**16. Scored vs. pairwise — and the biases that will fool you.**
- **Scored** (absolute, "rate this 1–3") is simple and aggregates easily, but models are inconsistent
  at absolute scales.
- **Pairwise** ("is answer A or B better?") is often *more reliable* — models compare better than they
  score in the abstract — and is ideal for "did version 2 beat version 1?" Use pairwise to *choose
  between options*, scored when you need an absolute bar to gate on.
- **The biases** (real, silently skew results): **position bias** (favoring first/last — run *both*
  orderings and average), **verbosity bias** (favoring longer answers regardless of quality — watch
  length confounds), **self-preference** (a model rating its own family higher — judge with a
  *different* model family for high-stakes calls), and **authority bias** (favoring answers that *sound*
  confident/citational — formal tone, hedged-then-decisive, dressed with references — over answers that
  are actually correct; an answer that fabricates a citation can out-score a hedged-but-right one).
- *Build consequence:* Design around the biases or your eval lies to you confidently. At minimum: swap
  positions in pairwise, control for length, don't reward confident phrasing over correctness, and never
  let one model be both sole author and sole judge of a high-stakes decision.

**17. Validate the judge against humans — an unvalidated judge is just another opinion.**
Before trusting a judge to gate deploys, **confirm it agrees with human judgment**: have humans grade
~30–50 cases, run the judge on the same cases, measure agreement (even simple % agreement to start).
Disagreement almost always means an ambiguous rubric — fix it and re-check. Only a judge that tracks
human ratings is a *measurement*; otherwise it's an unaudited model guessing in bulk.
- *Build consequence:* The judge is infrastructure you *calibrate once and recheck periodically*
  (especially after a provider model update). A validated judge is among the highest-leverage assets
  you can build — it lets you evaluate everything else cheaply, forever.

---

## Interview drill (self-test)
Real Gen-AI interview questions on today's material — *answer from memory, then check against what you just read.*

1. How to benchmark Sonara?
2. What metrics did you use?
3. What are evaluations in AI/ML?
4. How do you evaluate LLM output?
5. What are LLM evolution metrics?
6. What metrics are used for measuring accuracy?
7. What is context precision and context recall?
8. Why is the formula for F1 score so complicated?
9. For your use case, which metric was most relevant?
10. What benchmarks should be used for evaluating LLMs?
11. What are the frameworks for evaluation? Keep it short.
12. Give some examples where precision is used over recall.
13. In a confusion matrix, which value should be preferred?
14. We need to automate this; what metrics will we be using?
15. What performance metrics are used for Hugging Face models?

_(48 matched; 15 shown — more in [`ai-eng-questions.md`](../ai-eng-questions.md).)_

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
