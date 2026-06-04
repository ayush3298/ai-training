## Day 3–4 — Talking to an LLM: from first call to production-ready

**Goal:** Go from "I've never called the API" to "I can build a real feature against it." By the
end you can make calls, get reliable machine-readable output, let the model call your own code,
and handle the failures that happen with real traffic.

**Why this matters:** Everything later in this program — RAG, agents, evaluation — is assembled
from the primitives in this block. The basic call takes ten minutes to learn. The other three
skills (structured output, tool calling, error handling) are the actual line between a demo
that works once on your laptop and a feature that survives real users. We front-load them here
so every later project rests on solid ground.

> **Setup assumed:** you have an API key in your environment (`ANTHROPIC_API_KEY` or
> `OPENAI_API_KEY`) and the SDK installed (`pip install anthropic` or `pip install openai`).
> Examples are Python; the JS/TS SDKs have the same shapes. How the key gets into your
> environment is out of scope for this plan.

**Suggested split:** Day 3 = Parts A–B (basics, streaming, cost). Day 4 = Parts C–E (structured
output, tool calling, production). Do them back to back.

---

## Part A — The basics of a call

**1. A request is a list of role-tagged messages — not a string.**
The mental shift from the chat UI to the API is this: you don't send "a prompt," you send an
ordered **list of messages**, each tagged with a **role**. There are three:
- **`system`** — the standing instructions that govern the whole conversation: who the assistant
  is, what rules it follows, what format to answer in. Set once, applies throughout.
- **`user`** — input from the human or your application.
- **`assistant`** — the model's own replies. When you continue a conversation, the model's
  previous answers are included here so it can see what it already said.

This is exactly the "conversation protocol" Karpathy demonstrated with the `im_start` /
`im_end` special tokens. The API just hands you a clean, structured version of it — but
underneath, the whole list is still flattened into one token sequence and fed to the model.
Understanding that the tidy `messages` list is *secretly just one long string of tokens*
explains everything about cost and context limits later.
- *Build consequence:* You are always constructing a list. Every feature you build — chat,
  extraction, agents — is "assemble the right list of messages, send it, read the response."
  Internalize the list and the rest is variation.

**2. The system prompt is your single highest-leverage knob.**
Before any prompt-engineering technique, the system prompt is where you set behavior: *"You are
a terse support agent. Answer in two sentences or fewer. If you don't know, say you don't know.
Never invent an order number."* One field, and it shapes tone, format, scope, and guardrails for
every turn. We devote real time to writing these well in Section 2; today the goal is just to
*feel* how much changing this one string changes the output.
- *Build consequence:* When output is wrong, the system prompt is the first place you look —
  most "the model won't behave" problems are really "the instructions were vague," not "the
  model is dumb."

**3. The anatomy of a request and a response.**
A request carries a few fields you'll set constantly:
- **model** — which model (and therefore which capability/price tier — Day 2, concept 5).
- **messages** — the list above.
- **`temperature`** and optionally **`top_p`** — the sampling knobs from Day 2.
- **`max_tokens`** — a hard cap on how many tokens the model may *generate*. Too low and your
  answer is truncated mid-sentence.
- **`stop`** — optional strings that, if generated, end the response early.

A response carries three things worth caring about from day one:
- **the text** — what the model produced.
- **token usage** — input tokens and output tokens. *This is literally your bill.* Print it on
  your very first call and never stop being aware of it.
- **a stop / finish reason** — *why* generation ended: it finished naturally, it hit
  `max_tokens`, or it hit a stop sequence. This is your first debugging tool: truncated output
  almost always means the finish reason was `max_tokens`/`length`.
- *Build consequence:* Treat usage and the finish reason as first-class outputs, not
  afterthoughts. They are how you diagnose cost and truncation before they become production
  incidents.

**4. The API is stateless — "memory" is an illusion you maintain.**
This is the concept people get wrong most often. **The model remembers nothing between calls.**
Each request is processed from scratch. A chatbot appears to "remember" the conversation only
because **your code keeps the running list of messages and resends the entire list every single
time.** This is Day 2's "context = working memory" made literal: the *only* thing the model
knows about turn 1 when it's answering turn 5 is that turn 1 is still sitting in the `messages`
list you sent.

Two direct consequences fall out of this:
- **Cost climbs every turn.** The history grows, so you resend more input tokens with each
  message. A 30-turn conversation can be re-sending thousands of tokens on every reply.
- **Conversations have a ceiling.** Eventually the accumulated history approaches the model's
  context-window limit, and you must trim or summarize older turns (a real technique we'll use
  later).
- *Build consequence:* "Conversation memory" is a thing *you* build and manage, not a feature
  the API provides. If a bot forgets, the bug is in your message-list handling, not the model.

**The same basic call, two providers:**

```python
# Anthropic (Claude)
import anthropic
client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY from env

resp = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system="You are a terse assistant. Answer in one sentence.",  # top-level param
    messages=[{"role": "user", "content": "What is the capital of France?"}],
    temperature=0.2,
)
print(resp.content[0].text)     # the reply
print(resp.usage)               # input_tokens, output_tokens
print(resp.stop_reason)         # "end_turn" | "max_tokens" | "stop_sequence"
```
```python
# OpenAI
from openai import OpenAI
client = OpenAI()  # reads OPENAI_API_KEY from env

resp = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "You are a terse assistant. Answer in one sentence."},
        {"role": "user", "content": "What is the capital of France?"},
    ],
    temperature=0.2, max_tokens=1024,
)
print(resp.choices[0].message.content)   # the reply
print(resp.usage)                        # prompt_tokens, completion_tokens, total_tokens
print(resp.choices[0].finish_reason)     # "stop" | "length" | ...
```

**The three differences that trip everyone up:**

| | Anthropic | OpenAI |
|---|---|---|
| System prompt | top-level `system=` parameter | a `{"role":"system"}` message in the list |
| Reply location | `resp.content[0].text` | `resp.choices[0].message.content` |
| Usage field names | `input_tokens` / `output_tokens` | `prompt_tokens` / `completion_tokens` |

> Model names change often — `claude-sonnet-4-6` and `gpt-4o` are just examples. Check the
> provider's models page (you looked at it on Day 2) for the current id, context size, and price.

**Multi-turn — the one line that creates "memory":**
```python
messages = [{"role": "system", "content": "You are a helpful assistant."}]
while True:
    user = input("you> ")
    if user.strip() in {"quit", "exit"}: break
    messages.append({"role": "user", "content": user})

    resp = client.chat.completions.create(model="gpt-4o", messages=messages)
    reply = resp.choices[0].message.content
    print("bot>", reply)

    messages.append({"role": "assistant", "content": reply})  # <-- THIS is the "memory"
```
Delete that last append and the bot forgets everything between turns — proof that memory lives
in your list, not the model.

---

## Part B — Streaming and cost

**5. Streaming changes the UX, not the answer.**
By default you wait for the entire response, then receive it in one piece. With **streaming**,
you receive tokens as they're generated — which is why chat UIs appear to "type." The final
text is identical; what changes is *perceived* latency (the user sees progress immediately) and
the ability to **cancel early** or show a live view. The trade-off is more complex code: you
consume an event stream instead of reading one object.

```python
# OpenAI streaming
stream = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Write a two-line poem about tokens."}],
    stream=True,
)
for chunk in stream:
    delta = chunk.choices[0].delta.content
    if delta:
        print(delta, end="", flush=True)
```
```python
# Anthropic streaming
with client.messages.stream(
    model="claude-sonnet-4-6", max_tokens=256,
    messages=[{"role": "user", "content": "Write a two-line poem about tokens."}],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```
- *Build consequence:* Stream in anything a human watches in real time (chat, assistants).
  Don't bother streaming inside a background job that just needs the final string — it adds
  complexity for no user benefit.

**6. Cost math — read the bill on day one.**
Providers price **per million tokens (MTok)**, and **input and output are priced separately** —
output is typically several times more expensive than input. The formula for a single call:

```
cost = (input_tokens  / 1_000_000) * price_in_per_mtok
     + (output_tokens / 1_000_000) * price_out_per_mtok
```

Worked example (illustrative numbers — always check current pricing): input at $3/MTok, output
at $15/MTok. A call that uses 2,000 input tokens and 500 output tokens costs
`(2000/1e6 × 3) + (500/1e6 × 15)` = $0.006 + $0.0075 = **$0.0135**. That's tiny — until you
multiply by 100,000 calls/day (~$1,350/day). Two lessons fall out:
- **Output dominates.** At a 5× output premium, 500 output tokens cost more than 2,000 input
  tokens. Verbose responses are where the money goes — cap `max_tokens` and ask for concise
  output when you can.
- **System prompts are a recurring tax.** Every call resends the system prompt as input tokens,
  so a bloated system prompt is a cost you pay on every request forever.
- *Build consequence:* Estimate cost from real usage *before* you ship, not after the bill
  arrives. "What does one call cost, and how many calls/day?" is a design question, not an
  afterthought.

---

*(Split point — Day 3 ends here; Day 4 begins.)*

---

## Part C — Structured output (machine-readable, every time)

When a *human* reads the output, free text is fine. When your *code* must use the output —
store it, branch on it, pass it to another function — you need data you can parse reliably on
every single call, including the awkward ones. There are three levels of reliability, and the
jump between them matters:

1. **Ask in the system prompt** — *"Respond only with JSON shaped like `{...}`."* This works
   most of the time and fails the rest: the model wraps it in markdown ` ```json ` fences, adds
   a chatty preamble ("Sure! Here's your JSON:"), or emits a trailing comma. Fine for a
   prototype, **not safe for production on its own**, because the 2% of malformed responses
   become 2% of your requests crashing.
2. **JSON mode** — the API guarantees the output is **syntactically valid JSON**, so
   `json.loads()` won't throw. It does *not* guarantee your *shape* (the right keys/types).
   OpenAI: `response_format={"type": "json_object"}`.
3. **Schema-enforced structured output** — you supply a JSON Schema and the provider constrains
   generation so the output **matches your schema** (right keys, right types). OpenAI exposes
   this as `response_format` with a `json_schema` and `strict: true`, and a Pydantic helper that
   returns a parsed object. Anthropic achieves the same result through **tool calling** — you
   define a tool whose input schema *is* your desired shape, and the model "calls" it with
   structured arguments.

Regardless of level, **still parse and validate defensively** (e.g. with Pydantic). Schema
enforcement is strong, but validation at your boundary is cheap insurance and documents the
contract.

```python
# OpenAI — schema-enforced structured output via Pydantic
from pydantic import BaseModel

class Ticket(BaseModel):
    category: str
    urgency: int       # 1–5
    summary: str

resp = client.beta.chat.completions.parse(
    model="gpt-4o",
    messages=[{"role": "user", "content": "My payment failed twice and now I'm locked out. Furious."}],
    response_format=Ticket,
)
ticket = resp.choices[0].message.parsed     # a validated Ticket instance
print(ticket.category, ticket.urgency, ticket.summary)
```
- *Build consequence:* The boundary between "the LLM" and "the rest of your program" should
  almost always be **validated structured data, not prose.** Design that contract explicitly —
  decide the schema first, then prompt toward it — and your app stops being at the mercy of
  free-text formatting.

---

## Part D — Tool / function calling (the foundation for agents)

The single most important sentence in this block: **the model never runs your code. It only
*requests* a call; your program decides whether and how to run it, then feeds the result back.**
That control boundary is what makes tool use safe and predictable.

The round trip has five steps (six if it loops):
1. You describe the **tools** available — each with a name, a description (the model reads this
   to decide *when* to use it), and a JSON-schema for its parameters.
2. You send the user message *plus* the tool definitions.
3. The model responds in one of two ways: a normal answer, **or** a **tool call** — "call
   `get_weather` with `{city: 'Pune'}`."
4. **Your code executes the function** with those arguments and appends the result to the
   message list as a tool-result message.
5. You call the model again with the updated list; now it has the data and writes the final
   answer.
6. If the task needs several tools, this repeats — the model calls, you execute, you return, it
   calls again. **That loop is exactly what an "agent" is** (Section 5), so understanding it
   here is foundational.

```python
# OpenAI — one tool-calling round trip
import json

tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "Get the current weather for a city.",
        "parameters": {
            "type": "object",
            "properties": {"city": {"type": "string"}},
            "required": ["city"],
        },
    },
}]

messages = [{"role": "user", "content": "What's the weather in Pune?"}]
resp = client.chat.completions.create(model="gpt-4o", messages=messages, tools=tools)
msg = resp.choices[0].message

if msg.tool_calls:
    call = msg.tool_calls[0]
    args = json.loads(call.function.arguments)     # {"city": "Pune"}
    result = get_weather(args["city"])             # <-- YOUR function actually runs here

    messages.append(msg)                           # the assistant's tool request
    messages.append({                              # the tool's result, fed back in
        "role": "tool",
        "tool_call_id": call.id,
        "content": json.dumps(result),
    })
    final = client.chat.completions.create(model="gpt-4o", messages=messages, tools=tools)
    print(final.choices[0].message.content)        # answer that uses the live weather
```
Anthropic uses the same five-step dance with different names: tools go in a `tools=` list, the
model returns a `tool_use` content block, and you reply with a `tool_result` block.
- *Build consequence:* Tools are how you *fix* every limitation from Day 2. Can't do math
  reliably? Give it a calculator. Knowledge is stale? Give it search. Needs your data? Give it a
  database query tool. The model decides *when* to call; **you stay fully in control of what
  actually executes** — which is also where you put your safety checks.

---

## Part E — Production basics (demo → survives real traffic)

A script that works once on your laptop is not a feature. Real APIs fail in predictable ways,
and handling them is the difference. Know these:

- **Error types, by HTTP status:**
  - `401` — auth (bad/missing key). Your bug. **Don't retry.**
  - `400` — bad request (malformed input, too many tokens for the model's window). Your bug.
    **Don't retry.**
  - **`429` — rate limit** (too many requests or tokens per minute). Transient. **Retry after
    waiting.**
  - `500` / `503` — provider server error/overload. Transient. **Retry.**
  - **timeouts** — the call took too long (reasoning models and long outputs can take tens of
    seconds).
- **Retries with exponential backoff + jitter:** on `429`/`5xx`, wait and retry with increasing
  delays (1s, 2s, 4s…) plus a little randomness ("jitter") so a fleet of clients doesn't retry
  in lockstep. The official SDKs do a few automatic retries — know it's happening and tune
  `max_retries`.
- **Respect `Retry-After`:** a `429` response often tells you exactly how long to wait — use it
  instead of guessing.
- **Always set a timeout:** never let a request hang forever; pick a ceiling and fail gracefully
  past it.
- **Never retry `400`/`401`:** those won't succeed on retry — you'd just burn calls and money on
  your own bug.
- **Log usage and errors:** you can't control cost or reliability you don't measure. Capture
  token usage, latency, and error rates from day one.

```python
from openai import OpenAI
client = OpenAI(max_retries=4, timeout=30.0)   # built-in backoff + a hard timeout

try:
    resp = client.chat.completions.create(model="gpt-4o", messages=messages)
    reply = resp.choices[0].message.content
except Exception as e:
    # Log it; degrade gracefully — a cached answer, a retry queue, or a friendly
    # "please try again." NEVER let one failed call crash the whole request.
    reply = "Sorry, I'm having trouble right now — please try again in a moment."
    print("LLM call failed:", e)
```
- *Build consequence:* Assume every call can fail, and design for **graceful degradation** — a
  fallback message, a cached response, or a retry — rather than an exception bubbling up to the
  user. Reliability is a feature you build around the model, not something the model gives you.

---

### Resources
- Your provider's **API reference** for `messages` / `chat.completions` — skim the request and
  response fields you used today.
- The provider's **tool use / function calling** guide and **structured outputs** guide — one
  read each.
- The provider's **rate limits** and **error codes** pages — know your tier's limits before you
  hit them.

---

### Hands-on
**A. First call.** Send a system + user message; print reply, usage, and stop reason. Confirm
usage roughly matches your Day-2 tiktokenizer estimate.
**B. System-prompt swap.** Same user question, three system prompts ("one word" / "like a
pirate" / "as JSON `{\"answer\":...}`"). Note the control you get.
**C. Multi-turn loop.** Build the loop; ask a turn-2 question that depends on turn 1 ("capital
of France?" → "population of *that city*?"). Then delete the assistant-append and watch it break.
**D. Cost.** Print usage every turn of a 5-turn chat; compute the $ cost of each call from
current pricing; explain why input tokens climb.
**E. Streaming.** Convert call A to streaming; observe tokens arriving.
**F. Structured output.** Extract `{category, urgency, summary}` from 3 support messages into a
validated object; feed it one message designed to be awkward and confirm it still parses.
**G. Tool calling.** Implement the `get_weather` round trip with a stubbed function returning
fake data; confirm the final answer uses your result. Bonus: add a second tool and ask a
question that needs both.
**H. Break it on purpose.** Trigger a `400` (e.g. ask for more tokens than the model allows) and
a timeout; confirm your error handling degrades gracefully instead of crashing.

---

### Questions

**Check your understanding (one sentence each):**
1. What are the three message roles, and what is each for?
2. Where does the reply text live in the response for Anthropic vs OpenAI?
3. The API is stateless — so how does a chatbot "remember" earlier turns?
4. Which two numbers in the response tell you what a call cost, and which usually dominates?
5. Output got cut off mid-sentence. Which field tells you why, and what's the fix?
6. In tool calling, who actually executes the function — the model or your code?
7. Why is "ask for JSON in the system prompt" not sufficient for production, and what's the
   stronger option?
8. Name two error statuses you should retry and two you should not.
9. What does streaming change, and what does it *not* change?
10. Why does a long conversation get more expensive with every turn?

**Apply it (2–3 sentences each):**
11. A teammate's multi-turn bot forgets everything after each message. What's the most likely
    bug?
12. You need the model's output to populate a database row with fields `name`, `amount`, `date`.
    Which level of structured output do you use, and what do you still do after receiving it?
13. Under load your app starts throwing `429`s and failing user requests. What's happening and
    what two changes do you make?
14. You're porting a working script from OpenAI to Anthropic. List the three concrete code
    changes.
15. A user asks your assistant "what's 4,891 × 2,307?" and it confidently returns a wrong
    number. Using Days 2–3, explain why and give the fix.

**Stretch / discussion (optional):**
16. Your assistant needs both live order data and the ability to do arithmetic. Sketch how the
    tool-calling loop handles a question that requires two tool calls in sequence.
17. A high-traffic feature has a 2,000-token system prompt. Estimate the *daily* extra cost of
    those system tokens at 200k calls/day (input $3/MTok), and suggest a mitigation.

**Answer key (peek only after attempting):**
1. `system` = standing instructions/persona/format; `user` = input; `assistant` = the model's
(and prior) replies. · 2. Anthropic `resp.content[0].text`; OpenAI
`resp.choices[0].message.content`. · 3. Your code resends the entire message history (including
past assistant turns) every call. · 4. Input/prompt tokens and output/completion tokens; output
usually dominates because it's priced higher. · 5. The stop/finish reason (`max_tokens`/
`length`) — raise `max_tokens`. · 6. Your code does; the model only *requests* the call. ·
7. The model may wrap/garnish/break the JSON; use JSON mode or (better) schema-enforced
structured output, and still validate. · 8. Retry `429` and `500/503`; don't retry `400` or
`401`. · 9. It changes perceived latency/UX (tokens arrive live); it doesn't change the final
text. · 10. The history grows, so you resend more input tokens each turn. · 11. The code isn't
persisting/appending turns to the messages list (or rebuilds it empty each call). · 12.
Schema-enforced structured output (or JSON mode + a schema); still parse and validate (e.g.
Pydantic) at the boundary. · 13. You're exceeding your rate limit; add exponential backoff +
jitter on `429` and respect `Retry-After` (and/or request a higher limit / spread load). ·
14. Move `system` to the top-level `system=` param, read `content[0].text`, use
`input_tokens`/`output_tokens`. · 15. Token-based mental arithmetic is unreliable (Day 2); give
it a calculator tool (Day 3) and have it call that instead of computing in its head. · 16. The
model calls tool 1 → you execute and return the result → you call the model again → it calls
tool 2 → you execute and return → it composes the final answer; the loop repeats until it
answers with no tool call. · 17. 2,000 × 200,000 = 4e8 input tokens/day → 400 MTok × $3 =
**$1,200/day** just for the system prompt; mitigate by trimming the prompt, or using prompt
caching if the provider offers it.

---

**Deliverable:** a small program that (a) holds a multi-turn conversation printing token usage
per turn, (b) extracts a validated structured object from free text, and (c) completes one
tool-calling round trip with a stubbed function — plus a short note covering what the system
prompt changed, what broke when you removed the assistant-append, your per-call cost estimate,
and one error you triggered and handled. Attach answers to Q1–15.

**Daily update (DM to Ayush):** one-liner — done / where you stopped / blockers.

**Time:** ~5–6 hours total (≈2.5–3 hrs/day across Day 3 and Day 4).

---

