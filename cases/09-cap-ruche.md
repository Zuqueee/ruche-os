# 9 · Cap Ruche — a browser video editor for ad creative

**A 9:16 editor that runs in the browser on a mid-range laptop, exports faster than real time, and
generates word-level captions — built because the designer was losing hours to a consumer editing
app.**

[← back to index](../README.md)

---

## Context

Paid social for home improvement runs on vertical video, and the agency's designer produced the
creative for every client. The tooling was a consumer editing app on a mid-range gaming laptop, and
the friction was constant: slow renders, a timeline that fought back, captions timed by hand, and no
connection to the agency's own asset library or AI tooling.

The brief was narrow and worth stating because it shaped every decision: **not a general-purpose
editor.** A lean 9:16 editor covering exactly what ad creative needs, fast on the hardware that
existed, with the agency's generative tooling on the same screen instead of in another tab.

## What was built

A browser editor with a TypeScript core and a server component for the operations that cannot run
client-side.

| Layer | What |
|---|---|
| Core | Edit model in integer microseconds, with undo/redo |
| Playback | Frame-accurate player, clock driven by `AudioContext` rather than video time |
| Timeline | Magnetic snapping, multi-track, clip panel (speed, adjustments, filters, transitions) |
| Text | 8 templates, 9 self-hosted fonts plus paid-font upload, 6 animations, 8 shapes, 4 masks with feathering and invert |
| Captions | 4 styles, word-level timing, editable text that redistributes timings |
| Persistence | Autosave, 2 GB local cache, resumable chunked upload with a queue |
| Export | FFmpeg in a web worker |
| Backend | Hono service: chat, generative media, cost quoting, caption transcription |

Measured on the target machine (a laptop GTX 1650): a 30-second 1080×1920 export in **4.7 s**; a
26.9-second timeline exported in **10.6 s**. Faster than real time on hardware that is not a
workstation, which was the whole point.

Test coverage ended at **57 front-end tests plus 21 backend tests**, with an end-to-end suite running
in a real browser.

## Design decision: integer microseconds, not floating-point seconds

The edit model stores every time value as an integer count of microseconds.

Floating-point seconds are the natural choice and they accumulate error. Trim a clip, split it, apply
a speed change, nest the result, and the arithmetic drifts — producing gaps of a fraction of a frame
that are invisible in the preview and visible as a flash in the exported file. Integers make every
operation exact and every comparison reliable, and `0.1 + 0.2` stops being a hazard in the one place
where frame alignment is the product.

## Design decision: the audio clock is the clock

Playback time comes from `AudioContext.currentTime`, not from the video element.

A video element's clock is driven by the browser's own decoding and stalls under load — exactly when
a preview is most likely to be scrubbed. The audio clock is driven by the sound hardware and does not
drift. Taking it as authoritative and synchronising the visual to it keeps audio and video locked
together during playback instead of letting them separate under pressure, which is the failure every
browser-based editor is judged on.

## The part I would show first: the end-to-end test caught three real bugs

The export path is where a browser editor is actually hard, because there is no way to inspect the
result except by inspecting the result. So the end-to-end suite runs in a real browser and does the
whole loop: generate test clips with `MediaRecorder`, apply a filter, a fade, a trim, a speed change
and a transition, export, then **parse the resulting MP4 and assert on its structure**.

Three bugs it found that no unit test would have:

- **The audio mix ran one second longer than the timeline.** Correct in every component, wrong in
  composition.
- **A transition was invisible in a paused preview.** It rendered during playback and not on a
  static frame — so it looked fine while editing and was reviewed as broken.
- **A colour-temperature adjustment had no effect on dark pixels.** Mathematically defensible, and
  not what the adjustment is for.

Each of those produces a *plausible, wrong output file* — the same failure class as the rest of this
portfolio, in a different medium. A test that asserts "export completed without error" catches none
of them. **Asserting on the artifact is the only test that matters when the artifact is the product.**

A fourth bug, found the same way, is the one I would use to argue for this kind of testing: **the
canvas does not load fonts on its own.** Without explicitly preparing the font set before drawing,
text in the exported MP4 silently renders in a fallback face — no error, no warning, just the wrong
typography in a file that has already been delivered to a client. Both the preview and the export now
block on font loading before drawing a frame.

## Captions: the feature with the most design in it

Word-level timing comes from a transcription model that returns per-word timestamps. The backend
extracts the audio with FFmpeg to a small mono 16 kHz file before uploading, because sending a video
to a transcription endpoint is slow and expensive for data it discards.

The design work is in what happens after the words come back:

- **Words group into blocks of at most four**, with breaks preferring natural pauses over about
  0.7 seconds. One word at a time is unreadable; a full sentence does not fit a 9:16 frame.
- **Editing the text redistributes the timings** across the corrected words, rather than discarding
  the timing and asking for a re-transcription. Transcription gets names and jargon wrong constantly,
  so correcting text is the normal path, not the exception.
- **Four caption styles** — karaoke, highlight box on the active word, pop-up, one word at a time —
  applied at the project level rather than per clip, matching how the editing apps designers already
  know behave.

The backend's generative and chat routes are **capability-gated by design**: with no API key
configured, the route returns `503` with a message naming exactly what is missing, and the UI shows
that reason. A missing credential produces an explanation rather than a blank screen or a generic
failure — verified as a behaviour, not assumed.

## Honest status

This is the least finished system in the portfolio, and the reason is worth recording rather than
hiding.

The editor itself works and is tested. What was never completed is the cloud half: the database
migration for cloud project storage was never applied, so autosave falls back to local browser
storage; the backend was never deployed, so the generative and chat routes have no live host; and no
API keys were provisioned, so those routes return their designed `503`.

Each of those blockers was outside my lane — applying migrations and deploying to the production
cluster belonged to the system owner, and the pilot it was being built for was overtaken by the
company's shutdown. The editor runs locally, exports real files, and the three unblocking steps are
documented in order.

What I would claim from it: the edit core, the export pipeline, the caption timing design, and a test
strategy that found four bugs that ship-and-see would have delivered to a client.

---

**Related:** [AI call auditing](02-ai-call-auditing.md) · [Engineering notes §VII (LLMs in a pipeline)](../ENGINEERING-NOTES.md)
