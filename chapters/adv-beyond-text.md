## Beyond Text — Vision, Voice & Generation: shipping multimodal features you can trust

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

**Suggested split:** three sessions. Session 1 = Parts A–B (the mental model + vision-in). Session 2
= Parts C–E (audio-in, audio-out, and the realtime voice latency budget). Session 3 = Parts F–G
(generation + provenance, then multimodal eval & cost).

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

## Part C — Audio-in: speech-to-text (STT) and diarization

**6. Speech-to-text is one API call, but production audio needs two things plain transcription doesn't give you — timestamps, and diarization — and diarization is a genuinely separate capability from transcription.** Turning a recording into text is now one call to a hosted **STT** model — *Speech-To-Text*, also called **ASR** (*Automatic Speech Recognition*); same thing. You hand it a `.wav` or `.mp3`, you get back a string. For a voice note, that's the whole job.

Production audio — support calls, meetings, depositions — needs more, and the "more" is two distinct additions:

- **Timestamps:** *when* each word or segment was spoken, so you can jump to `04:12` in the recording, align captions, or clip a quote.
- **Diarization:** *who* spoke each part — labelling the transcript `Speaker 1: …` / `Speaker 2: …`. The word comes from "diary": a per-speaker log of the conversation.

Here is the load-bearing point: **diarization is a separate capability from transcription, and knowing the words is not knowing the speaker.** Transcription answers *what was said*; diarization answers *who said it*. They are frequently produced by **different models stitched together** — one model transcribes the audio into words, a second clusters the voice's acoustic fingerprint to decide how many speakers there are and which segment belongs to whom — and your STT provider wires them into one response. The native-vs-pipeline choice from [Part A](#part-a--the-one-mental-model-every-modality-is-encode-then-decode) shows up exactly here: a plain transcription endpoint is the "native" simple path; turning on diarization and word timestamps is dropping down for **knobs** the bare transcription doesn't expose.

The analogy makes the separation obvious. Transcription is the **court stenographer**: their entire job is to capture every word, in order, verbatim — they are superb at *what was said* and pay no attention to who's talking. Diarization is **the person sitting in the gallery noting which mouth each sentence came out of**. These are two different people doing two different jobs. A perfect stenographer with no gallery-watcher gives you a flawless wall of text with no idea who said what. That is exactly what a transcription-only call returns.

- *Build consequence:* When a feature needs speaker labels, you are buying *two* capabilities, not one — and you must turn diarization on explicitly and verify it independently. Treating "transcribe with speakers" as a single reliable thing is the audio twin of the [Part B](#part-b--vision-in-documents-tables-charts--multi-image) trap "a vision model is as reliable as OCR." The next concept shows why the two error types must be measured apart.

**7. Transcription accuracy and diarization accuracy fail independently and stack — so you measure them as two separate numbers, with two separate metrics.** This deserves a *don't-confuse-X-with-Y* callout, because it's the single most common way people misjudge an STT feature.

> **Don't confuse transcription accuracy with diarization accuracy.** Transcription accuracy asks *did it get the words right?* and is measured by **WER — Word Error Rate** — the count of word-level mistakes (substitutions + insertions + deletions) divided by the number of words actually spoken. Diarization accuracy asks *did it attribute each word to the right speaker?* — a completely different question. A transcript can ace one and fail the other.

Work WER on one short sentence so the metric is concrete. The reference (truth) is:

```
Reference (7 words):   please send the invoice by Friday afternoon
Model transcript:      please send the invoices by Friday
```

Compare word by word: `invoices` is a **substitution** for `invoice` (1 error), and `afternoon` is a **deletion** (1 error). 2 errors over 7 reference words → **WER = 2 / 7 ≈ 0.286**, i.e. ~29%. That single number is your transcription quality. It says **nothing** about speaker labels.

Now the trap that proves they're independent — a 2-speaker support call where the *words are perfect and the speakers are swapped*:

```
What was actually said:
  Agent:    "Thanks for calling, how can I help?"
  Customer: "My order never arrived."

What the system returned (WER = 0 — every word correct):
  Speaker 1: "My order never arrived."          ← labelled Agent's slot
  Speaker 2: "Thanks for calling, how can I help?"
```

WER is **zero** — transcription is flawless. But diarization is **wrong on 100% of turns** — both labels are swapped. If you summarize this call as "the agent said their order never arrived," you've produced a confidently wrong artifact from a transcript that scored perfectly on the only metric most people check. The two errors don't just coexist; they **stack** — a real transcript can have *both* mangled words *and* misattributed speakers, and a single "accuracy" number hides which one is hurting you.

- *Build consequence:* Your eval for an audio feature reports **two** numbers — a transcription score (WER) and a diarization score (right-speaker rate) — never one blended "accuracy." A high-WER/correct-speaker failure and a zero-WER/swapped-speaker failure need opposite fixes (a better STT model vs better diarization or a known-speaker-count hint), and you can't tell them apart from a single score. Building that two-number eval is the job of [Part G](#part-g--multimodal-evaluation--cost).

**The call, two ways — and the knobs that are actually yours.** Whatever provider, the knobs you turn are the same short list: a **language hint** (tell it the audio is Spanish instead of making it guess — faster and more accurate); **timestamp granularity** (`segment`-level for captions vs `word`-level for precise clipping/alignment); and **diarization on/off plus an expected-speaker-count** (telling it "there are exactly 2 speakers" sharply improves attribution over making it infer the number).

```python
import os
# --- Hosted STT API (OpenAI-style; Deepgram is the same shape) ---------------
from openai import OpenAI
client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

with open("support_call.mp3", "rb") as f:
    tr = client.audio.transcriptions.create(
        model="whisper-1",
        file=f,
        language="en",                       # language hint (your knob)
        response_format="verbose_json",
        timestamp_granularities=["word"],    # word- vs segment-level timestamps
    )
print(tr.text)
for w in tr.words[:5]:
    print(f"{w.start:6.2f}s  {w.word}")
# Note: OpenAI's transcription endpoint does NOT diarize. For speaker labels you
# drop down to a diarization-capable provider (Deepgram diarize=True, expected
# speakers=2) or stitch a separate diarization model on top — concept 6's
# "different models stitched together."
```

```bash
# --- Local Docker, no API key: faster-whisper / whisper.cpp -------------------
# Transcription runs fully offline in a container; diarization is a second stage.
docker run --rm -v "$PWD":/audio \
  ghcr.io/ggerganov/whisper.cpp:main \
  "-m" "/models/ggml-base.en.bin" "-f" "/audio/support_call.wav" \
  "--output-words"        # word-level timestamps; no key, no network
# For speaker labels locally, pair the transcript with a diarization model
# (e.g. a pyannote container) and stitch by timestamp overlap.
```

**Streaming vs batch — the latency fork.** Both calls above are **batch**: you hand over a whole file and wait for the whole result. The other mode is **streaming** transcription — partial text comes back *while the person is still talking*, which is what makes the speaker feel heard in real time. That choice — and the related idea of **endpointing** (detecting when a speaker has *stopped* so the system can take its turn) — is the hinge between a transcription feature and a live voice agent. We don't spend its latency budget here; that is the whole subject of [Part E](#part-e--realtime-voice-agents-the-latency-budget--turn-taking).

**Hands-on ([Part C](#part-c--audio-in-speech-to-text-stt-and-diarization)):** Take a short 2-speaker clip and transcribe it **two ways** — a hosted STT API *and* a local Docker whisper container (`faster-whisper` or `whisper.cpp`, no key). Turn on **word timestamps** and **diarization** (with `expected speakers = 2` where the provider supports it), and produce a speaker-labelled transcript. Then do the eval by eye that concept 7 demands: against the truth, mark **where the words are wrong** (count substitutions/insertions/deletions → compute WER on a few lines by hand) and **separately** mark **where the speaker labels are wrong** (turns attributed to the wrong speaker). Record the two error types as **two numbers**, not one — that two-number habit is exactly what [Part G](#part-g--multimodal-evaluation--cost) formalizes.

## Part D — Audio-out: text-to-speech (TTS)

This is the **decode side** of [Part C](#part-c--audio-in-speech-to-text-stt-and-diarization).
There, an audio encoder turned a waveform into tokens; here a **decoder** runs the booth in reverse —
text tokens in, a waveform out. **TTS** (text-to-speech) is the one-call service that does it: you
hand it a string, it hands you back synthesized speech. Because it's the mirror of Part C, this Part
is deliberately short — most of the call shape is symmetric. There are exactly two things worth
slowing down for: the one genuinely new *build decision* (stream it or render it to a file), and the
one genuinely new *obligation* (a synthetic voice carries consent and provenance weight that a
transcript never did).

**8. Text-to-speech is one API call with three build-relevant knobs — voice, format, and how it's
priced — and the only new decision is whether to stream it.** Everything else about a TTS call you
already know: you authenticate, you send a string, you read bytes back. The knobs you actually turn
in a build are a short list. **Voice** — which synthetic speaker (a named preset like a warm female
narrator, or a cloned voice; more on cloning below). **Format** — the audio container and quality you
get back (`mp3` for a download a human will replay, raw `pcm`/`wav` for something you'll pipe
straight into a phone line or a player). And the **cost/latency model**, which is the one place TTS
*doesn't* look like the text API you're used to: TTS is billed **per character of input text** (the
hosted norm — you pay for what you send to be spoken) or **per second of audio produced**, never in
the input/output *token* split [Chapter 2](02-apis-and-integration.md) taught. Cost a TTS feature by
counting characters or output seconds, not tokens — habitually reaching for the token meter is the
predictable mistake carried over from the text chapters.

The analogy is a **recording studio versus a live announcer**. If you're producing a podcast episode
or pre-rendering an IVR ("press 1 for billing") menu prompt, you book the studio: the whole take is
rendered to a polished file once, offline, and replayed forever — nobody is standing there waiting
for the first word. But if a human is on the other end *right now* waiting to hear an answer, you
want the live announcer who starts speaking the moment the first sentence is ready, while the rest is
still being produced. That is the **streaming TTS vs full-file TTS** decision, and it turns on a
single question you already met for text in [Chapter 2](02-apis-and-integration.md) Part B (*streaming
changes the UX, not the answer*): **is a human waiting to hear the first syllable?**

- **Full-file TTS** — call returns when the entire clip is rendered. Simplest to handle (one blob,
  save it, done). Right for anything generated offline: podcast audio, a batch of IVR prompts, an
  audiobook, a notification sound bed.
- **Streaming TTS** — the service emits audio **chunks** as it synthesizes, so you can start playback
  on the first chunk instead of after the last. The number that matters here is **time-to-first-audio**
  (also called *time-to-first-byte* of audio): how long from your request until the listener hears the
  *first* sound. For anything conversational, that number is the whole game — and it is exactly the
  metric [Part E](#part-e--realtime-voice-agents-the-latency-budget--turn-taking) will put on a clock.

- *Build consequence:* You pick streaming vs full-file by who is waiting, and you pick the audio
  format by what consumes the bytes — a file for offline, raw PCM streamed for a live caller. If a
  human is in the loop listening, stream and measure time-to-first-audio; if the audio is produced
  ahead of time, render to a file and don't pay the complexity of chunk handling. That single choice
  is the bridge into the latency budget in the next Part.

**9. A synthetic voice is generated media — so it carries a consent-and-provenance obligation a
transcript never did.** Part C produced *text from someone's real voice*; Part D produces *a voice
that was never spoken*. That flip is not just technical. The moment you can make a convincing human
voice on demand, two duties attach that simply didn't exist on the encode side. First, **voice
cloning** — synthesizing speech in the likeness of a specific real person from a sample of their
voice — requires that person's **consent**: cloning a voice you don't have permission to use is the
audio equivalent of identity theft, and reputable hosted providers gate cloning behind an explicit
verification step for exactly this reason. Use named *preset* voices freely; treat *cloned* voices as
consent-gated by default. Second, in many contexts a synthetic voice must be **disclosed** as
synthetic — a caller has a reasonable expectation of knowing they're talking to a machine, and a
growing body of regulation (and plain product ethics) treats an undisclosed synthetic voice as
deceptive.

This is also the first place this chapter *produces* the synthetic media that
[Part F](#part-f--image--video-generation--provenance-c2pa) is about. The audio bytes a TTS call hands
you are exactly the artifact that **provenance** — the **C2PA** standard, a tamper-evident "this was
machine-generated, here's by what and when" manifest — is designed to travel with. We won't re-cover
C2PA here; just hold the forward-pointer: *what you generate in this Part is what you'll learn to
sign and disclose in [Part F](#part-f--image--video-generation--provenance-c2pa).*

- *Build consequence:* Before you ship a TTS feature, answer two questions in writing: *is this a
  cloned voice, and if so do I have documented consent?* and *does this context require disclosing the
  voice as synthetic?* Wire preset voices in freely; route any voice-cloning request through a consent
  gate; and assume the generated audio will need a provenance manifest attached downstream
  ([Part F](#part-f--image--video-generation--provenance-c2pa)). These are not legal footnotes — they
  are the new failure mode this modality drags in, the same way Part B's vision model dragged in silent
  mis-reads.

Here is the call, both shapes — a **hosted** voice and a **local Docker** voice with no API key —
each rendered once to a file and once streamed for time-to-first-audio. Secrets come from the
environment, never hard-coded.

```python
import os, time

SENTENCE = "Your order shipped this morning and arrives Thursday."

# --- Hosted TTS (OpenAI; ElevenLabs is the same shape: pick a voice, stream or save) -----
from openai import OpenAI
openai = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

# (a) Full-file: render the whole clip, then save. Right for offline/IVR/podcast.
resp = openai.audio.speech.create(model="tts-1", voice="alloy", input=SENTENCE)
resp.stream_to_file("hosted_full.mp3")

# (b) Streaming: measure time-to-first-audio — the number Part E puts on a clock.
t0 = time.perf_counter()
first_audio_at = None
with openai.audio.speech.with_streaming_response.create(
    model="tts-1", voice="alloy", response_format="pcm", input=SENTENCE,
) as stream:
    for chunk in stream.iter_bytes():
        if first_audio_at is None:
            first_audio_at = time.perf_counter() - t0   # <-- time-to-first-audio
        # ...push chunk to the player/phone line here...
print(f"hosted time-to-first-audio: {first_audio_at*1000:.0f} ms")

# --- Local Docker TTS (Piper), no API key -------------------------------------------------
# Run once:  docker run --rm -p 59125:5000 rhasspy/wyoming-piper --voice en_US-amy-medium
# Piper/Coqui are local, free, lower-latency on a LAN — the native-vs-specialist trade from Part A.
import urllib.request, json
req = urllib.request.Request(
    "http://localhost:59125/api/tts",
    data=json.dumps({"text": SENTENCE}).encode(),
    headers={"Content-Type": "application/json"},
)
t0 = time.perf_counter()
with urllib.request.urlopen(req) as r:
    audio_bytes = r.read()                              # full-file from the local service
print(f"local TTS produced {len(audio_bytes)} bytes in {(time.perf_counter()-t0)*1000:.0f} ms")
open("local_full.wav", "wb").write(audio_bytes)
```

**Hands-on ([Part D](#part-d--audio-out-text-to-speech-tts)):** synthesize the **same one sentence**
four ways — hosted-saved-to-file, hosted-streamed, local-Docker-saved-to-file, local-Docker-streamed
(or full-file if your local engine doesn't stream) — and for each **measure time-to-first-audio**: the
wall-clock ms from request to the first audio byte you could play. Note all four numbers down. The
streamed hosted number is the one you'll plug into the budget in
[Part E](#part-e--realtime-voice-agents-the-latency-budget--turn-taking); the gap between *saved* and
*streamed* is the same UX-not-answer distinction from [Chapter 2](02-apis-and-integration.md), now
measured in your own milliseconds.

---

## Part E — Realtime voice agents: the latency budget & turn-taking

Now compose the whole chapter into one loop and put it on a clock. A voice agent is the Part A
pipeline run end-to-end against a stopwatch: capture the caller's speech →
[Part C](#part-c--audio-in-speech-to-text-stt-and-diarization) STT turns it into text → an LLM (your
[Chapter 5](05-agents.md) loop, or a single call) decides the reply → [Part D](#part-d--audio-out-text-to-speech-tts)
TTS turns that reply into audio → playback. Every stage you've already built; the new thing is that
when a human is *listening*, the round trip is graded by a clock it never had to pass before. This is
the chapter's centerpiece because it's where "I wired the parts together and each one works" still
produces a product that *feels broken* — and the only way to see why is to sum the budget.

**10. A voice agent feels human only if the whole round trip fits a turn-taking latency budget of
roughly a few hundred milliseconds to about one second — and the naive chained pipeline blows it.**
In a text chat, an extra 900ms before the answer appears is invisible; the reader is looking at a
spinner and waiting is normal. In a *voice* conversation it is not invisible at all, because human
conversation has an unwritten timing rule. The analogy is a **phone call**: there is a silence after
which the other person assumes the line dropped. On a phone, a gap longer than about **one second**
between "you stop talking" and "they start talking" reads as *something is wrong* — the caller says
"hello? are you there?" That gap is the **latency budget** (the maximum end-to-end delay you're
allowed before the experience degrades), and for turn-taking in a live voice agent it's roughly
**200ms–1s** from end-of-user-speech to start-of-agent-audio.

Now **derive the problem** — sum the chained pipeline with hand-checkable numbers. A naive voice agent
does the stages strictly in sequence, each finishing before the next begins:

| Stage | What it is | Typical time |
|---|---|---|
| End-of-speech detection (**endpointing**) | deciding the user actually *stopped* talking, not just paused | 200 ms |
| STT ([Part C](#part-c--audio-in-speech-to-text-stt-and-diarization)) | audio → final transcript | 300 ms |
| LLM **time-to-first-token** | request → first token of the reply | 500 ms |
| TTS **time-to-first-audio** ([Part D](#part-d--audio-out-text-to-speech-tts)) | reply text → first audio byte | 300 ms |
| **Total before the user hears anything** | | **1300 ms** |

200 + 300 + 500 + 300 = **1.3 s** — and that's *before the first syllable plays*, with optimistic
per-stage numbers and no network jitter. You are over the ~1s budget on a good day. Crucially, the
problem isn't any single slow part; it's that the naive design makes them **add up in series**. That
sum is the entire reason realtime speech-to-speech APIs exist.

- *Build consequence:* The first thing you do when building a voice agent is **write down this budget
  table with your own measured per-stage numbers** (you already have the STT number from
  [Part C](#part-c--audio-in-speech-to-text-stt-and-diarization)'s hands-on and the TTS time-to-first-audio
  from [Part D](#part-d--audio-out-text-to-speech-tts)'s). If the series total clears ~1s, the
  architecture is wrong and no amount of prompt-tuning fixes it — you reach for the repairs in concept
  11, not for a faster model.

**11. There are exactly three repairs, and they form a ladder: overlap the stages, let the user
interrupt, then collapse the chain entirely.** The budget in concept 10 only blows because the stages
run one-after-another and the agent can't be interrupted. Each rung fixes a specific crack the
previous one leaves.

**Rung 1 — stream every stage and overlap them.** This is the single biggest win and it costs you no
new service. The naive pipeline waits for the *full* transcript before starting the LLM, and the
*full* reply before starting TTS. Don't. Feed the LLM the transcript *as STT emits it* (start
reasoning on a partial transcript), and feed TTS the reply *as the LLM emits tokens* (start speaking
the first sentence while the model is still writing the second) — exactly the streaming you measured
in [Chapter 2](02-apis-and-integration.md) Part B and [Part D](#part-d--audio-out-text-to-speech-tts).
When the stages **overlap** instead of summing in series, your effective latency collapses toward the
*longest single stage* plus a little, not the *sum* — the same 1.3s of work, but the user hears the
first audio far sooner because TTS started before the LLM finished.

**Rung 2 — voice-activity detection and barge-in, so the user can interrupt.** **VAD** (voice-activity
detection) is the cheap signal that detects *when speech is present* in the incoming audio; it's what
drives **endpointing** (concept 10's first stage — deciding the user has *stopped*). Tune it and you
trade latency against false cut-offs: a short endpointing silence (~200ms) feels snappy but clips
people who pause mid-thought; a long one (~800ms) never interrupts them but feels sluggish. **Barge-in**
is the rule that when the user *starts talking while the agent is speaking*, the agent **stops its own
playback immediately** and listens — exactly how a real phone call works, and its absence is the
tell-tale sign of a robotic IVR you can't interrupt. Barge-in is a turn-taking behavior, not a model
setting: your code watches VAD during playback and cancels the current TTS stream the instant the user
speaks.

**Rung 3 — collapse the chain with a native realtime speech-to-speech API.** The top rung removes the
explicit STT and TTS hops altogether. A **realtime speech-to-speech API** (OpenAI Realtime, Gemini
Live) takes audio *in* and emits audio *out* over a single persistent connection (a WebSocket), doing
endpointing, understanding, and speech generation inside one model with overlap built in. This is Part
A's *native-vs-specialist* decision in its sharpest form: instead of chaining four specialists you call
one native multimodal model that was built for the loop. You give up some control (you can't swap in a
specific STT engine or inspect the intermediate transcript as easily) in exchange for the lowest
achievable latency and turn-taking handled for you.

Decide between the bottom of the ladder (a chained, overlapped pipeline) and the top (a native realtime
API) with this table:

| | Chained pipeline (overlapped) | Realtime speech-to-speech API |
|---|---|---|
| Latency | good *if* you overlap; you own the budget | lowest; turn-taking built in |
| Control | full — swap any stage (your STT, your TTS, your LLM/tools) | less — STT/TTS are inside the box |
| Inspectability | you see every intermediate transcript/reply | intermediate transcript is harder to get at |
| Maturity | every part is stable and well-understood | newer, fewer providers, evolving APIs |
| Reach for it when | you need a specific knob (diarization, a particular voice, custom tools mid-turn) | conversational latency is the top priority and a vanilla loop is fine |

Name the anti-pattern plainly: **building a voice agent as a synchronous request-response loop — wait
for the full transcript, then the full reply, then the full audio — will always feel broken**, no
matter how fast each individual call is, because the delays sum in series and the user can't interrupt.
The fix is never a faster model; it's overlap (rung 1), barge-in (rung 2), and — when latency is
paramount — collapsing the chain (rung 3).

- *Build consequence:* Start at rung 1 — stream and overlap every stage — because it's free and it's
  the biggest win; add rung 2 (VAD-driven endpointing + barge-in) the moment real users talk to it,
  tuning the endpointing silence against your own false-cutoff rate; and step to rung 3 (a native
  realtime API) only when conversational latency is the headline requirement and you can afford to give
  up the inspectability and stage-level control of the chained pipeline. As with the agent ladder in
  [Chapter 5](05-agents.md), each rung up buys latency at the cost of control — so climb only as far as
  the requirement forces you.

**Hands-on ([Part E](#part-e--realtime-voice-agents-the-latency-budget--turn-taking)):** instrument the
[Part C](#part-c--audio-in-speech-to-text-stt-and-diarization)+[Part D](#part-d--audio-out-text-to-speech-tts)
chained pipeline end-to-end for **one turn** — capture → STT → LLM → TTS → first audio — and log a
per-stage timestamp so you can fill in the concept-10 budget table with **your real measured numbers**.
Read off the total time-to-first-audio. Then make **one** improvement and measure the drop: either
**overlap the stages** (start the LLM on the partial transcript and start TTS on the first LLM tokens —
rung 1), *or* wire a **hosted realtime speech-to-speech API** for the same turn (rung 3). Record both
budget tables side by side — naive-series vs overlapped/collapsed — and confirm the second one fits the
~1s turn-taking budget. That before/after number is the proof the architecture, not the model, was the
fix.

## Part F — Image & video generation + provenance (C2PA)

*Deepens [Part A](#part-a--the-one-mental-model-every-modality-is-encode-then-decode)'s decode
side and [Chapter 1](01-foundations.md)'s sampling die; does NOT re-cover the diffusion math
that turns noise into pixels — that lives **inside** the booth and you never train it. Here you
**call** a generator and turn its knobs, the same way [Chapter 2](02-apis-and-integration.md)
taught you to call a text model.*

So far every Part has produced **text** out of the booth — extracted fields, a transcript, a
spoken waveform. Generation is the decode side where the **output is the artifact itself**: an
image or a video file. Everything you learned about the encode side still holds — you send tokens
in (your prompt, optionally a reference image) and the booth decodes them — but one thing inverts
that matters for the rest of this chapter. With text out, you could check the answer against a
known string. **You cannot string-match an image.** There is no "correct" pixel array to compare
against, so the eval problem turns inside-out the moment the output stops being text. We name the
new failure mode here and hand the *measurement* of it to [Part G](#part-g--multimodal-evaluation--cost)
— for now, just hold the thought: *generation is the one modality where you ship the artifact
before you know how to grade it.*

**12. Generating and editing an image is an API call with its own knobs — prompt, size, edits,
and a seed — priced per image, not per token.** Up to now "more output" meant "more tokens on the
bill." Generation breaks that habit: you ask for *an image* and you are billed *per image* (or per
megapixel / per quality tier), regardless of how long your prompt was. The call shape is familiar —
a prompt in, a file out — but the knobs are new, so let's name them before any code. **Image
generation** is producing a brand-new image from a text prompt. **Image editing** (also called
**image-to-image**) starts from an image you supply and changes it. **Inpainting** is editing only
*part* of an image: you supply a **mask** — a black-and-white stencil that says "repaint the white
region, leave the black region untouched" — so you can swap one object without disturbing the rest.
A **negative prompt** is the inverse of your prompt: a short list of what you *don't* want ("no
text, no watermark, no extra fingers"), steering the generator away from those rather than toward
something. And the **seed** is the single integer that picks which random draw the generator
starts from.

The analogy for the seed is one you already own. [Chapter 1](01-foundations.md) showed that a model
picks each token by rolling a **loaded die** — a weighted random draw. A generator rolls that same
kind of die, just many times, to decide the random starting noise it sculpts into pixels. The
**seed is the die's starting position.** Fix the seed and you re-roll the *exact same* image every
time; change the seed and you get a different image from the identical prompt. That is what makes
generation debuggable: hold the seed constant, change *one* knob (the prompt wording, the mask, the
size), and any difference in the output is attributable to the knob you moved — not to the dice.
Vary the seed deliberately when you want options to choose from; freeze it when you want a
controlled A/B.

Here is the call, hosted side-by-side and then the no-key Docker variant. Secrets come from the
environment, never hard-coded.

```python
import os, base64

# --- OpenAI (gpt-image-1): generate, then inpaint with a mask + fixed seed ---
from openai import OpenAI
openai = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

# 1) Generate from a prompt. Priced PER IMAGE at a given size/quality — not per token.
gen = openai.images.generate(
    model="gpt-image-1",
    prompt="a ceramic coffee mug on a wooden desk, soft morning light",
    size="1024x1024",
    n=1,
)
open("mug.png", "wb").write(base64.b64decode(gen.data[0].b64_json))

# 2) Inpaint: keep the mug, repaint only the masked region (the desk).
#    mask.png is transparent where you want change, opaque where you want it kept.
edit = openai.images.edit(
    model="gpt-image-1",
    image=open("mug.png", "rb"),
    mask=open("mask.png", "rb"),      # the stencil: "repaint here, leave the rest"
    prompt="the same mug on a white marble countertop",
    size="1024x1024",
)
open("mug_edited.png", "wb").write(base64.b64decode(edit.data[0].b64_json))
```

```python
# --- Hosted-with-an-explicit-seed (Stability / many hosted diffusion APIs) ----
# The pattern that matters here is the SEED, which gpt-image-1 doesn't expose as a knob.
import os, requests
r = requests.post(
    "https://api.stability.ai/v2beta/stable-image/generate/core",
    headers={"authorization": f"Bearer {os.environ['STABILITY_API_KEY']}",
             "accept": "image/*"},
    files={"none": ""},
    data={
        "prompt": "a ceramic coffee mug on a wooden desk, soft morning light",
        "negative_prompt": "text, watermark, blurry",   # steer AWAY from these
        "seed": 42,            # fix the die -> re-roll the identical image
        "aspect_ratio": "1:1",
        "output_format": "png",
    },
    timeout=120,
)
r.raise_for_status()
open("mug_seeded.png", "wb").write(r.content)
```

```bash
# --- No-key local variant: Automatic1111 / ComfyUI in Docker -------------------
# Run a generator on your own GPU; same knobs (prompt, negative_prompt, seed, size)
# arrive as a JSON body to a local HTTP endpoint — no provider, no API key.
docker run --gpus all -p 7860:7860 ghcr.io/abetlen/stable-diffusion-webui:latest
# then POST to http://localhost:7860/sdapi/v1/txt2img with
#   {"prompt": "...", "negative_prompt": "text, watermark", "seed": 42, "steps": 30}
```

- *Build consequence:* You **budget generation per image, not per token**, and you make a feature
  reproducible by pinning the seed in code (logging it next to the prompt) so a "regenerate" button
  or an eval run can reproduce the exact frame later. When a stakeholder says "make it like that one
  but with a blue mug," the answer is *same seed, edit the prompt* — not *roll again and hope.* If
  the generator you picked doesn't expose a seed, write that down: you've traded reproducibility for
  whatever else it gave you, and that's a real engineering cost, not a footnote.

**13. Video generation is the same call shape with a far bigger latency and cost envelope — so
it's an async job you submit and poll, not a request you wait on.** A text model answers in
seconds; an image lands in seconds-to-tens-of-seconds. A few seconds of video is a different order
of magnitude — it can take **minutes** to render and cost **dollars per clip**, not fractions of a
cent. A normal synchronous HTTP request would time out long before the clip is done. So **video
generation** is structured as an **async job**: you `POST` a request, get back a **job id**
immediately, and then **poll** that id (or receive a webhook) until the status flips to `completed`
and a download URL appears. This is not a new pattern — it's exactly the
[Chapter 2](02-apis-and-integration.md) discipline of timeouts, retries, and not letting a request
hang forever, applied to a job that genuinely takes minutes.

The analogy: an image call is **ordering at a counter** — you wait at the register and walk away
with it. A video call is **dropping film off to be developed** — you get a ticket (the job id),
leave, and come back to check whether it's ready. You design the *waiting* deliberately: a bounded
poll loop with a sane interval and a hard ceiling, so a stuck job fails your feature gracefully
instead of wedging a worker forever.

```python
import os, time, requests

API = "https://api.example-video.com/v1"
H = {"authorization": f"Bearer {os.environ['VIDEO_API_KEY']}"}

# 1) Submit the job — returns immediately with an id, NOT the video.
job = requests.post(f"{API}/generate",
    json={"prompt": "a paper boat drifting down a rain-soaked gutter, cinematic",
          "duration_s": 5, "resolution": "720p", "seed": 42},
    headers=H, timeout=30).json()
job_id = job["id"]

# 2) Poll with a bounded loop: a sane interval AND a hard ceiling (Ch2 timeout habit).
start = time.monotonic()
DEADLINE_S, POLL_S = 600, 10        # give up after 10 min; check every 10 s
url = None
while time.monotonic() - start < DEADLINE_S:
    st = requests.get(f"{API}/jobs/{job_id}", headers=H, timeout=30).json()
    if st["status"] == "completed":
        url = st["output_url"]; break
    if st["status"] == "failed":
        raise RuntimeError(f"video job failed: {st.get('error')}")
    time.sleep(POLL_S)
else:
    raise TimeoutError(f"video job {job_id} not done after {DEADLINE_S}s")

elapsed = time.monotonic() - start
print(f"done in {elapsed:.0f}s -> {url}")   # LOG wall-clock time; it IS the cost story
```

A blunt reality-check on the envelope: a single 5-second 720p clip on a current hosted generator
commonly runs **on the order of $0.50–$2.00 and 1–5 minutes of wall-clock**. Put that next to a
text call at a fraction of a cent in under a second and the design implication is obvious — **video
generation is never on the synchronous path of a user request.** You queue it, you show progress,
you cache aggressively, and you treat each clip as a line item a human probably approved.

- *Build consequence:* You architect video generation as a **background job with a queue, a poll
  loop (or webhook), a timeout, and a cost ceiling per clip** — never as an inline call a user
  waits on. The wall-clock and dollar numbers you log here are exactly the inputs
  [Part G](#part-g--multimodal-evaluation--cost)'s cost table consumes.

**14. Anything you generate is synthetic media that increasingly must carry provenance — C2PA
content credentials — and attaching it is a build obligation, not an afterthought.** When your
feature emits an image or a video, it is putting **synthetic media** into the world, and platforms,
publishers, and regulators increasingly expect that media to declare *where it came from*.
**Provenance** is that declaration: a verifiable record of how a file was created and edited.
**C2PA** (the Coalition for Content Provenance and Authenticity) is the open standard for it, and
the user-facing name is **Content Credentials**. Concretely it attaches a **provenance manifest** —
a small, cryptographically **signed** record — to the file, stating things like "created by this
model, on this date, edited thus." Because it's signed, it's **tamper-evident**: change the pixels
without re-signing and verification fails loudly.

The analogy is a **nutrition label that can't be peeled off and forged.** It travels with the
product and a checkout scanner can confirm it's genuine. Two operations matter, and they sit at
opposite ends of your pipeline. You **attach** a manifest **on generation** — the moment your
feature produces the artifact, you stamp it. You **verify** a manifest **on ingest** — whenever
media arrives from outside (a user upload, a third-party asset), you check whether it carries
credentials and whether they validate. Build both in from day one; retrofitting provenance after a
file has been copied, re-encoded, and screenshotted around the internet is mostly impossible.

```bash
# --- Open c2patool in Docker: no API key, inspect + attach + verify -----------
# Inspect any file for existing Content Credentials (the "ingest / verify" side).
docker run --rm -v "$PWD":/work contentauth/c2patool:latest \
  /work/mug.png

# Attach a signed manifest on generation (the "attach" side).
# manifest.json declares the assertions: AI-generated, by which tool, actions taken.
docker run --rm -v "$PWD":/work contentauth/c2patool:latest \
  /work/mug.png -m /work/manifest.json -o /work/mug_signed.png

# Verify the result — tamper-evident: edit pixels after signing and this fails.
docker run --rm -v "$PWD":/work contentauth/c2patool:latest \
  /work/mug_signed.png
```

> **Don't confuse C2PA provenance with an invisible watermark — they live in different places and
> they stack.** A **C2PA manifest** is a signed record *attached to the file's metadata* — a
> document travelling alongside the pixels, cryptographically bound to them. An **invisible
> watermark** is a faint signal *embedded into the pixels themselves*, designed to survive
> screenshotting, cropping, and re-encoding — the operations that strip metadata. They differ on
> **where the signal lives** (beside the pixels vs inside them) and they have opposite weaknesses:
> strip the metadata and the manifest is gone but the watermark remains; re-encode hard enough and
> the watermark degrades but a re-signed manifest stays intact. Serious provenance uses **both** —
> the manifest for a rich, verifiable history, the watermark as a survives-the-screenshot backstop.
> Reaching for one and calling provenance "done" is the predictable beginner mistake.

- *Build consequence:* You add **two seams** to any generation feature: a *signing* step that
  attaches a C2PA manifest the instant you produce an artifact, and a *verification* step that
  checks credentials on every inbound asset and routes failures (unsigned, or signature-invalid) to
  your moderation/trust logic. Provenance becomes a tested component with the open `c2patool` in
  CI — not a compliance box someone bolts on after launch.

**Hands-on ([Part F](#part-f--image--video-generation--provenance-c2pa)):** generate one image from
a prompt; save the file. Then **edit it via inpainting**: draw a simple mask over one region, keep
the **seed fixed**, and regenerate — confirm only the masked region changed while the rest stayed
put (that's controlled change, the seed proving it). Now inspect the file for Content Credentials
with the open `c2patool` in Docker; if none are present, **attach** a manifest declaring it
AI-generated and **verify** it, then flip one pixel and re-verify to watch validation fail
(tamper-evidence, demonstrated). *Stretch:* kick off **one async video-generation job**, poll it to
completion, and **log the wall-clock time and the clip's cost** — you'll feed both numbers straight
into the Part G cost table.

---

## Part G — Multimodal evaluation & cost

*Deepens [Chapter 7](07-evaluation.md)'s exact-match/normalize ladder and
[Chapter 2](02-apis-and-integration.md)'s cost-math habit; does NOT re-derive the confusion
matrix, P/R/F1, or LLM-as-judge from scratch — it **re-aims** them at non-text outputs. It also
does NOT re-cover CLIP training; we name CLIP-style similarity as a tool you call, never one you
train.*

[Chapter 7](07-evaluation.md) built a ladder for grading text: start at **exact match**, repair it
with **normalize**, climb to classification metrics and LLM-as-judge as the output got fuzzier.
Every rung on that ladder quietly assumed one thing — **the output is a string you can compare.**
That assumption is exactly what this whole chapter has been dismantling. So here is the crack we
derive from: *Chapter 7's ladder told you how to exact-match a text answer — but how do you
exact-match a **transcript**, an **image**, or a **spoken reply**?* You can't. The reassuring news,
which this Part keeps returning to, is that **most multimodal eval reduces back to a text comparison
once you extract the right thing** — and where it can't, you fall back to the same LLM-as-judge and
human-judgement tools Chapter 7 already gave you. Nothing here is a new evaluation philosophy; it's
the same discipline, applied per modality.

**15. You evaluate each modality by reducing its output to something you already know how to score
— and generation is the one that resists, which is the honest caveat.** The trick for every
modality is the same: find the representation where Chapter 7's tools apply again. Walk them in
order, easiest reduction first.

- **Vision-extraction → field-level accuracy against a labelled set.** When you pulled structured
  data out of an invoice in [Part B](#part-b--vision-in-documents-tables-charts--multi-image), the
  output *is already text* — a JSON object of fields. So evaluation is just Chapter 7's exact/
  normalize comparison done **per field**: `total`, `invoice_number`, each line item, scored
  against a human-labelled answer. This is the **reassuring bridge** — vision eval is text eval
  wearing a hat. Report accuracy per field (not one blended number), because a model that nails
  `vendor` but fluffs `total` is a very different risk from the reverse.
- **STT → Word Error Rate (WER).** A transcript is text, but two correct transcripts rarely match
  character-for-character, so exact match is the wrong rung. **WER** is the standard metric: count
  the **S**ubstitutions, **D**eletions, and **I**nsertions needed to turn the model's transcript
  into the reference, divided by the number of words in the reference. `WER = (S + D + I) / N`.
  Worked on one sentence — reference (N = 7 words): *"please send the invoice by next friday"*;
  hypothesis: *"please send the invoices by friday"*. Align them: "invoice"→"invoices" is **1
  substitution**, the dropped "next" is **1 deletion**, no insertions. So
  `WER = (1 + 1 + 0) / 7 = 2/7 ≈ 0.286`, i.e. **28.6%**. Lower is better; 0 is a perfect match.
- **Diarization → speaker-attribution error.** Diarization answers *who spoke when*, so its error is
  separate from WER: even with a perfect transcript, the labels can be wrong. Measure the **fraction
  of speech time attributed to the wrong speaker** (the family of metrics around **DER**, diarization
  error rate). A transcript can score WER 0% and still be useless if every "Agent:" and "Customer:"
  is swapped — which is why [Part C](#part-c--audio-in-speech-to-text-stt-and-diarization) insisted
  they fail independently.
- **TTS / voice → human or MOS-style judgement, or an LLM-judge on the transcript-of-the-output.**
  There's no reference waveform to diff. The classic metric is **MOS** (Mean Opinion Score) — humans
  rate naturalness 1–5 and you average. When humans don't scale, transcribe the synthesized audio
  back to text (STT) and run an LLM-as-judge on *that* transcript for correctness and tone — you've
  reduced an audio-quality question to a text question you already know how to grade.
- **Image generation → reference-based similarity or LLM-as-judge on a rubric — the least-solved
  rung.** With no "correct" image, you have two honest options. **CLIP-style similarity** scores how
  well an image matches a text description (or a reference image) using a pretrained vision-text
  model — named, not derived; you *call* it, you never train CLIP. Or run a **multimodal
  LLM-as-judge**: hand a capable vision model your image plus a written rubric ("is there exactly one
  mug? is the lighting soft? any text artifacts?") and have it score each criterion. Both are
  proxies, and you should say so out loud: **generation eval is the least solved problem in this
  list**, which is precisely why [Part F](#part-f--image--video-generation--provenance-c2pa) warned
  you that you ship the artifact before you can fully grade it.

The analogy across all five: every modality hands you an output in a foreign currency, and
evaluation is **finding the exchange rate back to text** — sometimes it's a clean 1:1 (extracted
JSON), sometimes a lossy conversion (WER, MOS), and for generated images the rate is genuinely
fuzzy, so you quote it with a confidence caveat.

- *Build consequence:* You **pick the metric from the output's shape, then build the smallest
  labelled set that exercises it** — field-accuracy for extraction, WER for STT, DER for
  diarization, MOS-or-LLM-judge for voice, similarity-or-rubric for generation. The metric tells you
  which rung of Chapter 7's ladder you're really standing on, and naming it stops you from
  pretending a fuzzy generation score is as solid as an exact-match one.

**16. You can't cost a multimodal feature in tokens by habit — each modality is billed in its own
unit, so you do the arithmetic once, up front, the way Chapter 2 taught.** [Chapter 2](02-apis-and-integration.md)
drilled one reflex: *what does one call cost, and how many calls a day?* — answered in tokens. The
modalities break the unit. Vision is priced by **image detail / resolution** (an image is hundreds
to ~1.5k tokens, scaling with size — the gap you measured back in
[Part A](#part-a--the-one-mental-model-every-modality-is-encode-then-decode)). Audio is priced **per
minute** of input or output. Generation is priced **per image** or **per video job**. Costing all of
that in "tokens" by reflex is the habit to break; the fix is one unified table, then a worked
estimate.

The analogy: it's a **utility bill with several meters** — electricity by the kWh, water by the
litre, gas by the therm. You don't bill water in kilowatt-hours, and you don't bill audio in tokens.
Read each meter in its own unit, then sum the money.

| Modality | Billing meter | Rough unit cost (order-of-magnitude) | What drives it |
|---|---|---|---|
| Vision-in (extract) | per image (by detail/resolution) | ~$0.001–$0.01 / image | resolution / detail tier |
| Text reasoning | per 1M tokens (in + out) | ~$1–$15 / MTok | prompt + output length |
| STT (audio-in) | per minute of audio | ~$0.006–$0.02 / min | call duration |
| TTS (audio-out) | per minute (or per char) | ~$0.015–$0.10 / min | reply length |
| Image generation | per image (by size/quality) | ~$0.01–$0.08 / image | size / quality tier |
| Video generation | per clip / per job | ~$0.50–$2.00 / 5-s clip | duration / resolution |

*(Order-of-magnitude rates; always re-check the provider's current price page before you commit a
number to a plan — these move.)*

Now the worked estimate, for a realistic mixed feature: **process 10,000 documents + 1,000 support
calls + generate 500 images** in a month. Pick mid-range rates from the table and do it meter by
meter, Chapter 2 style:

- **Vision — 10,000 documents.** One image each at ~$0.004 → `10,000 × $0.004 = $40`. Plus the
  reasoning to turn each into JSON: say ~1,500 in + ~300 out tokens at ~$3/$15 per MTok →
  `(1,500 × $3 + 300 × $15) / 1e6 = $0.0090` per doc → `10,000 × $0.0090 = $90`. **Vision subtotal ≈ $130.**
- **Audio — 1,000 support calls** averaging **5 minutes** each = 5,000 minutes. STT at ~$0.01/min →
  `5,000 × $0.01 = $50`. If you also summarize each call (~2,000 tokens in / 200 out at $3/$15) →
  `(2,000 × $3 + 200 × $15) / 1e6 = $0.0090` × 1,000 = $9. **Audio subtotal ≈ $59.**
- **Generation — 500 images** at ~$0.04 each → `500 × $0.04 = $20`. **Generation subtotal = $20.**
- **Monthly total ≈ $130 + $59 + $20 = $209.**

The point isn't the exact $209 — it's that you **read three different meters and summed dollars**,
and you can now see at a glance that *vision reasoning tokens*, not the images, quietly dominate the
document line. That's the kind of thing a tokens-only habit hides until the bill arrives.

> **Anti-pattern — "eyeball three outputs and ship."** This is the same trap
> [Chapter 7](07-evaluation.md) (and Chapter 3) named for text: looking at a handful of outputs,
> deciding they "look good," and shipping with no number. It is **more** tempting here because the
> outputs are *pretty* — a crisp transcript, a slick image — and pretty reads as correct. It isn't.
> A transcript can look fluent and carry a 28% WER; an image can look gorgeous and ignore half your
> rubric. Three eyeballed samples is not an eval set; build the small labelled set and report the
> metric, exactly as you would for text.

- *Build consequence:* Before you commit to a multimodal feature you produce a **one-page cost table
  in the correct units per modality** and a **per-modality eval number**, not a tokens-only guess and
  a vibe. The cost arithmetic decides whether the feature is viable; the eval number decides whether
  it's correct — and neither is the thing you can fake by looking at three nice-looking outputs.

**Hands-on ([Part G](#part-g--multimodal-evaluation--cost)):** build a **tiny labelled eval set
(5–10 items)** for ONE modality you used earlier — either **field-accuracy** for the
[Part B](#part-b--vision-in-documents-tables-charts--multi-image) invoices (hand-label `total`,
`invoice_number`, and the line items for each) or **WER** for the
[Part C](#part-c--audio-in-speech-to-text-stt-and-diarization) transcripts (hand-type the reference
for each clip). Score your existing pipeline against it. Then **change exactly one knob** — bump the
image **detail** level, or swap the **STT model** for a larger one — and rerun, showing the metric
move (and noting whether the cost moved with it). Finally, produce a **one-page cost table** for a
realistic mixed-modality feature in the correct billing units (images per-image, audio per-minute,
generation per-image/job), reusing the worked-estimate shape from concept 16.

**Resources**

- Vision: [Anthropic vision](https://docs.anthropic.com/en/docs/build-with-claude/vision) and [OpenAI vision](https://platform.openai.com/docs/guides/vision); document AI — [AWS Textract](https://docs.aws.amazon.com/textract/), [Azure Document Intelligence](https://learn.microsoft.com/azure/ai-services/document-intelligence/), open-source [Docling](https://github.com/docling-project/docling) (Docker).
- Audio-in: [OpenAI transcription](https://platform.openai.com/docs/guides/speech-to-text), [Deepgram](https://developers.deepgram.com/); self-host [faster-whisper](https://github.com/SYSTRAN/faster-whisper) / [whisper.cpp](https://github.com/ggml-org/whisper.cpp) (Docker).
- Audio-out & realtime: [OpenAI TTS](https://platform.openai.com/docs/guides/text-to-speech), [ElevenLabs](https://elevenlabs.io/docs); self-host [Piper](https://github.com/rhasspy/piper) / [Coqui TTS](https://github.com/coqui-ai/TTS); [OpenAI Realtime API](https://platform.openai.com/docs/guides/realtime) and [Gemini Live](https://ai.google.dev/gemini-api/docs/live).
- Generation & provenance: [OpenAI images](https://platform.openai.com/docs/guides/images); [C2PA / Content Credentials](https://c2pa.org/) and the open [c2patool](https://github.com/contentauth/c2patool) (Docker).
- Eval: WER & MOS background, and reuse [Chapter 7](07-evaluation.md)'s judge for the reference-free cases.

**Questions**

*Check your understanding*

1. What does schema-enforced structured output fix in a vision extraction, and what does it NOT fix?
2. Define WER on this pair — reference "open the front door now" (5 words), transcript "open the front door".
3. How is a hosted TTS call typically priced, and why is that a gotcha for someone coming from the text API?
4. What single question decides streaming TTS vs full-file TTS?
5. In Part F, what does fixing the seed do when you generate or edit an image, and which Chapter 1 idea is it the same as?
6. Why is video generation structured as an async job you poll rather than a synchronous call?

*Apply it*

7. An invoice extraction returns line-items summing to 88.00 but a total field of 100.00. What do you do, and why is this better than asking a second LLM to "double-check"?
8. You need to extract values AND their pixel coordinates on a scanned form for an audit overlay. Native VLM, Document AI, or both — and why?
9. Sum the naive voice-agent budget for endpointing 200ms + STT 300ms + LLM time-to-first-token 500ms + TTS time-to-first-audio 300ms. Is it within a ~1s turn-taking budget?
10. A voice agent that won't let the user interrupt it mid-sentence is missing which turn-taking behavior, and how is it built?
11. Compute the WER for reference "please send the invoice by next friday" (7 words) vs hypothesis "please send the invoices by friday".
12. A C2PA manifest and an invisible watermark both signal "AI-generated" — where does each live, and why use both?
13. Estimate the monthly cost of 10k documents + 1k five-minute calls + 500 generated images using mid-range rates.

*Stretch*

14. A support-call feature scores WER 0.02 (excellent) yet users complain summaries blame the wrong person. What's broken, and why didn't your metric catch it?
15. Why is choosing image `detail:"low"` to save tokens on an invoice a self-inflicted wound, and when IS "low" the right call?
16. Rung 1 says overlapping the stages collapses latency without a faster model. Toward what does the effective latency move, and why?
17. Your agent needs speaker diarization and a specific cloned brand voice mid-conversation. Chained pipeline or native realtime API, and why?
18. Your image generator hits a high CLIP-style similarity score against the prompt, so a teammate wants to ship with no other eval. What's your objection?

**Answer key** — *peek only after attempting.*

1. It fixes lost *structure* — guarantees the right keys/types come back (e.g. `total` is a number in the right field), reusing Ch2's mechanism. It does NOT fix *correctness*: the model can still silently mis-read a digit, so the value can be well-formed and wrong. That gap is why you add a verification pass.
2. One deletion ("now"), 0 substitutions/insertions → 1 error / 5 reference words = WER 0.20 (20%).
3. Per character of input text (or per second of audio out), not in input/output tokens. The gotcha: you habitually cost it with the token meter from Ch2 and get it wrong — count characters or output seconds instead.
4. "Is a human waiting to hear the first syllable?" If yes, stream and measure time-to-first-audio; if the audio is produced offline (podcast, IVR prompt), render to a file.
5. It pins the generator's random starting point so the identical prompt re-rolls the exact same image; it's the same loaded-die / weighted-random-draw idea from Chapter 1, with the seed setting the die's starting position. Hold it constant to attribute any output change to the one knob you moved.
6. A few seconds of video can take minutes and cost dollars per clip — far past any sane HTTP timeout — so you POST the job, get an id back immediately, and poll (or take a webhook) with a bounded loop and a hard deadline until it's completed, reusing Chapter 2's timeout/retry discipline.
7. The 12.00 gap means a silent misread; re-prompt with the discrepancy named ("items sum to 88.00 but total is 100.00, re-read") and re-validate. It's better because the document carries its own arithmetic constraint — the check is free, deterministic, and catches exactly the error class vision introduces, with no extra model call on the happy path.
8. Drop down to Document AI (Textract / Document Intelligence / Docling): the named knob the VLM can't reliably give is bounding boxes (coordinates). Optionally combine — Document AI for coordinate-anchored fidelity, a VLM for the semantic "which value means what" — but only because you can name why one alone isn't enough.
9. 200+300+500+300 = 1300ms = 1.3s before the user hears anything — over the ~1s budget. The stages run in series, so the delays add up; that sum is why realtime speech-to-speech APIs exist.
10. Barge-in: when VAD detects the user starting to speak during playback, the agent cancels its own TTS stream immediately and listens. It's code watching VAD during playback, not a model setting.
11. "invoice"→"invoices" = 1 substitution, dropped "next" = 1 deletion, 0 insertions. WER = (S+D+I)/N = (1+1+0)/7 = 2/7 ≈ 0.286 = 28.6%. Lower is better; 0 is perfect.
12. The C2PA manifest is a signed record in the file's metadata (a document beside the pixels, cryptographically bound); the watermark is a faint signal embedded inside the pixels. Strip metadata and the manifest is gone but the watermark survives; re-encode hard and the watermark degrades but a re-signed manifest holds. They stack, so serious provenance uses both.
13. Vision: 10k×$0.004 image + 10k×~$0.009 reasoning ≈ $130. Audio: 5,000 min×$0.01 STT + ~$9 summaries ≈ $59. Generation: 500×$0.04 = $20. Total ≈ $209 — and the vision *reasoning tokens*, not the images, dominate that line.
14. Transcription is fine; diarization is failing (speakers swapped/misattributed). A single WER number measures words only and is blind to speaker attribution — the two fail independently and stack. Fix: report a separate diarization (right-speaker) score, and pass an expected-speaker-count hint to improve attribution.
15. "low" encodes a small, cheap version that drops fine print — exactly the small digits/totals you're extracting — so it manufactures the silent misreads you then have to verify. "low" is right only when the answer doesn't live in fine print (e.g. "is there a cat in this photo?", coarse scene/layout questions). Documents default to "high".
16. Toward the *longest single stage* plus a little, instead of the *sum* of all stages — because streaming lets TTS start before the LLM finishes and the LLM start before STT finishes, so the work happens concurrently rather than in series. Same total work, first audio far sooner.
17. Chained pipeline: you need specific knobs (diarization from Part C, a particular voice from Part D, custom tools mid-turn) that a native realtime speech-to-speech API keeps inside the box. You accept higher latency management in exchange for stage-level control and inspectability.
18. CLIP-style similarity is a proxy for prompt-match, not a verdict on correctness — generation eval is the least-solved rung, and a pretty image can still flunk specific rubric criteria (one mug not three, no text artifacts). Pair it with a multimodal LLM-as-judge on a written rubric and/or human review; don't let one proxy plus three eyeballed samples stand in for an eval set.

**Deliverable:** one **multimodal feature** built end to end with measurement, not vibes — pick a modality from this chapter (a vision-extraction pipeline over real documents, a transcribe-and-diarize pipeline over a 2-speaker clip, or a realtime voice turn), and ship it with: the right knob choices justified (native-vs-specialist, image detail, diarization, streaming-vs-file), a **per-modality metric** computed on a small labelled set (extraction field-accuracy / WER / a generation rubric), and a **one-page cost estimate** in the correct billing units (per-image/megapixel, per-minute, per-image/job). If you generate media, attach and verify a C2PA manifest.

**Daily update:** one line — what modality you shipped and any blocker (e.g. "invoice extractor live: image + Pydantic schema + a sum-check verifier; field-accuracy 0.92 over 10 docs, up from 0.86 at high-detail; ~\$0.004/doc → ~\$40 for 10k — open question: scanned pages still drop the odd digit").

**Time:** three sessions (the largest chapter). Session 1: Parts A–B (the encode→decode mental model + vision-in). Session 2: Parts C–E (audio-in, audio-out, and the realtime voice latency budget). Session 3: Parts F–G (image/video generation + C2PA provenance, then multimodal eval & cost) plus the deliverable.
