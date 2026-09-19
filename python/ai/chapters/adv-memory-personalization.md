## Memory & Personalization: giving your app a memory it can govern

**Goal:** By the end you can give an LLM application a memory that survives across sessions **without retraining the model**: decide what to persist and what to deliberately drop, implement the four storage patterns (summary buffer, raw-message log, RAG-over-history, structured profile) behind one stable interface, assemble a per-turn context from them within a token budget, personalize responses from a stored profile, and ship the user-facing controls (view, export, edit, delete/forget) that turn an unaccountable memory blob into something a user *and* a regulator can trust.

**Why this matters:** Every chapter so far produced a system that forgets the user the instant the request ends; the 2026 product expectation is the opposite — the app should know the returning user's name, preferences, and the thread of last week's conversation. That expectation is entirely an engineering problem layered on top of a **stateless** model ([Chapter 1](01-foundations.md)): the model remembers nothing between calls, so *you* remember on its behalf and re-supply what's relevant. Done naively, memory becomes the most **expensive** part of the app (it inflates every single prompt), the **least reliable** (it surfaces stale or wrong facts the model then states with total confidence), and the **most dangerous** (a quietly growing pile of retained personal data). This chapter is where personalization stops being a demo trick and becomes a governed, bounded, deletable subsystem — and the natural home for the privacy reflex the curriculum keeps pointing at: a memory store *is* a personal-data store, and must be built like one.

> **Setup assumed:** You reuse the [Chapter 4](04-rag.md) `retrieve()` pipeline (here pointed at the user's *own* history rather than a doc corpus), a [Chapter 5](05-agents.md) agent or a [Chapter 2](02-apis-and-integration.md) chat loop as the thing you're giving memory to, and [Chapter 5](05-agents.md) Part D concepts 10–12 (context grows every step, short-term messages-list vs long-term RAG-over-history, compaction/summarization). This chapter assumes the reader has *met* those and **deepens** them into cross-session territory rather than re-teaching them. Local stores (Postgres / pgvector) run in Docker; secrets come from env, never hard-coded.

**Suggested split:** Two short sessions. Session 1 = Parts A–E (the storage subsystem: the two clocks, what to store, the four patterns behind one interface, budgeted per-turn assembly). Session 2 = Parts F–I (maintenance, personalization, and the governance + user-control climax).

---

## Part A — Recommendation: a standalone chapter, not a Part in Chapter 5

**1. Memory across sessions is its own subsystem — give it a standalone chapter that *deepens* Chapter 5, not another Part bolted onto it.**
The brief poses a scope question: should cross-session memory live as a new Part inside [Chapter 5](05-agents.md), or as its own chapter? The recommendation is a **standalone short extension chapter** that explicitly picks up [Chapter 5](05-agents.md) Part D and pushes it further — *not* an extra Part grafted onto Chapter 5. Three reasons it has to be standalone:

- **Scope mismatch.** [Chapter 5](05-agents.md) is about the agent *loop* — and memory there is correctly scoped to a within-a-task **scratchpad** (concepts 10–12: the context that grows every step, the short-term messages list, compaction when it gets long). Cross-session memory, structured profiles, prompt-time personalization, and view/delete controls apply *equally* to a plain [Chapter 2](02-apis-and-integration.md) chat app, a [Chapter 4](04-rag.md) RAG app, **and** an agent. None of it is agent-specific, so filing it under "agents" would mislabel a general capability as an agent feature.
- **A self-contained governance topic.** The privacy and GDPR controls — view, export, edit, and the right-to-erasure fan-out across every store — are a substantial, self-contained subject that deserves its own Parts ([Part H](#part-h--memory-is-a-personal-data-store-the-governance-reflex) and [Part I](#part-i--user-controls-view-export-edit-and-delete-the-right-to-be-forgotten)), not a few paragraphs squeezed into an agent chapter already at capacity.
- **Additive, no renumbering.** Keeping it standalone means it slots in after [Chapter 5](05-agents.md) without renumbering Chapter 5's concepts or disturbing anything that cross-references them.

The one condition under which this would instead be a Chapter 5 graft: if the course *only ever* built agents, so that "the agent's memory" were the only memory worth teaching. That's false here — the spine builds chat loops and RAG apps too — so the standalone chapter wins.

**The contract, stated up front:** this chapter **assumes and does NOT re-teach** [Chapter 5](05-agents.md) concepts 10–12. It takes the within-task scratchpad, the short-term-vs-long-term split, and compaction as already understood, and pushes all three into *cross-session* territory — memory that outlives the task, the session, and the process. When you see those ideas here, they're being *extended*, not re-explained.

One-line analogy: [Chapter 5](05-agents.md)'s memory is *"what the worker remembers while doing one task"* — it ends when the task ends. This chapter is *"what the company remembers about a customer across every visit."* Different scope, different lifetime, different home.

*Build consequence:* You file every piece of memory work in the right bucket. Within-task state lives in your agent loop (it's born and dies inside one run); cross-session state lives in a memory subsystem you design **separately**, with its own storage, budget, and delete path. Conflating the two is exactly why apps fail in one of two directions — they either forget too fast (treating durable user facts as disposable scratchpad) or hoard everything (treating disposable scratchpad as durable, indefinitely retained data).

**Hands-on ([Part A](#part-a--recommendation-a-standalone-chapter-not-a-part-in-chapter-5)):** No code. Write one sentence classifying your own app's memory needs as **session-only** (resets every visit — the scratchpad is enough) vs **cross-session** (must recall the returning user across visits — you need the subsystem this chapter builds).

## Part B — Session vs cross-session state: the two clocks

Part A settled *where* this work lives: a standalone subsystem that deepens [Chapter 5](05-agents.md)'s within-task scratchpad rather than re-teaching it. Before we build anything, we have to draw the one line everything else hangs on — the line between memory that *should* die at the end of a conversation and memory that *must* outlive it. Get this line wrong and every later Part inherits the error: you'll either throw away facts the user expects you to keep, or hoard transcripts you should have dropped.

**2. There are exactly two kinds of memory, separated by how long they're allowed to live — and the model has neither for free.**
**Session state** is everything inside one continuous conversation: the [Chapter 2](02-apis-and-integration.md) `messages` list, the [Chapter 5](05-agents.md) scratchpad, the half-finished thought from three turns ago. ("Session state" = the working memory of a single conversation — it exists only while that conversation is open.) It is *born* when the session opens and is *supposed* to die when the session closes. That death is not a bug; a conversation that ended last Tuesday has no business occupying tokens in today's prompt. **Cross-session state** — also called **persistent**, **long-term**, or just **stored** memory — is anything that must survive the end of a session: who the user is, what they prefer, a one-line summary of what you worked on last week, a fact they told you a month ago. ("Cross-session state" = facts you deliberately carry from one conversation into the next.) The crucial thing, carried straight from [Chapter 1](01-foundations.md): the model holds **neither** of these on its own. It is **stateless** — it forgets the instant a request returns. So *both* kinds of memory are notes **you** keep and re-supply. They differ in exactly one dimension: how long the note is allowed to live.

*Analogy — the barista.* Picture a coffee counter. Mid-order, you say "oat milk," and three sentences later the barista still steams oat milk for *this* drink — that's **session memory**: it holds for the length of one order, then it's gone, and that's correct. You don't want yesterday's "oat milk" forced onto today's stranger. Now picture a regular whose name and usual order are written on a card by the register; any barista on shift can pull that card and greet them by name — that's **cross-session memory**: it survives between visits because someone wrote it down on purpose. Here's the twist that makes the LLM case sharper than the café: the model is a **brand-new barista on every single request** ([Chapter 1](01-foundations.md)/[Chapter 2](02-apis-and-integration.md) statelessness). It never remembers the mid-order "oat milk" *either* — you re-hand it the whole order slip every turn. So both kinds of remembering are notes you keep and pass over the counter. The only question is whether the note gets shredded at end-of-order (session) or filed on the regular's card (cross-session).

*Build consequence:* You design two storage paths with two different lifetimes, not one memory blob. Session state lives in the request-scoped `messages` list you already manage and let it be garbage-collected when the conversation ends. Cross-session state lives in a store you write to **on purpose** and read from **on purpose** — a database row, a summary, a vector. When something "isn't remembered," the first diagnostic question is *which clock was it on?* A within-session detail that vanished after the session is working as designed; a user preference that vanished is a bug in your persistence path — never a model feature you forgot to switch on.

**3. The two clocks: the session clock resets every conversation; the user clock runs for the life of the account.**
The cleanest mental model for "how long is this note allowed to live" is **two clocks ticking at different rates**. The **session clock** starts at zero every time a conversation opens and is wiped when it closes — it measures the lifetime of one `messages` list. The **user clock** starts when the account is created and keeps running across every visit, every session, for months — it measures the lifetime of the *relationship*. A fact's home is decided by which clock it belongs on. "I'm getting a TypeError on line 12" ticks on the **session** clock: relevant now, meaningless next week, correctly discarded. "I prefer terse answers in Python" ticks on the **user** clock: true across visits, worth writing on the card. The bug both directions of failure share is putting a fact on the wrong clock — filing a transient detail on the user clock (now you hoard, see [Part C](#part-c--what-to-store-and-louder-what-not-to-store)) or treating a durable preference as session-scoped (now you forget the returning user). These are not different bugs; they're the same clock-mismatch error pointed in opposite directions.

*Analogy — extending the barista.* The session clock is the barista's short-term focus on *this* order; it resets the moment the cup is handed over. The user clock is the regular's card by the register; it only resets if someone tears the card up. Two clocks, two lifetimes, one decision per fact: *which clock does this tick on?*

*Worked trace — one user, two sessions, a week apart.* Make it concrete. User `u_42`, named Priya, an intermediate Python developer who prefers terse answers.

Session 1 (Monday), the `messages` list accumulates:
```
[user]  "Hi, I'm Priya. I work mostly in Python, intermediate level. Keep answers short."
[asst]  "Got it, Priya — terse it is."
[user]  "Why does `[]  == False` return False?"
[asst]  "Because `==` compares value/identity, not truthiness. Use `not []`."
[user]  "Got it, thanks."   ← session ends; messages list is discarded
```
For Session 2 (the following Monday) to "remember" Session 1, you must have **persisted onto the user clock** exactly the durable bytes — and *only* those:
```json
{ "user_id": "u_42",
  "name": "Priya",
  "expertise": "intermediate",
  "language_pref": "python",
  "answer_style": "terse",
  "session_summary": "2026-06-01: explained == vs truthiness for empty list",
  "written_at": "2026-06-01T10:14:00Z" }
```
The bytes you **correctly threw away** are the session-clock turns: the verbatim "Hi, I'm Priya…" message, the exact `[] == False` question, the line-by-line back-and-forth. They served their purpose inside Session 1 and have no claim on Session 2's token budget. Now Session 2 opens. The model is, again, a brand-new barista — it knows nothing. *You* read the card and prepend it:
```
[system] Returning user Priya (intermediate, Python, prefers terse answers).
         Last session: explained == vs truthiness for empty list.
[user]   "What's the cleanest way to dedupe a list?"
```
The "memory" the user experiences — being greeted by name, getting a terse Python answer without re-stating preferences — is entirely the few hundred bytes you chose to carry across the gap between two clocks. Nothing was learned by the model; something was *kept* by you. This `load_memory(user_id, current_query) -> context_block` move — read the card, return a context block to prepend — is the stable interface every later Part fills in.

**Hands-on ([Part B](#part-b--session-vs-cross-session-state-the-two-clocks)):** Take your [Chapter 5](05-agents.md) agent (or a plain [Chapter 2](02-apis-and-integration.md) chat loop) and add two identifiers to every turn: a `session_id` (new per conversation) and a `user_id` (stable across conversations). Run **two sessions for the same `user_id`** a few minutes apart — in Session 1 tell it your name and a preference; in Session 2 ask it to use them. With **no persistence layer yet**, confirm Session 2 knows *nothing* from Session 1: the `messages` list started empty, the model is stateless, and the user-clock facts were never written down. *Feel the gap before you fill it* — that empty-handed Session 2 is exactly the hole the rest of this chapter plugs. (Local stores run in Docker; any DB credentials come from env, never hard-coded.)

---

## Part C — What to store, and (louder) what NOT to store

Part B left you with a tempting fix for the empty-handed Session 2: just persist *everything* — log every message forever and you'll never forget anything. That instinct is the single most expensive mistake in this chapter, and it's wrong on three axes **at once**: cost, quality, **and** privacy. This Part derives the storage policy from those three failures, then hands you a rule you can apply per fact. The headline: memory's natural failure mode is not forgetting — it's **hoarding**.

**4. "Log everything forever" fails three ways at once — derive the rule from the failures, don't assert it.**
Start naive: the **memory write policy** is "append every message to a per-user store, replay it all into every future prompt." ("Memory write policy" = your rule for what gets written to the cross-session store and what doesn't.) Walk it into three concrete walls.

*Failure 1 — cost.* Every stored message rides in **every** future prompt, because the model is stateless and you re-supply context each call ([Chapter 2](02-apis-and-integration.md): every call resends the whole transcript; you pay per token, every time). Hand-check it: a chatty user accrues ~200 turns/month, ~50 tokens/turn = 10,000 tokens of history *per month*. Replay six months and you're prepending **~60,000 tokens to every single request** before the user has typed a word. At a representative \$3 / million input tokens, that's ~\$0.18 of pure history tax on *each* call — and unlike a one-shot prompt, it **compounds**: the bill grows every month the account stays alive, and it's paid on every turn of every session forever. You have built an app whose cost rises without bound for doing nothing new.

*Failure 2 — quality.* Suppose one Tuesday the user snaps "ugh, just give me the short version" mid-frustration, and you dutifully store `answer_style: terse` as a permanent fact. Now *every* future response — including the ones where they'd have wanted detail — is clipped, because a one-off mood got promoted to a durable preference. Worse is the subtle case: a **low-confidence inferred fact**. You guess from one message that the user is a beginner and store it as fact; weeks later they're shipping production code and the model is still explaining what a variable is. Stale or wrongly-inferred memory doesn't sit quietly — it **poisons every future turn**, and it does so with the model's full confidence. This ties straight to [Chapter 1](01-foundations.md)'s hallucination point: a wrong *stored* fact is worse than no memory, because the model will state it with total confidence **and** you've made the hallucination both **persistent** (it resurfaces every session) and **personalized** (it's specifically about this user). You didn't avoid a hallucination; you saved one to disk and aimed it.

*Failure 3 — privacy.* Every message you keep is personal data you now have to **secure, justify, and be able to delete**. A six-month transcript is a pile of someone's life — possibly secrets, payment details, health mentions — that you must encrypt, defend against breach, produce on a data-access request, and erase on a deletion request. Storing it "just in case" means you've taken on every one of those obligations for data you had no use for. (This is the governance reflex the chapter is built around — picked up in full in [Part H](#part-h--memory-is-a-personal-data-store-the-governance-reflex) and [Part I](#part-i--user-controls-view-export-edit-and-delete-the-right-to-be-forgotten).)

*Build consequence:* The three failures point at one rule: **store the least that makes the app feel like it remembers.** Memory is a liability you take on deliberately for a specific payoff, not a default you accumulate. Every write needs a justification — "this is durable, reusable, and specific to this user" — and anything that can't earn all three is dropped, not kept "just in case."

**5. Store durable/reusable/user-specific facts; refuse to store the rest — and split "remembering a conversation" from "remembering a fact."**
Turn the rule into a checklist. **Store** something only if it is all three: **durable** (true beyond this session — survives on the user clock), **reusable** (it'll improve a *future* turn, not just this one), and **user-specific** (about this user, not general knowledge the model already has). That admits a short, clean list: stable preferences (terse answers, Python, dark mode), identity/profile fields (name, role, expertise level), decisions-and-their-outcomes ("chose Postgres over Mongo for the analytics service"), and **distilled summaries** of past conversations. Against it, a hard **NOT-store** list:

- **Secrets and credentials** — API keys, passwords, tokens. Never. (If one appears, redact, don't persist.)
- **Full payment / health data you don't need** — card numbers, diagnoses. Keep the minimum the feature actually requires, nothing "in case."
- **One-off transient details** — today's error message, the specific file path, the value of a variable. Session clock; let it die.
- **Anything you can't justify keeping** — if you can't name the future turn it improves, it fails "reusable." Drop it.
- **Low-confidence inferred facts** — guesses about the user. The subtle killer: a hesitant inference, once written, resurfaces as a *confident* memory the model states as fact. If you must keep an inference, store it *with* its confidence and its source turn, never as a bare assertion.

*Analogy — the junk drawer.* The anti-pattern is **memory as a junk drawer**: every receipt, twist-tie, and dead battery swept in "because you might need it," until the one thing you actually want is un-findable under the clutter — and the drawer is now a fire hazard you're liable for. A good memory store is a labeled filing cabinet: few folders, each justified, each findable, each disposable.

*The confusable that trips everyone — two things decay on different clocks.* Don't conflate **remembering a conversation** with **remembering a fact about the user**. They are different memory types on different clocks, and they **stack** rather than substitute:
- *Remembering a conversation* is **transient detail** → you **summarize, then drop the detail**. The verbatim Monday transcript becomes one line ("explained `==` vs truthiness"); the turn-by-turn text is thrown away. Store-as-**summary**.
- *Remembering a fact about the user* is **durable** → you **store it structured** and keep it. `expertise: intermediate` goes in a profile field and stays there until superseded. Store-as-**structured**.

The decision table the policy reduces to:

| Fact type | Store? | How |
|---|---|---|
| Stable preference (terse, Python) | Yes | Structured profile field |
| Identity / profile (name, role, expertise) | Yes | Structured profile field |
| Decision + outcome ("chose Postgres") | Yes | Structured, with a date |
| Past conversation (the actual turns) | As **summary** | Distill to 1–2 lines; drop the transcript |
| One-off detail (today's error, a file path) | No | Session clock — let it die |
| Secret / credential | No | Redact; never persist |
| Full payment / health data not needed | No | Keep the minimum the feature requires |
| Low-confidence inference about the user | No (or with confidence + source) | Never as a bare confident fact |

*Build consequence:* `write_memory(user_id, session)` is not a dump — it's a **classifier with a filter in front of it**. Before anything is persisted it's tagged store / don't-store / store-as-summary, with secrets and one-offs failing closed (dropped by default). This is what keeps the store you'll build in later Parts small, cheap to replay into a budget ([Part E](#part-e--assembling-the-turn-a-memory-budget-not-a-memory-dump)), honest enough not to poison future turns, and small enough to govern and delete. The work of memory is at least as much *refusing to write* as it is writing.

**Hands-on ([Part C](#part-c--what-to-store-and-louder-what-not-to-store)):** Write a `should_remember(message) -> bool | 'summarize'` classifier. **Rules first:** drop anything matching a secret/credential pattern (API-key shapes, "password", long hex/base64 blobs), drop obvious one-offs (stack traces, file paths, "today/now" references), keep messages that state a stable preference or identity fact, and return `'summarize'` for narrative conversation turns. **Then optionally** add a cheap model call for the ambiguous middle (one classification prompt, low cost). Run it over a **real chat transcript** and print three columns — *persist as fact / summarize / drop* — with the reason for each. Sanity-check the output: **no secrets and no one-off details made the "persist" cut.** If one did, your filter failed closed where it should have failed safe — fix the rule, not the example.

## Part D — The four memory patterns, behind one stable interface

[Part C](#part-c--what-to-store-and-louder-what-not-to-store) decided *what* earns a place in the store. This Part decides *how* it's stored and recalled — and it does so the way the curriculum has handled every storage subsystem: behind **one stable interface** (a single function signature every caller uses, so the implementation behind it can change without rewriting the callers — the same device as [Chapter 4](04-rag.md)'s `retrieve(question)`). Two functions cover the whole subsystem, and the rest of this chapter ([Part E](#part-e--assembling-the-turn-a-memory-budget-not-a-memory-dump) onward) only ever calls these two:

```python
def load_memory(user_id, current_query) -> str:   # returns a context_block to splice into the prompt
    ...
def write_memory(user_id, session) -> None:        # persists what this session is worth keeping
    ...
```

Everything below is an implementation *behind* those two signatures. That matters because there isn't one right way to persist cross-session state — there are four, each fixing a specific crack in the one before it, and real systems run **two to four of them at once**. If each pattern hid behind its own bespoke API, combining them would be a rewrite every time. Behind `load_memory`/`write_memory`, combining them is a change to one function body.

**The running example.** Carry one user through all four: a developer, **Maya** (`user_id="maya"`), who uses your coding assistant. Across sessions she has told it: *"my name is Maya," "I'm a senior backend engineer — skip the basics," "answer in Spanish,"* and a long thread last month debugging a gnarly Postgres deadlock. Watch the **same four facts** get stored and recalled four different ways. The point of the example is that no single pattern handles all four facts well — which is exactly why you combine them.

> **Setup assumed:** Postgres + pgvector in Docker (the [Chapter 4](04-rag.md) store), `embed()` and `normalize()` from [Chapter 4](04-rag.md), and the [Chapter 4](04-rag.md) `retrieve()` pipeline — which you **reuse, not rebuild**. Secrets (DB URL, API keys) come from env. The model-call helpers `anthropic_msg()` / `openai_msg()` are the thin wrappers from [Chapter 2](02-apis-and-integration.md).

---

**6. Summary buffer: keep a running natural-language paragraph of the relationship, rewritten each session.**
The cheapest cross-session memory is a single block of prose — the **summary buffer** (a short, running natural-language summary of everything that matters about this user, stored as one text field and updated at the end of each session). You don't store the conversation; you store *the gist of the relationship so far*, and at the end of every session you ask the model to fold the new session into the old summary.

*Analogy: the "previously on…" recap.* A TV episode doesn't replay all prior seasons — it plays a 20-second montage of just the beats you need to follow tonight. The summary buffer is that montage for the user: "Maya is a senior backend engineer; prefers Spanish; we've been debugging a Postgres deadlock." Compact, human-readable, and enough to feel continuous.

This is [Chapter 5](05-agents.md)'s compaction (concept 12) pushed across the session boundary: there, you summarized old *steps within one task*; here, you summarize old *sessions across the relationship*. Same operation, longer clock ([Part B](#part-b--session-vs-cross-session-state-the-two-clocks)).

The update is one model call, and it's worth seeing **side by side** because it's the same call on either provider — only the SDK surface differs:

```python
# --- summary update: fold this session into the running summary (Anthropic) ---
SUMMARY_PROMPT = (
    "Update the running summary of this user with anything durable from the new session. "
    "Keep it under 120 words. Prefer stable facts (role, preferences, ongoing problems) "
    "over one-off chatter. Return only the updated summary.\n\n"
    "CURRENT SUMMARY:\n{old}\n\nNEW SESSION:\n{transcript}"
)

def update_summary_anthropic(old_summary, transcript):
    msg = anthropic_msg(  # thin Ch2 wrapper around client.messages.create(model="claude-...")
        prompt=SUMMARY_PROMPT.format(old=old_summary or "(none yet)", transcript=transcript),
        max_tokens=300,
    )
    return msg.strip()
```
```python
# --- the identical operation (OpenAI) — same prompt, different SDK surface ---
def update_summary_openai(old_summary, transcript):
    msg = openai_msg(  # thin Ch2 wrapper around client.chat.completions.create(model="gpt-...")
        prompt=SUMMARY_PROMPT.format(old=old_summary or "(none yet)", transcript=transcript),
        max_tokens=300,
    )
    return msg.strip()
```

The summary buffer is **lossy by design** — that's the feature, not a bug. It will say "we debugged a Postgres deadlock" but won't preserve the exact stack trace or the precise `pg_locks` query that fixed it. Great for *gist*; useless when you need the *exact* fact verbatim.

*Build consequence:* Reach for a summary buffer when the recall guarantee you need is **"the gist, cheaply"** — a warm sense of continuity in one small, always-affordable block. One row per user (`user_id`, `summary`, `updated_at`); cost is one model call per session, not per turn. Its crack: you cannot retrieve a specific past detail from prose you've already compressed away — which forces the next pattern.

---

**7. Raw message log: store every turn verbatim in a table keyed by user — complete, auditable, but not yet context.**
The fix for the summary buffer's lossiness is to throw nothing away: a **raw message log** (a database table holding the full, verbatim turns of every conversation, keyed by `user_id` and `session_id`). Now the exact stack trace, the exact `pg_locks` query, every word Maya ever typed — all of it is preserved and **auditable** (you can prove what was actually said, which the governance Parts will need).

*Analogy: the security-camera tape.* A shop records every hour of every day. The footage is complete and authoritative — but nobody watches all of it. You **have** it; you don't **replay** it. The raw log is the same: total recall in storage, but you cannot pour Maya's entire history into a prompt. Months of transcripts would blow past the context window and cost a fortune *per turn* — the exact "memory becomes the most expensive part of the app" failure [Part C](#part-c--what-to-store-and-louder-what-not-to-store) warned about, except now self-inflicted every single request.

So the raw log is **storage, not context** — until you *select* from it. That is the crucial distinction: a complete log you can't afford to replay is an archive, not a memory you can use. The table itself is unremarkable — Postgres in Docker, one row per turn:

```sql
-- raw message log: the verbatim archive. Note user_id on every row — this becomes load-bearing in concept 8.
CREATE TABLE messages (
    id          bigserial PRIMARY KEY,
    user_id     text        NOT NULL,          -- whose turn (the isolation key, tagged at write time)
    session_id  text        NOT NULL,          -- which conversation
    role        text        NOT NULL,          -- 'user' or 'assistant'
    content     text        NOT NULL,          -- the verbatim turn
    written_at  timestamptz NOT NULL DEFAULT now(),
    embedding   vector(1536)                   -- filled in for concept 8; NULL is fine until then
);
CREATE INDEX ON messages (user_id, session_id);   -- fast "this user's history"
```
```python
def log_turn(user_id, session_id, role, content):
    conn.execute(
        "INSERT INTO messages (user_id, session_id, role, content) VALUES (%s, %s, %s, %s)",
        (user_id, session_id, role, content),
    )
```

*Build consequence:* Reach for a raw log when the recall guarantee you need is **"audit / completeness"** — the durable, provable record of what was actually said (and the substrate the right-to-erasure work in [Part I](#part-i--user-controls-view-export-edit-and-delete-the-right-to-be-forgotten) deletes from). Tag `user_id` at write time; you cannot reconstruct it later ([Chapter 4](04-rag.md) concept 8's metadata rule). Its crack: you can store everything but you can't *afford to read* everything — which forces a way to pull back only the relevant turns.

---

**8. RAG-over-history: embed the past and retrieve only the relevant turns — Chapter 4's pipeline, pointed at the user's own history.**
The raw log gives you everything; you need a way to fetch only the slice that matters *this turn*. That is **RAG-over-history** (Retrieval-Augmented Generation run over the user's own past messages and summaries instead of over a document corpus): embed each past turn, and at request time retrieve only the handful semantically relevant to the current query. [Chapter 5](05-agents.md) concept 11 named this in one line — *"long-term memory is just retrieval pointed at the agent's own history."* Here you build it, and the build is almost nothing new, because **it is exactly [Chapter 4](04-rag.md)'s `retrieve()` pipeline** with the corpus swapped for the user's history.

*Analogy: searching the tape for the relevant minute.* Concept 7 had the full security footage but nobody watches it all. RAG-over-history is the timestamp search: Maya asks a follow-up about her deadlock, and instead of replaying months of tape you jump straight to the three minutes where that deadlock was discussed. Relevant, bounded, affordable.

Here is the whole point of the stable interface paying off — you do **not** write a new retriever. You call [Chapter 4](04-rag.md)'s, pointed at the `messages` table, scoped to one user:

```python
# RAG-over-history = Ch4 retrieve(), corpus swapped for the user's own turns, scoped by user_id.
# This is the Ch4 retrieve(question, tenant_id) signature with tenant_id -> user_id (see note below).
def retrieve_history(user_id, query, k=4):
    if not user_id:                      # fail-closed (Ch4 concept 51): no scope -> refuse, never run unscoped
        raise PermissionError("retrieve_history() requires a user_id; refusing an unscoped memory search")
    qv = normalize(embed(query))
    return conn.execute(
        """
        SELECT content, embedding <=> %s AS distance
        FROM messages
        WHERE user_id = %s               -- pre-filter: candidate set is THIS user's memories only
        ORDER BY embedding <=> %s
        LIMIT %s
        """,
        (qv, user_id, qv, k),
    ).fetchall()
```

The embed step is the same `embed()`/`normalize()` from [Chapter 4](04-rag.md), and again it's the **identical operation on either provider** — only the embeddings endpoint differs:

```python
# embedding a past turn so it becomes retrievable — Anthropic-side helper
vec = embed_anthropic(turn_text)   # -> normalize() -> store in messages.embedding
# the same operation, OpenAI-side — same vector role, different SDK surface
vec = embed_openai(turn_text)      # -> normalize() -> store in messages.embedding
```

> **Cross-reference note.** The RAG-over-history leg **is** the [Chapter 4](04-rag.md) pipeline with a `user_id` metadata **pre-filter** — the exact mechanism from [Chapter 4](04-rag.md) Part L (multi-tenant isolation, concepts 50–51). There, `tenant_id` ANDed into the `WHERE` clause kept one tenant from retrieving another's documents; here, `user_id` ANDed into the `WHERE` clause keeps **one user from retrieving another's memories**. It is the same hard pre-filter doing the same job: a memory store is multi-tenant by nature (every user is a tenant), so isolation is not optional. Tag `user_id` at write time, derive it from the **authenticated session, never the request body** (an `IDOR` otherwise — letting a caller read another's memory just by naming their `user_id`), and **fail-closed** when it's missing. Prove it with the same isolation test [Chapter 4](04-rag.md) Part L mandated — see this Part's Hands-on.

*Build consequence:* Reach for RAG-over-history when the recall guarantee you need is **"the relevant slice of the fuzzy past"** — anything where you can't predict in advance which old turn matters, so you find it by similarity at request time. It reuses the [Chapter 4](04-rag.md) retriever wholesale; your only new work is embedding turns on write and ANDing `user_id` into the filter. Its crack: similarity search is *fuzzy* — it surfaces "probably relevant" turns, never a guaranteed exact value. When you need Maya's name spelled correctly **every** time, you cannot leave it to a top-k similarity roll.

---

**9. Structured profile: extract durable facts into typed fields you read deterministically — no retrieval, no guessing.**
The fix for retrieval's fuzziness is to stop searching for the facts that never change and just **store them as fields**. A **structured profile** is a small typed record (name, language preference, expertise level, key dates) that you read by key — `profile["language"]` — with no embedding, no similarity, no model call. It's **deterministic** (the same read returns the same value every time, no probabilistic ranking in the path).

*Analogy: the form fields at the top of a customer record.* A support rep doesn't run a semantic search to find your name and account tier — they're printed in labeled boxes at the top of the screen. Maya's name, her Spanish preference, her senior-level expertise: these are form fields, not tape you search. You read them; you don't retrieve them.

"From scratch" here is genuinely just a dict plus a typed schema — **no magic, no library required**. The schema (a declared shape with types, so a bad write fails loudly instead of silently corrupting the profile) is the only discipline:

```python
# from scratch: a JSON blob + a typed schema. That's the entire pattern.
from pydantic import BaseModel   # the schema = a typed contract; a bad write raises, it doesn't corrupt

class Profile(BaseModel):
    name: str | None = None
    language: str = "en"               # default until the user states otherwise
    expertise: str | None = None       # e.g. "senior backend engineer"
    key_dates: dict[str, str] = {}     # {"renewal": "2026-09-01"}

# stored as one JSON row per user; read by key, deterministically — no retrieval in the path.
def load_profile(user_id) -> Profile:
    row = conn.execute("SELECT data FROM profiles WHERE user_id = %s", (user_id,)).fetchone()
    return Profile(**row["data"]) if row else Profile()

def save_profile(user_id, profile: Profile):
    conn.execute(
        "INSERT INTO profiles (user_id, data) VALUES (%s, %s) "
        "ON CONFLICT (user_id) DO UPDATE SET data = EXCLUDED.data",   # supersede-by-key (see Part F)
        (user_id, profile.model_dump_json()),
    )
```

How does a fact *become* a profile field? You extract it — a model call that reads the session and emits the typed shape (the same structured-output technique from [Chapter 3](03-structured-output.md)). But once extracted, reading it is pure key access. Maya's name lands in `profile.name` once and is read **verbatim, every turn, forever** — no roll of the retrieval dice.

*Build consequence:* Reach for a structured profile when the recall guarantee you need is **"this exact field, deterministically"** — the durable facts you must get right every single time and can name in advance (identity, preferences, entitlements, dates). It's the cheapest to *read* (one keyed lookup, no model call) and the easiest to *show and edit* in the user-facing controls of [Part I](#part-i--user-controls-view-export-edit-and-delete-the-right-to-be-forgotten) — a profile field maps straight onto a settings form. Its limit is the mirror of its strength: it only holds facts you predicted and typed; the open-ended past still belongs to RAG-over-history.

---

### The punchline: you combine them, you don't choose one

Lay the four against Maya's facts and the lesson is immediate — **no single pattern handles all four well**:

| Maya's fact | Best-fit pattern | Recall guarantee |
|---|---|---|
| "my name is Maya" | **structured profile** | exact field, every time |
| "answer in Spanish" | **structured profile** | exact field, every time |
| "senior engineer — skip basics" | **profile** (fact) + **summary** (tone) | exact + gist |
| last month's deadlock thread | **RAG-over-history** over the **raw log** | the relevant fuzzy slice |
| "prove what was said on May 3" | **raw log** | audit / completeness |

A real coding assistant for Maya runs a **structured profile** (name, language, expertise — read deterministically every turn), a **summary buffer** (the running gist of the relationship), a **raw log** (the auditable archive + the substrate for deletion), and **RAG-over-history** over that log (the relevant slice of past detail). That's all four — and they coexist precisely because they hide behind `load_memory`/`write_memory`. `write_memory(user_id, session)` updates the summary, appends + embeds the raw turns, and re-extracts the profile; `load_memory(user_id, query)` reads the profile by key, takes the summary, and retrieves the relevant history — then hands the assembled result to [Part E](#part-e--assembling-the-turn-a-memory-budget-not-a-memory-dump), which fits it inside a token budget.

> **Anti-pattern, named.** "Pick the one memory pattern." There is no single winner; picking one means a known failure — profile-only forgets the conversation, RAG-only fuzzes the name, summary-only loses the audit trail, raw-log-only bankrupts you per turn. The senior move is to **pick patterns by the recall guarantee each fact needs** — exact field → profile; fuzzy relevant past → RAG; gist → summary; audit → raw log — and let them compose behind one interface.

A closing scope reminder for all four: every pattern here **persists and retrieves**; **none of them trains the model**. You are keeping notes and re-supplying them, exactly as [Part A](#part-a--recommendation-a-standalone-chapter-not-a-part-in-chapter-5) framed it — the model stays stateless ([Chapter 1](01-foundations.md)), and the entire subsystem is data you can read, edit, and delete. Personalization from these stores is a **prompt-time** act ([Part G](#part-g--personalization-turning-stored-memory-into-a-tailored-response)), never a weight change.

**Hands-on ([Part D](#part-d--the-four-memory-patterns-behind-one-stable-interface)):** Implement all four patterns behind `load_memory(user_id, current_query)` and `write_memory(user_id, session)`, against Postgres + pgvector in Docker (reuse the [Chapter 4](04-rag.md) store; DB URL and keys from env):

1. **Summary buffer** — a `summaries(user_id, summary, updated_at)` row + an `update_summary()` call that folds a session into the running prose (concept 6). Have `write_memory` call it.
2. **Raw message log** — the `messages` table (concept 7); `write_memory` appends every turn with `user_id` tagged at write time.
3. **RAG-over-history** — embed each logged turn, then implement `retrieve_history(user_id, query)` by **calling [Chapter 4](04-rag.md)'s `retrieve()`** pointed at `messages` with a `user_id` pre-filter (concept 8) — do not write a new retriever.
4. **Structured profile** — the typed `Profile` schema + `load_profile()`/`save_profile()` JSON store (concept 9); `load_memory` reads it by key.

Then **assemble** them: `load_memory` returns a `context_block` combining profile fields, the summary, and the retrieved history; `write_memory` updates all three stores.

**Prove the `user_id` filter isolates users** (echoing [Chapter 4](04-rag.md) Part L's multi-tenant test): seed memories for two users where **user B owns the turn most semantically similar to user A's query**. Assert `retrieve_history("A", query)` **never** returns a B turn and **still** returns A's best in-scope turn. Add a negative test: a missing/empty `user_id` **fails-closed** (raises) rather than searching everyone's memory. That passing test is the Definition of Done for this Part — isolation you haven't tested is isolation you don't have.

## Part E — Assembling the turn: a memory budget, not a memory dump

[Part D](#part-d--the-four-memory-patterns-behind-one-stable-interface) gave you four stores behind one interface — `load_memory(user_id, current_query)` and `write_memory(user_id, session)`. But having four stores is useless until you decide *what goes into THIS prompt*. The model is stateless ([Chapter 1](01-foundations.md)); every turn you hand it a fresh context assembled from scratch. Parts E adds the function that does the assembling — `assemble_context(user_id, query, token_budget)` — and the discipline that keeps it from blowing up: a **token budget** (a fixed ceiling, in tokens, on how much memory you're allowed to paste into the prompt this turn).

**10. The per-turn context is *curated* under a budget, not dumped — it's a fixed-size working set chosen from an unbounded history.**
You have, in principle, everything: a structured profile, a running summary, every raw turn the user ever sent, and a vector index over all of it. You cannot send all of it — a two-year user has megabytes of history and the window is finite and billed per token. So each turn you *select*. This is the cross-session generalization of two things you already know: [Chapter 5](05-agents.md) concept 10 (context grows every step — your main enemy) and [Chapter 2](02-apis-and-integration.md)'s window management (trim or summarize older turns). There, the unbounded thing was *one task's transcript*; here it's *one user's entire relationship with the app*. Same move, longer clock: curate a fixed-size working set from an unbounded source.

The pieces are not equal, so don't treat them equally. Rank them:

- **System prompt** — fixed, always present (instructions, tools, format rules).
- **Structured profile** — *always* included. It's tiny and exact (name, expertise level, language preference — a handful of fields), and it's the cheapest, highest-value memory you own. Never make the profile compete for space.
- **Recent raw turns** — *always* included. The last few messages are the live thread; drop them and the model loses the plot of the current conversation.
- **Running summary** — included when it exists. One compact paragraph standing in for everything older than the recent turns.
- **Retrieved relevant memories** — RAG-over-history ([Part D](#part-d--the-four-memory-patterns-behind-one-stable-interface)), pointed at the user's own past. These are *not* always included; they fill whatever budget remains, ranked by relevance to the current query.

*Analogy:* packing a **carry-on with a hard size limit**. The bag is your token budget — it does not stretch. Some things go in *no matter what*: your passport (the profile — small, identifies you, useless trip without it) and today's clothes (the recent turns — what you need right now). Then you fill the *remaining* space by what *this specific trip* needs: ski gloves for a ski trip, swimwear for a beach trip (the retrieved memories relevant to *this* query). You do not empty your entire wardrobe into a bag that can't close.

*Build consequence:* You write `assemble_context` as a *prioritized packer*, not a concatenator. Profile and recent turns are reserved allocations taken off the top; the summary takes its slot if present; retrieved memories are sorted by relevance and added until the budget is spent, then *cut off*. The cut-off is the point — it means context size stays flat as history grows, so cost and latency stay flat too. If your per-turn token cost climbs the longer a user has been with you, you built a dumper, not a packer.

**11. More memory in context makes answers *worse* past a point — your own store can trigger "lost in the middle."**
The beginner instinct is "memory is good, so more memory is better — stuff everything in to be safe." [Chapter 4](04-rag.md) already taught the counter-lesson for documents: a model's attention degrades across a long context, and facts buried in the middle of a big blob get *ignored* even when they're present ("lost in the middle" — the model attends well to the start and end of its context and poorly to the middle). That trap doesn't disappear because the text came from your own memory store instead of a document corpus. Fifty retrieved memories don't make the model fifty-times-more-personal; they bury the three that mattered and dilute the recent turns that carry the actual conversation.

Make it concrete with a **4 000-token memory budget** (the ceiling on memory content this turn, *separate* from the system prompt and the model's reply allowance). A sane allocation:

| Slot | Tokens | Notes |
|------|-------:|-------|
| Structured profile | 200 | always — tiny, exact, identity + 3 prefs |
| Recent raw turns (last 6) | 1 500 | always — the live thread, ~250 each |
| Running summary | 300 | one paragraph standing in for older history |
| Retrieved memories | ~2 000 | fills the remainder, ~250 tokens each |
| **Total** | **4 000** | hard ceiling |

The remainder for retrieved memories is `4000 − 200 − 1500 − 300 = 2000` tokens. At ~250 tokens per retrieved memory that's `2000 / 250 = 8` memories. So you include the **top 8** by relevance score — and *only* 8.

Now the user has *two years* of history — say 1 200 stored memories that the retriever ranks against this query. The budget did not change: still 8 slots. Memories ranked 9th through 1 200th are **dropped this turn** — not deleted, just not in *this* prompt. That's correct behavior, not a bug. If instead you "played it safe" and pasted the profile plus all summaries plus 50 retrieved memories, you'd spend perhaps 13 000+ tokens, push the recent turns toward the lost-in-the-middle dead zone, pay 3× the bill, and get a *worse* answer because the three memories that mattered are now needles in a haystack you built yourself.

*Anti-pattern:* *"dump the whole profile + every summary + the top 50 retrieved memories, just in case."* This is the memory version of the [Chapter 4](04-rag.md) huge-context trap: it maximizes cost and minimizes signal-to-noise. The fix is the budget itself — a hard ceiling forces you to *rank and cut*, which is exactly what keeps the answer sharp.

> Profile and recent turns are non-negotiable reservations; the summary is conditional; retrieved memories are the *only* part that flexes with the budget. When you must trim, you trim retrieved memories from the bottom of the relevance ranking — never the profile, never the recent thread.

**Hands-on ([Part E](#part-e--assembling-the-turn-a-memory-budget-not-a-memory-dump)):** Write `assemble_context(user_id, query, token_budget)` that pulls from all four stores ([Part D](#part-d--the-four-memory-patterns-behind-one-stable-interface)) and packs to the budget by priority: reserve fixed allocations for the structured profile and the last N raw turns, slot the summary if present, then add retrieved memories in descending relevance order until the remaining budget is spent, and cut the rest. Use a token counter (the provider's tokenizer, or a ~4-chars-per-token estimate) so the reserved-vs-remaining arithmetic is real, not guessed.

```python
def assemble_context(user_id, query, token_budget=4000):
    profile = load_profile(user_id)            # structured store — always
    recent  = load_recent_turns(user_id, n=6)  # raw log tail — always
    summary = load_summary(user_id)            # summary buffer — if present

    blocks, used = [], 0
    for label, text in [("profile", render_profile(profile)),
                        ("recent",  render_turns(recent)),
                        ("summary", summary or "")]:
        if not text:
            continue
        blocks.append((label, text))
        used += count_tokens(text)

    remaining = token_budget - used
    hits = retrieve(query, user_id=user_id, k=50)   # ch04 retrieve(), user_id pre-filter
    for h in hits:                                   # already sorted by relevance
        cost = count_tokens(h["text"])
        if cost > remaining:
            break                                    # budget spent — cut the rest
        blocks.append(("memory", h["text"]))
        remaining -= cost

    context = "\n\n".join(f"<{lbl}>\n{txt}\n</{lbl}>" for lbl, txt in blocks)
    log.info("assemble", user=user_id, tokens=token_budget - remaining,
             retrieved=sum(1 for l, _ in blocks if l == "memory"))
    return context
```

Then log the token cost per turn for a user whose history you grow from 10 to 1 000 stored memories, and confirm the assembled-context token count **stays flat** (pinned near the budget) instead of climbing with history. That flat line is the whole point of the budget.

---

## Part F — Memory maintenance: updating, decaying, resolving conflicts

[Part D](#part-d--the-four-memory-patterns-behind-one-stable-interface)'s four stores quietly assumed something false: that a stored fact is *stable* — write it once, read it forever. Users aren't stable. They change their mind, their preferences age out, and what they tell you *today* can flatly contradict what's in the store from *January*. Memory is not write-once; it **rots**. This Part is the maintenance layer that keeps a memory store honest as the user it describes keeps changing.

**12. A wrong-but-confident stored fact is a *persistent* hallucination — un-timestamped memory patronizes the user.**
[Part C] warned that a wrong stored fact is worse than a forgotten one, because the model will state it with total confidence every single turn. The sharper version: facts don't have to be wrong *when written* to become wrong — they **go stale**. Walk the running example forward.

In **January**, the user says "I'm a beginner, explain things simply." You store it. Good — at the time, true and useful. In **June**, the same user says "I ship to prod daily, just give me the API signature." Now your store holds *two* facts about their expertise, and they contradict. An un-timestamped, un-decayed store has no way to know which one wins. The retriever, ranking purely by relevance to a query like "how do I cache responses?", happily surfaces the January "I'm a beginner" memory — it's a perfectly relevant memory about this user's skill level — and the model dutifully delivers a hand-holding explainer to someone who ships daily. The user feels **patronized** by their own app, and the cause is purely structural: you stored facts as if they were timeless.

*Build consequence:* Every memory must carry a `written_at` timestamp from the moment you adopt this design — retrofitting it later is painful because old rows have no honest date. Without it you cannot tell a current fact from a stale one, and "the app thinks I'm still a beginner" becomes an un-debuggable complaint. The timestamp is the cheapest insurance in the whole subsystem; pay it up front.

**13. Memory is a wiki with edit history, not a pile of sticky notes — updates supersede, recency decays, last-write-wins.**
*Analogy:* a pile of **sticky notes** is append-only — you slap a new note on the wall and the old contradictory one is still right there, equally loud, with no way to tell which you wrote first. A **wiki page with edit history and a "last updated" date** is the opposite: there's one current value, the old value is superseded (still in history, but not what you read), and every revision is dated. Build your memory like the wiki, not the sticky-note wall. Three concrete mechanisms, one per failure:

- **Updates → supersede-by-key.** For the *structured profile* (fixed fields like `expertise_level`), an update **overwrites the field in place** — "supersede-by-key" means the key (`expertise_level`) has exactly one current value, and writing a new one *replaces* the old rather than appending a second. "Actually I prefer Python now" sets `language = "python"`, full stop; the old `"javascript"` does not linger as a competing fact. This is why the profile is the easiest store to keep correct: one key, one value, latest wins.
- **Decay/staleness → recency weighting or TTLs.** For the *retrieval* store (RAG-over-history), you can't overwrite — there's no single key, just a pile of past statements. So you down-weight old ones. Score each candidate by **relevance × recency-decay** instead of relevance alone, where recency-decay is a multiplier that shrinks as a memory ages (e.g. `0.95 ** age_in_days`, or a hard **TTL** — time-to-live — that drops memories past, say, 180 days). A June statement now outranks a January one of equal topical relevance. (A `0.95` daily factor halves a memory's weight in about two weeks — tune the base to how fast *your* domain's facts go stale.)
- **Conflict resolution → last-write-wins, by timestamp.** When the store says X and the user just said not-X, the default is **last-write-wins**: the most recent `written_at` is authoritative. The current turn always beats the store, and a fresher memory beats an older one. Simple, predictable, and correct for the overwhelming majority of preference facts.

Replay the scenario with these in place: in June you write `expertise_level = "expert"` (supersedes January's `"beginner"` by key in the profile); the stray January "I'm a beginner" line in the retrieval store still exists but, scored by `relevance × recency`, ranks far below June's "I ship to prod daily" and falls *out of the budget* from [Part E](#part-e--assembling-the-turn-a-memory-budget-not-a-memory-dump). The latest fact wins on both paths. No more patronizing.

*Build consequence:* You pick the maintenance mechanism per store: profile → supersede-by-key (one current value); summary → regenerate/replace on update (never append contradictions); retrieval → relevance × recency-decay so stale memories sink. The `written_at` timestamp from concept 12 is what makes all three possible — it's the "last updated" date the whole wiki runs on.

*Anti-pattern:* *append-only memory that never forgets a wrong guess.* If every statement and every guess accretes forever with equal standing and no date, the store fills with contradictions and your retriever surfaces them at random. "Never forgets" sounds like a feature; in a memory store it's the bug — it means *never corrects*, either.

**Hands-on ([Part F](#part-f--memory-maintenance-updating-decaying-resolving-conflicts)):** Add a `written_at` timestamp to every memory you write (profile fields, summary, and each retrieval-store row). Implement profile updates as **supersede-by-key** (writing a field overwrites its current value, no second row), and change your retrieval ranking from relevance alone to **relevance × recency-decay**:

```python
import math, time

def recency_weight(written_at, half_life_days=14):
    age_days = (time.time() - written_at) / 86_400
    return 0.5 ** (age_days / half_life_days)   # 1.0 fresh → 0.5 at one half-life

def rank(hits, now=None):                       # hits: [{"text","score","written_at"}, ...]
    for h in hits:
        h["final"] = h["score"] * recency_weight(h["written_at"])
    return sorted(hits, key=lambda h: h["final"], reverse=True)

def update_profile(user_id, key, value):        # supersede-by-key: one current value
    profile = load_profile(user_id)
    profile[key] = {"value": value, "written_at": time.time()}
    save_profile(user_id, profile)
```

Replay the beginner→expert scenario: write the January "beginner" memory with a January timestamp, the June "expert" memory with a June timestamp, then query for help on a task. Confirm the June fact wins — the profile reads `expert`, and the January retrieval memory ranks below June's after the recency multiplier (assert the latest fact is the one that lands in the assembled context).

---

## Part G — Personalization: turning stored memory into a tailored response

Everything so far built the *input* side: stores ([Part D](#part-d--the-four-memory-patterns-behind-one-stable-interface)), budgeted assembly ([Part E](#part-e--assembling-the-turn-a-memory-budget-not-a-memory-dump)), maintenance ([Part F](#part-f--memory-maintenance-updating-decaying-resolving-conflicts)). **Personalization is the output** — actually *using* that memory to tailor the response: adapt tone, verbosity, and format to stored preferences; reference prior context ("last time we set up X…"); pre-fill known fields. Memory is the fuel; personalization is what you do with it.

**14. Personalization is *prompt-time*, not training-time — you inject the profile into the context, you do not fine-tune a model per user.**
The seductive misreading is "to make the app personal *to* a user, train a model *on* that user." It's almost always wrong, and the curriculum already armed you against it. [Chapter 4](04-rag.md)'s concept 18 and the whole [Chapter 6](06-customization.md) adaptation ladder establish the split: **RAG/prompting adds knowledge and context; fine-tuning changes default behavior** — and you reach for the *least invasive* mechanism that works, escalating only when it plateaus. Per-user adaptation is the same decision at user granularity. The user's name, expertise, and language preference are *facts about a person* — exactly the kind of specific, changeable information that belongs in the **prompt**, not baked into weights.

**Personalization as prompt-time adaptation** means: you keep one shared model, and per request you *inject* this user's profile and relevant memories into the system prompt and context. Nothing about the model changes; only the input does. Walk the decision-ladder — least invasive first:

1. **Inject preferences into the system prompt.** *(Default.)* Instant, per-user, zero training, reversible the instant you stop injecting. "User is an expert, prefers concise answers, uses TypeScript" goes into the system message and the response adapts this turn. Start here. This handles the vast majority of personalization.
2. **Retrieve relevant past context.** ([Part D](#part-d--the-four-memory-patterns-behind-one-stable-interface)'s RAG-over-history.) When the adaptation needs *specific prior facts* ("last time we set up your Postgres connection…"), pull them into context. Still prompt-time, still per-request, still deletable.
3. **Per-user fine-tuning.** Only at extreme scale, and *almost never the right first move* — see [Chapter 6](06-customization.md) for when fine-tuning earns its place at all. It means **one model per user** (operationally absurd past a handful of users), it costs a training run per person, and — the disqualifier for this chapter — **you cannot delete a single fact from weights**. "Forget I work at Acme" is a profile-row deletion in rungs 1–2; in a fine-tuned model it's un-removable short of retraining. That collides head-on with the deletion requirement in [Part I](#part-i--user-controls-view-export-edit-and-delete-the-right-to-be-forgotten). Forward-pointer noted; do not start here.

*Build consequence:* Default to rung 1 for personalization, escalate to rung 2 when you need specific recalled facts, and treat rung 3 as a last resort you'll likely never reach. This keeps personalization cheap, instant, per-user, and — critically — *deletable*, which the governance Parts ([Part I](#part-i--user-controls-view-export-edit-and-delete-the-right-to-be-forgotten)) will require of you.

**15. Same query, empty vs populated profile — the profile rides in the system message, and the model adapts.**
Make the mechanism concrete. The user asks: *"How should I cache API responses?"*

With an **empty profile**, the system prompt is generic and you get a long, beginner-pitched answer covering what caching is, in language-neutral pseudocode. With a **populated profile** — `expertise: expert`, `style: concise`, `language: TypeScript` — you inject those three facts and get a short, TypeScript-flavored answer that skips the basics and gives the signature. *Same model, same question, same code path* — only the injected profile changed.

Side-by-side, the only difference is the system message you build from the profile:

```python
# Build the system prompt from the stored profile (prompt-time personalization)
def system_prompt(profile):
    base = "You are a helpful engineering assistant."
    if not profile:
        return base
    prefs = []
    if profile.get("expertise") == "expert":
        prefs.append("The user is an expert; skip basics and be direct.")
    if profile.get("style") == "concise":
        prefs.append("Prefer concise answers.")
    if lang := profile.get("language"):
        prefs.append(f"Default to {lang} for code examples.")
    # Treat these as fallible, user-correctable facts — not gospel (see guard below).
    prefs.append("These are remembered preferences; if the user corrects one, follow the correction.")
    return base + " " + " ".join(prefs)
```

```python
# Anthropic
resp = client.messages.create(
    model="claude-sonnet-4-6", max_tokens=512,
    system=system_prompt(profile),                       # profile injected here
    messages=[{"role": "user", "content": query}],
)
```

```python
# OpenAI
resp = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": system_prompt(profile)},   # profile injected here
        {"role": "user", "content": query},
    ],
)
```

Pass `profile={}` and you get the generic answer; pass the populated profile and the same call returns the tailored one. No retraining, no second model, no deploy — just a different system string. That *is* personalization.

> **Don't confuse personalization with fine-tuning.** *Personalization* is prompt-time: per-request, reversible, and **deletable** — stop injecting a fact (or delete its profile row) and it's gone next turn. *Fine-tuning* is train-time: baked into the weights, shared across all requests, and **not individually deletable** — you can't surgically remove one user's fact from a trained model. For per-user adaptation you almost always want the former; the latter collides with the right-to-be-forgotten ([Part I](#part-i--user-controls-view-export-edit-and-delete-the-right-to-be-forgotten)).

**Name the failure modes** — personalization done wrong has three signature ways to hurt:

- **Over-personalization (creepy).** Surfacing a remembered detail the user never expected you to retain ("How's the job hunt going?" unprompted) reads as surveillance, not service. Personalize the *response*, don't perform the *memory*.
- **Filter-bubble lock-in.** If you only ever retrieve and reinforce past preferences, you trap the user in their own history — they asked for Python once and now never see anything else, even when they've moved on. Recency-decay ([Part F](#part-f--memory-maintenance-updating-decaying-resolving-conflicts)) and a willingness to *not* personalize every turn are the counterweight.
- **Acting on a wrong remembered fact.** A stale or mis-stored fact ([Part F](#part-f--memory-maintenance-updating-decaying-resolving-conflicts)) gets confidently applied — you greet a user by an old employer, or pitch a beginner answer to an expert. The defense is to treat remembered facts as *fallible and correctable mid-conversation*, never as gospel.

*Build consequence:* Inject preferences as *soft guidance the user can override*, not hard rules. Phrase the injected profile so a mid-conversation correction ("actually, I work at Globex now") wins immediately — and route that correction back through [Part F](#part-f--memory-maintenance-updating-decaying-resolving-conflicts)'s supersede-by-key so the store updates too. Personalization that can't be corrected in the moment is exactly the failure mode above; build the correction path in from the start.

**Hands-on ([Part G](#part-g--personalization-turning-stored-memory-into-a-tailored-response)):** Inject the structured profile into the system prompt (the `system_prompt(profile)` pattern above) and answer the same question — *"How should I cache API responses?"* — twice: once with `profile={}` and once with `{"expertise": "expert", "style": "concise", "language": "TypeScript"}`. Diff the two responses and confirm the populated one is shorter, skips basics, and uses TypeScript. Then add a **guard** that makes the model treat remembered facts as fallible: include a line in the system prompt licensing the model to follow an in-conversation correction over the stored profile, and verify it — send "actually I'm new to this, explain simply" *after* the expert profile is loaded, and confirm the reply switches to a beginner explanation (and that the correction is written back via supersede-by-key so the profile now reads `beginner`).

## Part H — Memory is a personal-data store: the governance reflex

You spent [Part D](#part-d--the-four-memory-patterns-behind-one-stable-interface) building four stores behind `load_memory`/`write_memory` and [Part E](#part-e--assembling-the-turn-a-memory-budget-not-a-memory-dump) assembling them into a tailored prompt, and it works: the returning user's name, expertise level, and language preference flow through every turn. Step back and look at what you actually built. It is not "a memory feature." It is **a database of personal facts about every user, kept indefinitely.** That single relabeling is this Part's entire lesson — because the moment a pile of user facts becomes durable, every data-protection obligation in the curriculum attaches to it, and the capabilities you were proud of in Parts C–G turn out to be governance duties you've half-discharged without naming them.

This Part is deliberately the API-consumer's slice: enough to know *what to build*, not a tour of statutes. When you need the full regulatory treatment — lawful bases, GDPR mechanics, the EU AI Act risk tiers, HIPAA's de-identification standard — that lives in the [Security, Privacy & Governance](adv-security-privacy-governance.md) chapter, and Part H forward-points to it rather than re-teaching it. What you owe *here* is the reflex: recognize the store for what it is, and map each thing you already built to the duty it satisfies.

**16. The instant you persist user facts across sessions, you have built a personal-data store — and every data-protection obligation attaches to it.**
A within-task scratchpad ([Chapter 5](05-agents.md) concepts 10–12) is born and dies inside one run; nobody regulates a variable that lives for three seconds. But cross-session memory is the opposite by design: it *survives* the session, the process, and the user's absence. That durability is the whole point of the chapter — and it is also exactly what makes the store a regulated artifact. "Personal data" is the legal term for any information about an identifiable person, and a memory store is, almost by definition, **the single most concentrated pile of personal data your app holds**: not one document among thousands like your [Chapter 4](04-rag.md) corpus, but a tight, indexed, per-user dossier of names, preferences, expertise, and the gist of past conversations, assembled precisely so it's easy to retrieve. The curriculum has been pointing at this reflex all along — PII redaction before a call ([Chapter 3](03-prompt-engineering.md)), multi-tenant isolation in retrieval ([Chapter 4](04-rag.md) Part L), the every-sink erasure trace ([Security, Privacy & Governance](adv-security-privacy-governance.md)) — and a memory store is where all of those threads converge into one object you own.
- *Analogy:* in [Part D](#part-d--the-four-memory-patterns-behind-one-stable-interface) you built a tidy filing cabinet so the front desk could greet each returning customer by name and remember their usual order. Re-read the label on the drawer. It does not say "convenience features." It says *"a dossier on every person who ever walked in, kept forever, cross-indexed for fast lookup."* You didn't build a feature; you appointed yourself custodian of a records room. The cabinet didn't change — your responsibility for what's inside it did.
- *Build consequence:* The deliverable of recognizing this is an **audit**, not a paragraph of worry. For the store you built, you now owe answers to five concrete questions — lawful basis to keep each fact, data minimization (do you store only what earns its place), security at rest (is it encrypted), retention limits (does anything expire), and user rights (can a user see, fix, and delete their data). The first four you've largely built already, as the next concept shows; the fifth is the gap [Part I](#part-i--user-controls-view-export-edit-and-delete-the-right-to-be-forgotten) fills. Treat the store as a regulated subsystem from line one — secrets from env, encryption at rest, an access path, a delete path — not as a feature you'll "add governance to later."

**17. Each capability you already built maps onto a governance duty — and the one with no mapping yet is the user's right to see and control their data.**
Here is the reframe that turns dread into a checklist. You are not starting governance from zero; you are *relabeling work you've already done*. Walk the chapter backwards and the mapping falls out: **"what to store, louder what NOT to" ([Part C](#part-c--what-to-store-and-louder-what-not-to-store)) is data minimization** — the duty to keep only personal data you actually need, which is exactly the junk-drawer discipline Part C enforced for cost and quality reasons; you built the right thing for a second reason you hadn't named. **The `user_id` pre-filter in [Part D](#part-d--the-four-memory-patterns-behind-one-stable-interface) is access control** — the same isolation reflex as [Chapter 4](04-rag.md) Part L's multi-tenant pre-filter, ensuring one user's memory never assembles into another user's prompt. **The decay/TTL in [Part F](#part-f--memory-maintenance-updating-decaying-resolving-conflicts) is retention limiting** — "time to live" (TTL) is the engineering name for "don't keep personal data longer than it's useful," a duty regulators state and Part F implemented as a quality measure. Three duties, three capabilities, already standing. The fourth duty — **the user's right to access, rectify, and erase their own data** — has *no* capability behind it yet. That is the precise, bounded gap, and it is [Part I](#part-i--user-controls-view-export-edit-and-delete-the-right-to-be-forgotten)'s entire spec.

Three terms, expanded once because they name what you must build: the **right to access** is the user's right to see exactly what you've stored about them; the **right to rectify** is their right to correct a wrong stored fact; the **right to erase** (the "right to be forgotten") is their right to have it deleted. These are the three [Part I](#part-i--user-controls-view-export-edit-and-delete-the-right-to-be-forgotten) implements; the [Security, Privacy & Governance](adv-security-privacy-governance.md) chapter is where you go for *which law* obliges *which user* under *what lawful basis*.

One disambiguation to fix before you move on, because it's the predictable beginner mistake: **encrypting the store and letting the user delete from it are different obligations, and you need both.** Encryption at rest protects the data against a *breach* — a thief who steals the disk gets ciphertext, not names. Deletion honors a *right the user holds regardless of breach* — a perfectly secure, never-breached store that ignores a "delete my data" request has still failed its duty. They answer different threats: one is "an outsider must not read this," the other is "the owner of this data can make me forget it." A store can be flawlessly encrypted and wholly non-compliant. Build the lock *and* the delete button.
- *Analogy:* the store is a guest's records room. Encryption is the steel door and the alarm — it stops a burglar. The user's right to delete is the guest walking up to the front desk and saying *"shred my file."* A vault with no shredder is secure and still wrong; you can't tell the guest "your file is very safe in here" when what they asked was for it to be *gone*. The burglar and the guest are two different people with two different claims on the same drawer.
- *Build consequence:* Audit your store against all five duties and write the gap list. For most readers, after Parts C/D/F the gaps that remain are *exactly* the user-control surface: there is no way for a user to see what's stored (access), correct it (rectify), or have it deleted (erase). That three-item gap list is not a worry — it is the literal feature spec you implement in [Part I](#part-i--user-controls-view-export-edit-and-delete-the-right-to-be-forgotten). Calibrate the depth: you build the *mechanisms* (a view endpoint, an export, an edit, a fan-out delete); *which* of them a given user can legally compel, and on what basis, you read off the [Security, Privacy & Governance](adv-security-privacy-governance.md) chapter — engineering here, law there.

**Hands-on ([Part H](#part-h--memory-is-a-personal-data-store-the-governance-reflex)):** Audit your [Part D](#part-d--the-four-memory-patterns-behind-one-stable-interface) store against the four duties you can check in code, and write the gap list — it becomes [Part I](#part-i--user-controls-view-export-edit-and-delete-the-right-to-be-forgotten)'s spec. (1) **Minimization:** pull up your [Part C](#part-c--what-to-store-and-louder-what-not-to-store) "what to store" list and confirm every field in the structured profile earns its place; flag anything stored "just in case." (2) **Isolation:** write a one-shot test that calls `load_memory` for user A and asserts not one fact belonging to user B appears — the `user_id` pre-filter, proven, exactly as [Chapter 4](04-rag.md) Part L proved tenant isolation. (3) **Retention:** check whether any store carries a TTL or decay ([Part F](#part-f--memory-maintenance-updating-decaying-resolving-conflicts)); if nothing expires, write "retention: unbounded — gap." (4) **User control:** try to answer "how does a user see and delete their own data?" — for most readers the honest answer is "there's no path yet." Those gaps, written down, are the spec for the next Part.

---

## Part I — User controls: view, export, edit, and delete (the right to be forgotten)

Part H ended with a three-item gap list: a user can't *see*, *correct*, or *delete* their own data. This Part closes it with four concrete operations over the [Part D](#part-d--the-four-memory-patterns-behind-one-stable-interface) interface — `view`, `export`, `edit`, and `forget`. Three of them are short. The fourth, `forget`, is the climax: it is the hardest to get right, the most often shipped broken, and the one that vindicates the entire architecture of [Part G](#part-g--personalization-turning-stored-memory-into-a-tailored-response) — because the *only* reason a fact *can* be deleted at all is that personalization is prompt-time, not baked into weights.

**18. Ship four user-facing operations over the memory store: view, export, edit, and delete — no hidden memory.**
The three duties from concept 17 — access, rectify, erase — become four operations, because "access" splits into seeing it and taking it with you. **VIEW** shows the user *exactly* what's remembered: the structured profile fields plus a readable list of the stored memories and any summaries. The governing rule is **no hidden memory** — if a fact influences the prompt, the user must be able to see it; a memory the system acts on but won't show the user is the anti-pattern this operation exists to forbid. **EXPORT** hands that same data over in a portable format (JSON, say) — this is **data portability**, the right to take your data elsewhere rather than have it held hostage in a shape only your app can read. **EDIT/RECTIFY** lets the user correct a wrong remembered fact — and notice this also fixes [Part F](#part-f--memory-maintenance-updating-decaying-resolving-conflicts)'s wrong-fact problem *at the source*: instead of waiting for the system to detect and supersede a stale fact, the person who knows the truth corrects it directly. **DELETE/FORGET** is the right to erasure, treated on its own in concept 19 because it is structurally different from the other three.
- *Analogy:* a bank statement, an export button, a "correct my address" form, and "close my account." Three of the four are things every account-holder takes for granted — and an app that remembers you across sessions but offers *none* of them is asking for trust it hasn't earned. The reason to show the user their memory is the same reason a bank shows you your statement: the data is *about them*, so it is *theirs to see*.
- *Build consequence:* These are four functions over the one [Part D](#part-d--the-four-memory-patterns-behind-one-stable-interface) interface, not a new subsystem — `view(user_id)`, `export(user_id)`, `edit(user_id, key, value)`, `forget(user_id, fact)`. The first three are nearly free once the store exists; budget your engineering for the fourth. And enforce *no hidden memory* as a test, not a hope: whatever `view` returns must be the complete set of facts that can reach the prompt — if `assemble_context` ([Part E](#part-e--assembling-the-turn-a-memory-budget-not-a-memory-dump)) can surface a fact that `view` doesn't list, you have a hidden-memory bug.

**19. Delete is the climax: it must fan out across all four stores plus the summary plus the vector index — a half-delete is a failed delete.**
Here is the subtlety that sinks careful teams. In [Part D](#part-d--the-four-memory-patterns-behind-one-stable-interface) a single user fact does not live in *one* place — it was deliberately written into *four* stores, each for a different retrieval job. So when a user says **"forget that I work at Acme,"** that one fact is sitting in at least four locations at once:
1. the **structured profile** — a field like `employer: "Acme"`;
2. the **raw message log** — the original turn where they said "I just started at Acme";
3. the **vector index** — an embedded memory like *"works at Acme"*, sitting in the approximate-nearest-neighbor (ANN) index, retrievable by [Chapter 4](04-rag.md)'s `retrieve()` over their history;
4. the **summary** — a "previously on…" sentence such as *"Senior engineer, works at Acme, prefers Python and terse answers."*

Now watch the naive fix fail. `DELETE FROM profile WHERE user_id=? AND key='employer'` clears location 1 and feels done. But the raw log still contains "I just started at Acme." The embedded memory *"works at Acme"* is **still in the ANN index** — the next history retrieval pulls it straight back into the prompt. And the summary sentence still reads *"works at Acme"* — so even with the profile row gone, the very next turn re-tells the model the fact the user just asked it to forget. **A delete that leaves the fact in a summary or an un-reindexed vector is not a partial success; it is a failed deletion** — the user asked to be forgotten and the system still remembers, just from a different drawer. This is the same **every-sink** discipline the [Security, Privacy & Governance](adv-security-privacy-governance.md) chapter runs for GDPR erasure and PHI de-identification — the identical problem wearing the memory chapter's hat.

The repair is a **fan-out delete**: the operation must touch every store the fact can live in. Delete the profile field; purge the matching turns from the raw log; **remove the embedded memory from the vector index and re-index** (a "tombstone" marks a vector as deleted so retrieval skips it, then the index is rebuilt or upserted so it's truly gone — the same `upsert`/reindex machinery from [Chapter 4](04-rag.md) Part L); and **regenerate the summary** from the now-corrected stores so the offending sentence is rewritten without the fact. Skip any one and the fact survives.
- *Analogy:* deleting a fact is not peeling one sticky note off the monitor. It is finding **every copy you ever made** of that note — the one on the monitor, the one filed in the cabinet, the one you photographed for search, *and the line where you paraphrased it into the meeting minutes.* Miss the paraphrase in the minutes and the fact is still on record, in your own handwriting, in a place you forgot to look. The summary sentence *is* that paraphrase.
- *Build consequence:* Write `forget` as a fan-out from day one, and adopt the **fail-safe rule**: on delete, **fail toward over-deletion, never toward residue.** If you're unsure whether a memory or a summary line references the fact, delete it and regenerate — losing a little benign context is recoverable; leaving a fact the user asked to erase is a compliance failure and a betrayal of trust. And name the specific anti-pattern: a **soft-delete** that merely hides the row from the `view` UI but *still feeds it into the prompt* is the worst of both worlds — the user is shown a clean profile while the model is quietly told the fact they thought they erased. Soft-delete is a display trick; erasure is a data operation. They are not the same, and only one of them is honest.

**This is exactly why personalization had to be prompt-time, not in the weights.** Tie it back to [Part G](#part-g--personalization-turning-stored-memory-into-a-tailored-response): personalization works by *injecting* stored facts into the prompt at request time, never by fine-tuning a per-user model. That design choice looked like a simplicity preference in Part G. It is actually what makes concept 19 *possible at all.* **You can delete a row; you cannot delete a fact from a fine-tuned model's weights.** If "works at Acme" had been trained into a per-user model, "forget that I work at Acme" would have no answer short of retraining the model from scratch without that example — there is no `DELETE` for a weight. Because every personal fact lives in *data you control* (a row, a log entry, a vector, a summary string) rather than in frozen weights, every personal fact is *deletable*. The right to be forgotten is not a feature you bolted onto the architecture; it is a property the prompt-time architecture *grants you for free* — and the reason "personalize by fine-tuning per user" was the wrong answer all along.

**Hands-on ([Part I](#part-i--user-controls-view-export-edit-and-delete-the-right-to-be-forgotten)):** Implement the four operations over your [Part D](#part-d--the-four-memory-patterns-behind-one-stable-interface) interface — `view(user_id)`, `export(user_id)` (return portable JSON), `edit(user_id, key, value)`, and `forget(user_id, fact)` — then *prove* the fan-out. Seed the running user with "works at Acme" so the fact lands in all four stores (profile field, raw log, an embedded memory in the vector index, and a regenerated summary that mentions it). Call `forget(user_id, "works at Acme")`. Then confirm, with four checks, that it's actually gone: (1) the profile field is cleared; (2) the raw log no longer contains the turn; (3) **re-query the vector index for "employer" and confirm it returns nothing** — the embedded memory is tombstoned and reindexed, not merely hidden; and (4) the regenerated summary no longer contains the sentence. Finally, the proof that matters: call `assemble_context` ([Part E](#part-e--assembling-the-turn-a-memory-budget-not-a-memory-dump)) for the next turn and confirm **the assembled prompt no longer contains "Acme" anywhere** — not in the profile block, not in the retrieved memories, not in the summary. If the prompt is clean, your delete fanned out; if "Acme" reappears, trace which store you missed.

**Resources**

- [Chapter 5](05-agents.md) Part D — the within-task memory this chapter deepens to cross-session.
- The [Security, Privacy & Governance](adv-security-privacy-governance.md) chapter — the full GDPR / right-to-erasure / every-sink treatment that Parts H–I point to.
- [Microsoft Presidio](https://microsoft.github.io/presidio/) — reuse the redaction reflex so a memory store never persists raw PII it doesn't need.
- Provider "memory" features ([OpenAI memory](https://help.openai.com/en/articles/8590148), [Anthropic memory tool](https://docs.anthropic.com/en/docs/build-with-claude/tool-use/memory-tool)) — managed versions of the four Part D stores; read them as one implementation of this chapter's patterns, not the model "gaining recall."
- [pgvector](https://github.com/pgvector/pgvector) (Docker) — the store behind the raw-log + RAG-over-history patterns.

**Questions**

*Check your understanding*

1. What are the "two clocks," and which clock does the `messages` list of one conversation tick on?
2. The model is stateless, so which kind of memory does it hold on its own between requests — session or cross-session?
3. Name the four memory patterns and the one recall guarantee each is best at.
4. Why is the raw message log called "storage, not context" until you select from it?
5. In the Part E carry-on analogy, which two memory pieces are the "passport and today's clothes" that always go in, and which piece flexes to fill the rest?
6. What does "supersede-by-key" mean for a structured-profile update?
7. Is personalization prompt-time or training-time, and why does that matter for deletion?
8. At what point does an app's memory become a "personal-data store" carrying data-protection obligations?
9. Encrypting the memory store and letting the user delete from it — same obligation or different?

*Apply it*

10. A user snaps "just give me the short version" once, mid-frustration. Should `answer_style: terse` be stored as a durable fact? Why or why not?
11. Same user, two sessions a week apart. Which bytes must you persist for Session 2 to "remember" Session 1, and which do you throw away?
12. A user told the assistant "call me Dr. Reyes." Which pattern stores it, and why not RAG-over-history?
13. Where does the `user_id` pre-filter go in RAG-over-history, and what breaks without it?
14. A 4 000-token memory budget allocates 200 to profile, 1 500 to 6 recent turns, 300 to summary. At ~250 tokens per retrieved memory, how many memories fit?
15. A user said "I'm a beginner" in January and "I ship to prod daily" in June. With relevance-only ranking, why does the app patronize them, and what one change fixes it?
16. Map each memory capability you already built (Parts C, D, F) onto its governance duty, and name the duty with no mapping yet.
17. A user says "forget I work at Acme" and you run `DELETE FROM profile WHERE key='employer'`. Why is the fact not actually forgotten?

*Stretch*

18. Why is a wrong *stored* fact worse than simply having no memory at all?
19. "Remembering a conversation" and "remembering a fact about the user" look similar. How do you store each, and why differently?
20. Why hide all four behind `load_memory`/`write_memory` instead of calling each store directly from the app?
21. A user with 2 years of history (1 200 stored memories) gets the same 8 retrieved slots as a new user. Justify why this is correct, not a bug, in cost and quality terms.
22. You're tempted to fine-tune a per-user model to "really learn" each user. Give the decision-ladder rung you should use instead and three reasons fine-tuning is the wrong first move here.
23. Why does the right to be forgotten depend on personalization being prompt-time rather than fine-tuned per user?
24. What's wrong with a "soft-delete" that hides a memory row from the view UI but leaves it in the store?

**Answer key** — *peek only after attempting.*

1. The session clock (resets every conversation) and the user clock (runs for the life of the account). The `messages` list ticks on the session clock — it's born when the conversation opens and is correctly discarded when it closes.
2. Neither. It forgets the instant a request returns. Both session and cross-session memory are notes you keep and re-supply; they differ only in how long the note is allowed to live.
3. Summary buffer → the gist, cheaply; raw message log → audit/completeness; RAG-over-history → the relevant slice of the fuzzy past; structured profile → an exact named field, deterministically every time.
4. It holds everything verbatim (complete, auditable) but you can't replay months of transcripts into a prompt — that blows the context window and costs a fortune per turn. It's an archive you must *select* from (via RAG-over-history) before it's usable context.
5. Profile and recent raw turns are always included (passport + today's clothes); retrieved relevant memories flex to fill the remaining budget by relevance.
6. The key (e.g. `expertise_level`) has exactly one current value; writing a new value overwrites the old in place rather than appending a second, competing row.
7. Prompt-time: the profile is injected into the system prompt per request, so it's reversible and deletable. Training-time (fine-tuning) bakes facts into weights, which can't be individually deleted — colliding with the right-to-be-forgotten.
8. The instant you persist user facts *across sessions*. A within-task scratchpad ([Chapter 5](05-agents.md)) dies with the run and isn't regulated; cross-session memory survives the session by design, and that durability — a concentrated, indexed, per-user dossier — is exactly what makes it a regulated artifact.
9. Different, and you need both. Encryption protects against a *breach* (a thief gets ciphertext). Deletion honors a *right the user holds regardless of breach* — a flawlessly encrypted, never-breached store that ignores "delete my data" has still failed. One is "outsiders can't read this"; the other is "the owner can make me forget it."
10. No — it's a low-confidence inference from a one-off mood, not a durable/reusable/user-specific preference. Storing it poisons every future turn (Failure 2) and turns a transient signal into a persistent, personalized wrong "memory." Keep it only as a low-confidence note with its source, or drop it.
11. Persist the user-clock facts: name, expertise, language/style preference, and a 1–2 line summary of the session, each with a `written_at`. Throw away the session-clock bytes: the verbatim greeting, the exact question, and the turn-by-turn transcript.
12. A structured profile field (e.g. `name="Dr. Reyes"`), read by key deterministically every turn. RAG-over-history is similarity search — fuzzy and top-k — so it can miss or mis-rank an exact identity fact you must get right every single time; an exact, predictable field belongs in the typed profile.
13. ANDed into the retrieval `WHERE user_id = %s` (the Ch4 Part L pattern), derived from the authenticated session — never the request body. Without it one user can retrieve another's memories (a cross-user leak / IDOR); a missing `user_id` must fail-closed (refuse) rather than search everyone.
14. Remaining = 4000 − 200 − 1500 − 300 = 2000 tokens; 2000 / 250 = 8 memories (the top 8 by relevance).
15. Relevance-only surfaces the equally-relevant January "beginner" memory, so the model hand-holds. Adding `written_at` and ranking by relevance × recency-decay sinks the stale January fact below the fresh June one.
16. "What to store / what NOT to" (Part C) = data minimization; the `user_id` pre-filter (Part D) = access control; decay/TTL (Part F) = retention limiting. The unmapped duty is the user's right to access, rectify, and erase their own data — which is exactly Part I's spec.
17. That clears only the profile field. The fact still lives in three other stores: the raw log turn ("I started at Acme"), the embedded memory still in the ANN vector index (retrieval pulls it back), and the summary sentence "works at Acme." A correct delete fans out to all four plus a vector reindex/tombstone and a summary regeneration.
18. Because it's a hallucination you've made persistent and personalized: the model states it with total confidence, it's specifically about this user, and it resurfaces every session until something supersedes it. No memory just means re-asking; wrong memory means confidently misleading the user repeatedly.
19. A conversation is transient detail → summarize to 1–2 lines and drop the transcript (store-as-summary). A user fact is durable → store it structured and keep it until superseded. They decay on different clocks and stack, so conflating them either bloats the store with raw turns or loses the durable fact.
20. Real systems run 2–4 patterns together; a stable interface lets you add, drop, or re-tune a pattern by editing one function body instead of rewriting every caller. It also gives [Part E](#part-e--assembling-the-turn-a-memory-budget-not-a-memory-dump) one assembly seam and [Part I](#part-i--user-controls-view-export-edit-and-delete-the-right-to-be-forgotten) one delete fan-out point across all four stores.
21. The budget caps context size so cost/latency stay flat as history grows; and per "lost in the middle," more retrieved memories bury the few that matter and dilute the recent turns — so 8 high-relevance memories beat 1 200 dumped ones on both axes. Dropped memories aren't deleted, just absent from this turn.
22. Use rung 1 (inject preferences into the system prompt), escalating to rung 2 (retrieve past context) only if needed. Fine-tuning means one model per user (operationally absurd), costs a training run per person, and can't delete a single fact from weights — breaking the Part I deletion requirement.
23. Prompt-time personalization keeps every personal fact in *data you control* — a row, a log entry, a vector, a summary string — all of which support `DELETE`. Fine-tuning bakes the fact into frozen weights, and there is no `DELETE` for a weight; you'd have to retrain from scratch without that example. Deletability is a property the prompt-time architecture grants for free.
24. If the hidden row still feeds into `assemble_context`, the user is shown a clean profile while the model is still told the fact they thought they erased — the worst of both worlds. Soft-delete is a display trick; erasure is a data operation. The fail-safe rule is to fail toward over-deletion (delete and regenerate when unsure), never toward residue.

**Deliverable:** a **governed memory subsystem** for your [Chapter 4](04-rag.md)/[Chapter 5](05-agents.md) app, behind `load_memory()` / `write_memory()` / `assemble_context()`: the four storage patterns wired up (summary buffer, raw log, RAG-over-history with a `user_id` pre-filter, structured profile), per-turn assembly under a token budget that stays flat as history grows, `written_at` timestamps with supersede-by-key + recency-weighted retrieval, prompt-time personalization from the profile, and the four user controls — `view` / `export` / `edit` / `forget` — where `forget()` provably fans out across **every** store (profile, raw log, vector index, regenerated summary) so the fact is gone from the next prompt.

**Daily update:** one line — what you built and any open risk (e.g. "cross-session memory live behind load/write/assemble: profile + summary + RAG-over-history with user_id isolation; 4k budget holds flat at 2 years of history; forget() fans out across all four stores and the next prompt is clean — open item: summary regeneration on delete is slow, needs a queue").

**Time:** two short sessions (it deepens existing concepts rather than introducing a new mechanism). Session 1: Parts A–E (the two clocks; what to store; the four patterns behind one interface; budgeted assembly). Session 2: Parts F–I (maintenance/decay; prompt-time personalization; the governance reflex; and the view/export/edit/delete controls) plus the deliverable.
