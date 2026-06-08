# Day 30 — Memory, part 2 — maintenance, personalization & user controls

> [← Day 29](day-29.md) · [All days](README.md) · [Day 31 →](day-31.md)

**Module:** Memory & Personalization  ·  **Time:** ~2.5 hrs

## Where we are

_Continues **Memory & Personalization**. Earlier days covered Parts A, B, C, D, E; today picks up where they left off._

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

---

## Module wrap-up — hands-on, questions & deliverable

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

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
