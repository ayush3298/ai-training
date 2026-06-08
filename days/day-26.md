# Day 26 — Beyond Text, part 1 — the encode→decode model & vision-in

> [← Day 25](day-25.md) · [All days](README.md) · [Day 27 →](day-27.md)

**Module:** Beyond Text  ·  **Time:** ~2.5–3 hrs

## About this module

### Beyond Text — Vision, Voice & Generation: shipping multimodal features you can trust

**Goal:** By the end you can build features that go beyond plain text in and out: extract structured
data from documents, screenshots, charts and multi-image inputs (vision-in); turn speech into text
with speaker labels (audio-in / STT + diarization); turn text into speech (audio-out / TTS); reason
about the latency budget of a realtime voice agent and wire turn-taking; generate and edit images
(and reason about video generation) through an API; attach and check **provenance** (C2PA) on what
you generate; and evaluate and cost a multimodal feature the same disciplined way you cost a text
one. The deliverable verb stays **CALL/CONSUME** throughout — you call hosted vision, speech, and
generation models and turn their knobs; you never train one.

**Why this matters:** Most real-world data is not clean text: invoices, receipts, screenshots,
scanned PDFs, phone calls, voice notes, and the images and video your product must produce. The
biggest capability jump for an API-consumer after text is learning that "read this document",
"transcribe this call", "speak this answer", and "generate this image" are each now *mostly one API
call*. But each modality drags in a failure mode text never had: a vision model silently mis-reads a
digit, an STT model drops the wrong speaker, a voice agent feels broken at 900ms of latency a
chatbot wouldn't notice, and a generated image carries legal and provenance obligations. This
chapter ties each new capability to its specific new failure mode and its new knob — so you ship
multimodal *with confidence, not vibes*. It also closes [Chapter 3](03-prompt-engineering.md)'s
one-paragraph multimodal promise (its Part G showed an image dropped into the messages list; this
chapter is the full sequel).

> **Setup assumed:** you can read side-by-side Anthropic and OpenAI Python and run a local service in
> Docker (used for the self-hosted STT/TTS/image-gen variants later in the chapter). Reuse
> [Chapter 2](02-apis-and-integration.md)'s call / usage / structured-output / cost primitives and
> [Chapter 3](03-prompt-engineering.md)'s multimodal seed (image-in-the-messages-list). Secrets come
> from the environment, never hard-coded.

---

## Part A — The one mental model: every modality is encode-then-decode

**1. Images and audio are "just more tokens" — and that one sentence is the whole chapter's spine.**
[Chapter 1](01-foundations.md) told you an LLM is a next-token predictor over tokens, and
[Chapter 3](03-prompt-engineering.md)'s Part G dropped an image into the messages list with the
one-liner *"images and audio are just more tokens."* Make that literal. Here is the pipeline, in
prose, with no boxes:

```
photo ──▶ [vision ENCODER] ──▶ ~N image tokens ──┐
                                                 ├──▶ the SAME transformer you already call ──▶ text tokens out
text  ──▶ [tokenizer]       ──▶ M text tokens  ──┘
```

The photo never reaches the transformer as pixels. A **vision encoder** turns it into roughly *N*
image tokens — vectors that live in the exact same space as the text tokens your prompt became — and
then the *same* model you've been calling all course reasons over the concatenated stream and emits
text tokens. Audio is the same story with a different front door: a waveform goes through an audio
encoder into tokens, the model reasons, and (for speech-out) a **decoder** turns tokens back into a
waveform.

The analogy to hold onto is a **universal translator booth**. Every language coming in — pixels,
waveform, text — is translated into one shared internal language (tokens). The thinking happens
*once*, in that shared language. Then, if the output needs to leave as something other than text
(speech, an image), a **decoder** translates the shared language back out. Encode in → reason in the
shared space → decode out. Every one of the seven Parts in this chapter is just this single pipeline
viewed from a different stage: vision-in is the encode side for images, STT is the encode side for
audio, TTS and image generation are the decode side, and the realtime voice agent is the whole loop
run end-to-end against a clock.

- *Build consequence:* You debug a misbehaving multimodal feature at a **stage**, not as one black
  box. Bad encode (the model never "saw" the digit clearly), bad reasoning (it saw it but answered
  the wrong question), or bad decode (it reasoned correctly but the spoken/generated output is
  wrong)? Naming the stage tells you *which* knob to reach for and which Part of this chapter owns
  the fix — instead of re-rolling the whole prompt and hoping.

**2. The one decision that organizes everything: one native multimodal model, or a pipeline of
specialists you chain.** There are two ways to get a non-text capability, and almost every design
choice in this chapter is an instance of picking between them:

- **One native multimodal model** that already sees and/or hears — you hand a single chat model the
  image or audio in the messages list (exactly [Chapter 3](03-prompt-engineering.md)'s pattern) and
  it reasons over everything at once. One call, one bill, the encoder is built in.
- **A pipeline of specialist models you chain** — a dedicated OCR engine, a speech-to-text model, a
  text-to-speech model, an image generator — each a separate API doing one stage, wired together by
  your code.

Treat this as a **defaults-first ladder**, not a coin flip:

> **Reach for the native multimodal chat model first.** Drop to a specialist API only when you need
> a specific knob the native model doesn't expose.

The native model is fewer moving parts, one auth, one trace, and it reasons over the raw input with
full context. You step down to a specialist when you need something it structurally can't give you —
**word-level timestamps** (when exactly was each word spoken), **speaker diarization** (who said
which line), **bounding boxes** (where on the page a value sits), streaming **barge-in** for a voice
agent, or per-image generation controls like a fixed seed. Those are concrete capabilities, not
vibes — and "I need a knob the chat model doesn't have" is the only good reason to take on the extra
pipeline. Reaching for a four-stage specialist pipeline when one native call would do is the
predictable beginner over-build here; so is assuming "multimodal = one magic model" and being
surprised when you need a timestamp it can't produce.

- *Build consequence:* For every capability in this chapter you'll first ask "can the native chat
  model just do this?" and only assemble a specialist pipeline when you can *name the missing knob*.
  That single question keeps your system as simple as the requirement allows, and it makes the build
  vs. drop-down choice defensible in review instead of habitual.

*Scope note (forward-pointer).* This chapter is strictly about **calling** these models well. The
machinery *inside* the booth — how a vision encoder is architected, CLIP / contrastive training, the
diffusion math behind image generation, how an audio codec is learned — is **out of scope: you never
train these, you call them.** We treat the encoder and decoder as hosted black boxes with knobs and a
bill, the same way [Chapter 2](02-apis-and-integration.md) treated the text model. When a later Part
needs you to understand a knob (a generation **seed**, a diarization setting), it explains that knob —
not the model internals beneath it.

**Hands-on ([Part A](#part-a--the-one-mental-model-every-modality-is-encode-then-decode)):** take one
photo of a receipt and send it to a *native multimodal chat model*, Anthropic and OpenAI side by
side, and **print the token usage** — proving for yourself that "images are just (a lot of) more
tokens." This reuses [Chapter 3](03-prompt-engineering.md)'s image-in-the-messages-list shape and
[Chapter 2](02-apis-and-integration.md)'s usage primitive; no new infrastructure.

```python
import base64, os

# One shared input: the receipt photo, base64-encoded.
img_b64 = base64.standard_b64encode(open("receipt.jpg", "rb").read()).decode()
PROMPT = "What is the total on this receipt? Answer with just the number."

# --- Anthropic ---------------------------------------------------------------
from anthropic import Anthropic
anthropic = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])
a = anthropic.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=128,
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": PROMPT},
            {"type": "image", "source": {
                "type": "base64", "media_type": "image/jpeg", "data": img_b64}},
        ],
    }],
)
print("anthropic:", a.content[0].text)
print("anthropic usage:", a.usage)   # input_tokens INCLUDES the image's token cost

# --- OpenAI ------------------------------------------------------------------
from openai import OpenAI
openai = OpenAI(api_key=os.environ["OPENAI_API_KEY"])
o = openai.chat.completions.create(
    model="gpt-4o",
    max_tokens=128,
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": PROMPT},
            {"type": "image_url",
             "image_url": {"url": f"data:image/jpeg;base64,{img_b64}"}},
        ],
    }],
)
print("openai:", o.choices[0].message.content)
print("openai usage:", o.usage)      # prompt_tokens INCLUDES the image's token cost
```

Now do the experiment that makes the point: run the **same prompt text with no image**, and compare
the input-token counts. The image typically costs **hundreds to ~1.5k tokens** on its own —
*far* more than your one-line text prompt — and that count scales with the image's resolution. Note
the two numbers down. That gap *is* the "encode" stage showing up on your bill, and it's the seed for
Part G's multimodal cost math: when you cost a vision feature, the image is the line item that
dominates, and you size it by resolution, not by the length of your question.

## Part B — Vision-in: documents, tables, charts & multi-image

**3. Sending an image to a chat model and asking for schema-enforced output replaces a whole category of brittle OCR pipelines — but the model can silently mis-read, so you must scaffold for it.** For years, pulling data off a document meant a multi-stage **OCR** pipeline — *Optical Character Recognition*, the classic technology that turns pixels of text into character strings — followed by hand-written rules to glue those characters back into fields. That pipeline is fragile in a very specific way, and naming the fragility is what tells you what to build next.

Hold two failure modes side by side, because they are opposites:

- **Classic OCR gives you characters but loses structure.** It reads `1,240.00` correctly and reads `Subtotal` correctly, but it does not reliably know that `1,240.00` is *the subtotal* — that it belongs in *that* row, *that* column. On a clean line of prose OCR is excellent; on a table it spills the cells onto the floor and you spend your time writing positional heuristics to put them back.
- **A multimodal LLM gives you structure but can hallucinate a character.** A **vision language model (VLM)** — a chat model with a vision encoder bolted on the front, exactly the native-multimodal model from [Part A](#part-a--the-one-mental-model-every-modality-is-encode-then-decode) — reasons over the *whole* page at once. Ask it for the subtotal and it confidently returns the subtotal, in the right field, because it understood the layout. But because it is a next-token predictor over the encoded image, it can read `1,240.00` and emit `1,240.50` — a plausible, well-formatted, *wrong* digit. OCR drops a character noisily (you can see the garbage). A VLM substitutes one silently (it looks perfect).

The analogy: classic OCR is a **photocopier** — every mark faithfully reproduced, but it has no idea what a "total" is, so it hands you a pile of disconnected numbers. The VLM is a **fast clerk who reads the whole form and fills in your spreadsheet** — they understand the document, they put every value in the right box, and occasionally, reading quickly, they transcribe a `7` as a `1` without noticing. You would never let that clerk's work go straight to the ledger unchecked. That is the entire shape of vision extraction.

So the repair has two moves, and neither is optional for documents that matter:

1. **Ask for schema-enforced structured output.** This is the *exact* mechanism from [Chapter 2, Part C](02-apis-and-integration.md) — you supply a JSON Schema (or a Pydantic model) and the provider constrains generation so the keys and types come back right. We do not re-teach it here; we *reuse* it. It solves the "lost structure" problem for free: you get `{"line_items": [...], "total": ...}`, not prose you have to scrape.
2. **Add a verification pass that the pure-text world never needed.** Schema enforcement guarantees the *shape* is right (`total` is a number); it says nothing about whether that number is the *correct* number. Text extraction from a digital document doesn't have this gap — the characters are already exact. Vision does, because the encode step can mis-read. So you add a check.

- *Build consequence:* A vision extraction feature is **two stages, not one**: extract-into-schema, then verify-before-trust. If your design has only the first stage, you have shipped the silent-misread trap. The next concept is how to verify cheaply.

**4. The cheapest verification is an arithmetic constraint the document already carries — make the line-items prove themselves by summing to the stated total.** You don't need a second AI to check the first one's homework when the document contains a relationship that *must* hold. An invoice is the canonical case: every line-item amount, added up, equals the subtotal; subtotal plus tax equals the total. That is a closed arithmetic loop. If the model mis-read one digit, the loop almost certainly won't close — and now the silent failure is loud.

Work it by hand on one tiny invoice so the trap is visible:

```
Line items the model returned:
  Widget A    2 × 30.00 = 60.00
  Widget B    1 × 45.00 = 45.00
  Widget C    3 × 15.00 = 45.00
  ---------------------------------
  Sum of line items      = 150.00
  Total the model read   = 150.00      ✓ matches → trust it
```

Now suppose the encode step mis-read Widget C's unit price as `15.00` when the page actually said `19.00`. The model returns line items summing to `150.00`, but it *also* read the printed total — `162.00` — into the `total` field:

```
  Sum of line items      = 150.00
  Total the model read   = 162.00      ✗ 12.00 gap → DON'T trust; re-verify
```

The 12.00 discrepancy (3 × the 4.00 per-unit error) is the silent misread made audible by arithmetic you can check on a napkin. The repair when the check fails is a **re-ask**: send the image back with the discrepancy named — *"the line items you extracted sum to 150.00 but you reported a total of 162.00; re-read the document carefully and correct the error."* A second pass with the contradiction pointed out very often fixes it, because you've turned a silent task into a constrained one.

The analogy is **double-entry bookkeeping**: accountants don't trust a ledger because the numbers look neat; they trust it because two independent tallies agree. Your sum-check is the second tally.

- *Build consequence:* When the data has an internal constraint (line-items sum to total, debits equal credits, percentages add to 100), encode that constraint as an automatic assertion and re-prompt on failure — it catches the exact error class vision introduces, at zero extra model cost on the happy path. When the data has *no* such constraint (a chart with no totals, a free-form form), you fall back to a costlier verification — a second extraction you compare, or human review on low-confidence fields. Either way, full evaluation of *how often* the misread happens is a measurement job, deferred to [Part G](#part-g--multimodal-evaluation--cost).

**5. Two knobs decide every vision call — the detail/resolution tier (how hard the encoder looks) and how many images ride in one message — and a decision table decides whether you reach for a VLM at all.** Three loose ends from the previous concepts tie off here: the resolution knob, multi-image, and the OCR-vs-LLM choice.

*The detail/resolution tier.* Providers let you choose how finely the image is encoded — Anthropic resizes by pixel budget; OpenAI exposes an explicit `detail` of `"low"`, `"high"`, or `"auto"`. The analogy is **JPEG quality**: `"low"` ships a small, cheap version — fewer image tokens, faster, cheaper — but it *drops the fine print*, exactly the small digits and dense table text you most need on a document. `"high"` tiles the image and encodes each tile, costing more tokens but preserving the 8-point footer. So the default flips by content: **`"low"` for "is there a cat in this photo?", `"high"` (the default) for any document with small text, totals, or dense tables.** Choosing `"low"` to save tokens on an invoice is a classic way to *manufacture* the misreads of concept 4.

*Multi-image.* You are not limited to one image. The mental model from [Part A](#part-a--the-one-mental-model-every-modality-is-encode-then-decode) makes this trivial: **each image is just another block of tokens in the same user message.** Passing two images and asking *"what changed between these?"* — a before/after screenshot, two revisions of a contract — is the same call shape with a second image block appended. The cost simply stacks (two images = two images' worth of tokens), which is the seed for [Part G](#part-g--multimodal-evaluation--cost)'s cost math.

*The decision: dedicated OCR / Document AI vs the multimodal LLM.* **Document AI** is the category of purpose-built document services — AWS Textract, Azure AI Document Intelligence, the open-source **Docling** — that do OCR *plus* layout analysis and return characters anchored to coordinates. Per [Part A](#part-a--the-one-mental-model-every-modality-is-encode-then-decode)'s ladder, reach for the native VLM first; step down to Document AI when you can name the knob it gives you that the VLM can't:

| Axis | Dedicated OCR / Document AI | Multimodal LLM (VLM) |
|------|------------------------------|------------------------|
| **Character fidelity** | High — purpose-built; rarely "invents" a digit | Can silently substitute a character |
| **Structure / semantics** | Returns boxes; *you* assemble meaning | Native — understands "this is the subtotal" |
| **Bounding boxes** (where on the page a value sits) | Yes — exact coordinates | No (not reliably) — the knob to drop down for |
| **Hallucination risk** | Low | Real — needs the verification pass |
| **Layout complexity** it handles | Strong on standard forms; rules get brittle on novel layouts | Strong on novel/messy layouts it has never seen |
| **Cost** | Per-page pricing | Per-token, dominated by image tokens at `"high"` |
| **Best when** | You need exact coordinates, audit-grade fidelity, or fixed high-volume forms | Novel layouts, charts/diagrams, "understand and extract" in one call |

In practice many production pipelines **combine** them — Document AI for coordinate-anchored character fidelity, a VLM for the semantic "which value means what" — but you only take on two systems once you can name why one isn't enough.

- *Build consequence:* Set `detail`/resolution by whether the answer lives in fine print (default `"high"` for documents), reason about multi-image purely as more token blocks in the message, and justify any drop-down to Document AI by a named knob — bounding boxes, coordinate-grade fidelity, or fixed high-volume forms — never by habit.

**Hands-on ([Part B](#part-b--vision-in-documents-tables-charts--multi-image)):** Take a receipt or invoice image and extract its line-items and total into a **Pydantic-validated** object using schema-enforced output (reuse [Chapter 2, Part C](02-apis-and-integration.md)'s mechanism — don't rebuild it). Then add the verification step from concept 4: assert the line-items sum to the total, and if they don't, re-prompt the model with the named discrepancy and re-validate. Separately, send a **chart image** (a bar or line chart) and extract the underlying series as JSON (`[{"label": ..., "value": ...}, ...]`) — proving the VLM reads charts, not just text. **Docker / no-key variant:** run a local **Docling** (or another open OCR) container, feed it the *same* document, and diff its structured output against the LLM's — observe where Docling preserves exact characters that the LLM rephrased, and where the LLM assembled structure that Docling left as loose boxes.

```python
import base64, os
from pydantic import BaseModel

class LineItem(BaseModel):
    description: str
    qty: int
    unit_price: float
    amount: float

class Invoice(BaseModel):
    line_items: list[LineItem]
    total: float

img_b64 = base64.standard_b64encode(open("invoice.jpg", "rb").read()).decode()

# --- OpenAI: schema-enforced output (high detail for the small print) --------
from openai import OpenAI
client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

def extract(note: str = "") -> Invoice:
    resp = client.beta.chat.completions.parse(
        model="gpt-4o",
        messages=[{"role": "user", "content": [
            {"type": "text", "text": f"Extract the line items and total. {note}"},
            {"type": "image_url", "image_url": {
                "url": f"data:image/jpeg;base64,{img_b64}", "detail": "high"}},
        ]}],
        response_format=Invoice,   # <-- Ch2's schema-enforced mechanism, reused
    )
    return resp.choices[0].message.parsed

inv = extract()

# --- Verification pass: the constraint the document already carries ----------
def verify(inv: Invoice) -> Invoice:
    summed = round(sum(li.amount for li in inv.line_items), 2)
    if abs(summed - inv.total) > 0.01:
        # silent misread surfaced by arithmetic -> re-ask with the gap named
        return extract(note=(f"Your line items summed to {summed} but you "
                             f"reported total {inv.total}. Re-read carefully "
                             f"and correct the misread digit."))
    return inv

inv = verify(inv)
print(inv)
```

---

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
