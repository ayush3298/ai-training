# Day 3 — APIs, part 1 — first call, streaming & cost

> [← Day 2](day-02.md) · [All days](README.md) · [Day 4 →](day-04.md)

**Module:** APIs & Integration  ·  **Time:** ~2.5–3 hrs

## About this module

### Chapter 2 — Talking to an LLM: from first call to production-ready

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
every turn. We devote real time to writing these well in [Chapter 3](03-prompt-engineering.md); today the goal is just to
*feel* how much changing this one string changes the output.
- *Build consequence:* When output is wrong, the system prompt is the first place you look —
  most "the model won't behave" problems are really "the instructions were vague," not "the
  model is dumb."

**3. The anatomy of a request and a response.**
A request carries a few fields you'll set constantly:
- **model** — which model (and therefore which capability/price tier — [Chapter 1](01-foundations.md), concept 5).
- **messages** — the list above.
- **`temperature`** and optionally **`top_p`** — the sampling knobs from [Chapter 1](01-foundations.md).
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
time.** This is [Chapter 1](01-foundations.md)'s "context = working memory" made literal: the *only* thing the model
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
> provider's models page (you looked at it in [Chapter 1](01-foundations.md)) for the current id, context size, and price.

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

*(Split point — Session 1 ends here; Session 2 begins.)*

---

## Interview drill (self-test)
Real Gen-AI interview questions on today's material — *answer from memory, then check against what you just read.*

1. What are Copilot APIs?
2. How does temperature work?
3. How do you version ML APIs safely?
4. How do I integrate the OpenAI API?
5. How are roles defined in OpenAI's API?
6. Explain the temperature parameter in LLMs.
7. How to implement file upload using FastAPI?
8. In MedSphere, why did you use a temperature of 3?
9. What is the temperature parameter in OpenAI/LLMs?
10. What are the different types of APIs in Amazon Lex?
11. How do you build scalable APIs with Flask for GenAI?
12. How did you handle LLM API integration in Sonara.ai?
13. What does the temperature parameter control in AI models?
14. How would you design an API to handle ML models and inference?
15. How does temperature and top-k influence the behavior of LLM outputs?

_(31 matched; 15 shown — more in [`ai-eng-questions.md`](../ai-eng-questions.md).)_

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
