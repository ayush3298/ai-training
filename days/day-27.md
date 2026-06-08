# Day 27 — Beyond Text, part 2 — audio-in, audio-out & the voice latency budget

> [← Day 26](day-26.md) · [All days](README.md) · [Day 28 →](day-28.md)

**Module:** Beyond Text  ·  **Time:** ~3 hrs

## Where we are

_Continues **Beyond Text**. Earlier days covered Parts A, B; today picks up where they left off._

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

## Daily update
Send your one-liner: what you covered, what clicked, what's still fuzzy.
