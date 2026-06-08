# Day 28 — Beyond Text, part 3 — image/video generation, provenance & eval

> [← Day 27](day-27.md) · [All days](README.md) · [Day 29 →](day-29.md)

**Module:** Beyond Text  ·  **Time:** ~2.5–3 hrs

## Where we are

_Continues **Beyond Text**. Earlier days covered Parts A, B, C, D, E; today picks up where they left off._

---

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

---

## Module wrap-up — hands-on, questions & deliverable

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

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
