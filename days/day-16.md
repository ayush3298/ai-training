# Day 16 — Evaluation, part 2 — regression, guardrails & reliability

> [← Day 15](day-15.md) · [All days](README.md) · [Day 17 →](day-17.md)

**Module:** Evaluation & Safety  ·  **Time:** ~2.5–3 hrs

## Where we are

_Continues **Evaluation & Safety**. Earlier days covered Parts A, B, C; today picks up where they left off._

---

## Part D — Regression testing & the iteration loop

**18. Eval-driven development: change one thing → re-run the eval → compare the number.**
The core working loop and the payoff of Parts A–C. New prompt, different model, smaller chunks, a
reranker, a fine-tune? **Run it against the eval set and compare scores.** "0.88 vs 0.79 faithfulness"
is a decision you can defend; "it feels better" is not. Exactly the discipline [Chapter 4](04-rag.md) (chunk sizes),
[Chapter 5](05-agents.md) (trajectories), and [Chapter 6](06-customization.md) (base-vs-tuned) kept deferring to here.
- *Build consequence:* Every change becomes an experiment with a measured outcome. You stop arguing
  about whether a change helped and start *knowing* — and you catch the changes that quietly make
  things *worse*, the more common and dangerous case.

**19. Regressions: the failure you fixed coming back silently.**
LLM systems are entangled — improving behavior for case A can break case B, invisibly. So **every bug
you fix becomes a permanent eval case.** Re-running the full set on each change surfaces a regression
as a *dropped number*, not a user complaint three weeks later.
- *Build consequence:* Treat the eval set like a regression suite that only grows. Add the failing case
  the moment you find a bug — *before* you fix it, so you can confirm the fix — and never delete cases.
  The set becomes the institutional memory of every way your system has been wrong.

**20. Version prompts and datasets together; run eval in CI.**
Your prompt is code — version it ([Chapter 3](03-prompt-engineering.md)'s templating/versioning). Your eval set is the test suite —
version it alongside. Ideally **eval runs automatically in CI** on changes, gating merges below a
quality bar, like unit tests.
- *Build consequence:* "Which prompt version produced this result, and what did it score?" must always
  be answerable. Untracked prompt edits + no eval gate = silent quality drift nobody can explain later.

**21. The metric-chasing trap (Goodhart's law).**
*"When a measure becomes a target, it ceases to be a good measure."* You can push your number up while
real quality stalls or drops — tuning answers to please a verbosity-biased judge, or overfitting to a
stale eval set that no longer reflects real traffic. The metric is a *proxy* for quality, not quality
itself.
- *Build consequence:* Periodically sanity-check the proxy against reality: read actual outputs, refresh
  the eval set from live traffic, watch for "the number rose but users aren't happier." Trust the eval
  set *and* keep your eyes on real outputs — the moment you stop looking, the metric drifts from truth.

---

## Part E — Safety & guardrails (input and output)

**22. Guardrails wrap the model on both sides — input *and* output.**
The model is one component in a system *you* are responsible for. A guardrail is a check you run
**before** sending input to the model and **after** receiving output, enforcing safety/policy the
model won't reliably self-enforce. Picture a sandwich: input guardrails → model → output guardrails.
- *Build consequence:* Never let raw user input flow straight into the model, or raw model output
  straight to a user/downstream system. The guardrail layers are where *you* enforce the rules — the
  model is helpful, fallible, and manipulable, not a security boundary.

**23. Input-side guardrails — check before the model sees it.**
- **Prompt-injection defense** (recap [Chapter 3](03-prompt-engineering.md)): untrusted text (user input, retrieved docs, tool
  output) can carry instructions that hijack the model. Delimit and label untrusted content as *data,
  not instructions*; don't let it override the system prompt; be especially careful when it can drive a
  tool call ([Chapter 5](05-agents.md)).
- **PII / sensitive-data handling:** detect and redact/handle personal data per policy *before* sending
  or logging it (and never log secrets — your standing rule).
- **Abuse / moderation:** screen for disallowed or harmful requests with a **moderation classifier**
  (providers offer cheap ones) before spending a full model call.
- *Build consequence:* Treat *all* non-system input as untrusted by default — including text your own
  RAG retrieved and your own tools returned. The injection can come from a poisoned document, not just
  a malicious user.

**24. Output-side guardrails — check before the answer ships.**
- **Structural validation** ([Chapter 2](02-apis-and-integration.md)): does it parse / match the schema? Reject or repair if not.
- **Groundedness / hallucination check:** for RAG, run the faithfulness judge ([Part C](#part-c--llm-as-judge-grading-open-ended-quality-at-scale)) and block/flag
  answers unsupported by the context — your defense against confident fabrication.
- **Unsafe-content filtering:** moderate the *output* too (a clean input can still yield disallowed
  content) and handle it before the user sees it.
- **Refusal handling:** decide what happens when the model refuses or abstains ([Chapter 3](03-prompt-engineering.md)) — a graceful
  fallback, not a raw error or an empty box.
- *Build consequence:* The output guardrail is your last line before a user or downstream service.
  Anything you'd be unwilling to show or forward must be checked *here* — it's the only place left to
  catch it.

**25. Fail-safe, not fail-open; provider tools + your own layers.**
Providers ship moderation/safety tooling (use it — cheap and maintained), but it's a *layer*, not the
whole answer; add your own domain checks on top. And choose the failure direction deliberately: when a
guardrail or the model errors, do you **fail safe** (block/withhold — right for high-stakes: medical,
financial, irreversible actions) or **fail open** (let it through — only for low-stakes)? Default to
fail-safe unless you've consciously chosen otherwise.
- *Build consequence:* "What happens when the guardrail itself fails?" is a design question you answer
  on purpose, before it happens. Defense in depth = provider moderation + your own input/output checks
  + a deliberate fail direction — no single layer trusted alone.

---

## Part F — Reliability in production

**26. The four live metrics that actually matter — watch them continuously.**
Once live, quality alone isn't enough; track **quality** (online eval / user signals), **latency** (p50
*and* p95/p99 — tail latency is what users feel, and RAG/agents add steps), **cost** (per-request token
spend — [Chapter 2](02-apis-and-integration.md)/[Chapter 5](05-agents.md) — trending over time), and **error & refusal rates** (API failures, timeouts, and
how often the model refuses/abstains). A spike in any one is an incident signal.
- *Build consequence:* You optimized all four in development; in production you *monitor* them, because
  real traffic shifts and any prompt/model change can move them. An unwatched cost or latency curve is a
  bill or an outage you'll hear about from users first.

**27. Production traffic is the fuel for your eval set — close the loop.**
Real users send inputs you never imagined. **Log production traffic, sample it, route interesting/failed
cases into your eval set**, and use human review (thumbs, escalations, manual grading) to label them.
Over time the offline set comes to mirror reality because it's *built from* reality — and you watch for
**drift** (input distribution or model behavior shifting, e.g. a provider model update).
- *Build consequence:* Offline and online eval form a flywheel: production reveals new failures → they
  become eval cases → the set gets more representative → you ship more confidently. The eval set you
  launch with is the *worst* it will ever be.

**28. Reliability is a system property — defense in depth, not a better model.**
Pull the threads together: a reliable LLM feature is *engineered around* a fallible model. **Retries
with backoff** for transient errors ([Chapter 2](02-apis-and-integration.md) Part E), **fallbacks** (a smaller/alternate model, a cached
answer, graceful degradation when the primary fails), **timeouts and circuit breakers**, **bounds** on
agent loops ([Chapter 5](05-agents.md)), and the guardrails of [Part E](#part-e--safety--guardrails-input-and-output). No single piece makes the system reliable; the
*layering* does.
- *Build consequence:* Don't chase reliability by waiting for a better model — engineer it in with
  redundancy and graceful degradation, so that when (not if) the model misbehaves or the API hiccups,
  the feature degrades gracefully instead of breaking. Reliability is your architecture, not the model's
  promise.

---

## Interview drill (self-test)
Real Gen-AI interview questions on today's material — *answer from memory, then check against what you just read.*

1. How to do model evaluation?
2. How do you evaluate a model?
3. Explain precision and recall.
4. What metrics did you optimize?
5. What is faithfulness in metrics?
6. How do you implement a guardrail?
7. How do precision and recall work?
8. What are quality metrics for LLMs?
9. How do you evaluate an LLM application?
10. How do you configure Nvidia NeMo Guardrails?
11. Why use guardrails and how do you apply them?
12. How can you put guardrails in GitHub Copilot?
13. What Google tools are available for evaluation?
14. Do we need guardrails when using the OpenAI API?
15. How do you evaluate if a distribution is normal?

_(48 matched; 15 shown — more in [`ai-eng-questions.md`](../ai-eng-questions.md).)_

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
