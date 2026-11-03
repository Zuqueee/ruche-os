# 2 · AI call auditing

**Every recorded sales call transcribed, scored against a weighted rubric, and surfaced to the person
who has to coach the setter — with the arithmetic done in code and the judgment done by a model.**

[← back to index](../README.md)

---

## Context

The call centre dialled inbound leads and tried to book an in-home estimate. The agency was paid per
booked appointment, so setter performance was the single highest-leverage variable in the business —
and it was being managed by a manager listening to a handful of calls a week out of hundreds.

Sampling at that rate cannot answer the questions that matter. Which setter is slipping this week?
Is the booking rate down because of lead quality or because the pitch changed? Did the person who
booked eleven appointments do it by being good or by promising things that fall apart at the visit?

## What was built

```
dialer webhook ──► events (type=dial)
                        │  recording present?
                        ▼
              Deepgram nova-3 (diarized)
                        │
                        ▼
          LLM, strict-schema function calling
          ~4,500-token rubric system prompt
                        │
                        ▼
          score recomputed deterministically in SQL
                        │
                        ▼
      fn_upsert_call_audit ──► call_audits ──► reviewer UI
                                    │              + bilingual notes (EN/PT)
                                    └──► triggers booking-message generation
```

A dial lands in the shared event stream. If the dialer attached a recording, a separate workflow is
triggered — separate on purpose, see below — which transcribes with Deepgram nova-3 (diarized, smart
formatting, ~1.5 s synchronous), sends the transcript to the model with a strict-schema tool
definition, and writes the result through an RPC that resolves the dial's `event_id` from phone and
call time so the audit is linked to the actual call.

Transcription was initially AssemblyAI; Deepgram replaced it for latency, with the old nodes left
disabled rather than deleted so the change was one toggle to revert.

## Design decision 1: the model judges, code counts

The rubric is a weighted multi-criteria score where some criteria may be non-applicable to a given
call. The model returns, per criterion, a score, an applicability flag and a justification note —
through strict-schema function calling, so the shape is guaranteed:

```
{
  schema_version, call_type, scored,
  criteria: [ { key, group, weight, score, applicable, note, note_pt } ],
  groups, score_pct, process_followed, summary_pt
}
```

**`score_pct` is then recomputed in code** as the weighted percentage over only the applicable
criteria. The model's own arithmetic is discarded.

This is not a precaution, it is a correction. An LLM asked to produce a weighted average over
criteria it has just scored, some of which it has just marked non-applicable, gets it wrong often
enough to matter — and a score that is wrong by four points moves a setter between performance tiers.
Separating judgment from arithmetic also means the weights can be changed without re-running
inference over the history.

The model choice follows from the same reasoning. A cheap model handles the simpler generative tasks
in this system; the audit runs on the full-size model, because a multi-criteria rubric with weights
and calibration overrides is the one place where reasoning quality maps directly onto a number
somebody's performance review depends on.

## Design decision 2: not every call is scoreable

`call_type` is classified first: sales call, reschedule and cancellation are scored; voicemail, no
contact, wrong number and instant brush-off are marked `scored = false` with a null score. There is
a deterministic anti-voicemail gate *before* the model, because paying for inference to discover a
call was eight seconds of hold music is pure waste.

Status derives from the score: approved at 80 or above, needs review from 50 to 79, rejected below
50, and `not_scored` for the unscoreable. Null scores drop out of averages rather than counting as
zero — the difference between "this setter has a 40% average" and "this setter's scoreable calls
average 78%, and most of their dials hit voicemail" is the difference between firing someone and
fixing a lead list.

## Design decision 3: calibration is a product decision

This is the part I would most want to be judged on, because it is where the system stopped being a
technical exercise.

The first version produced defensible scores that the sales manager disagreed with — and he was
right. Each disagreement became an explicit override in the rubric prompt:

- **Never deduct for audio quality, background noise, accent, difficult English, or an ambiguous
  name.** The setter controls none of those. Penalising them makes the score measure the *recording*
  instead of the work, and the setters working the hardest leads had the worst audio.
- **A lead who engaged — discussed scope, price, timeline, or raised objections — and then declined
  is a scored sales call**, not an unscoreable brush-off. That call *is* the setter's performance.
  Only the instant hang-up is unscoreable. Getting this wrong let a setter improve their average by
  mishandling calls badly enough that they got reclassified.
- **When the visit is already booked, the criteria about offering time slots do not apply** rather
  than scoring zero. A confirmation call that correctly confirms is a good call.
- **A remote estimate is a variant of a sales call**, detected from the appointment stage: having no
  on-site date is correct there, so the scheduling criteria become non-applicable and the
  confirmation criteria get full credit.
- **A service genuinely outside the client's scope** makes the booking criteria non-applicable
  instead of zeroing them.
- **Budget discussed qualitatively counts** from partial to full credit; don't require a number.
- **Don't require digging for pain when the project is already clearly specified.**

The generalisable lesson: **scoring the wrong thing precisely is worse than scoring the right thing
roughly**, because people optimise against the score you give them. Every one of those overrides
exists because the model was reasonable and the resulting incentive was not.

## Design decision 4: keep the transcript

Transcripts are stored in their own column on the audit record rather than discarded after scoring.
That single choice made the whole history re-auditable against a revised rubric with no
re-transcription, no cost from the speech vendor, and no involvement from the automation layer —
just inference over data already on disk.

A re-audit of the full history produced: 169 approved, 127 needs-review, 14 rejected, 623 not
scored, with scored calls averaging 78.6%.

One sharp edge learned there: **re-audit by patching the content columns directly, never through the
normal upsert function.** That function recomputes the audit timestamp and re-resolves client and
agent — which destroys the very historical linkage the re-scoring exists to preserve.

## What it fed

**Daily reports.** A snapshot per setter per day, with the team view invisible to setters by RLS.
Tiering is deliberately rule-based rather than model-based, so it is auditable and stable:
*insufficient* under 15 dials; *critical* at QA below 50, or three rejected calls, or zero
appointments on eight or more answered calls; *excellent* at three appointments and QA 78+; *good*
at one appointment and QA 62+; *attention* otherwise. Logic lives in a pure module separate from the
UI so it could be ported into a workflow and used to generate the stored snapshots without
duplicating the rules.

Each audit is attributed to the day of the *call*, not the day of the audit — they usually coincide,
and when they do not, the one that matters is when the work happened.

**Booking messages.** A short post-booking message, pre-generated per appointment with structured
fields extracted from the transcript, so the setter sends something consistent without writing it.
Two triggers: a database trigger on transcript arrival for the fast path, and a cron sweep as a
backstop. This is where the pipeline's real bottleneck became visible — not the booking, but the
transcript, with lag from call to transcript at roughly 5–7 minutes at p50 and up to 8 hours at p90,
driven by call duration and the dialer's own delivery.

It is also where the system's most instructive bug lived, and it is written up in full in
[`ENGINEERING-NOTES.md §III`](../ENGINEERING-NOTES.md): an unpaginated dedup query stopped at 1,000
rows, so 65 appointments past that ceiling were regenerated every five minutes forever — $6.77 a day,
logging success.

**Objection classification.** The "not interested" bar in the lead-status chart is clickable and
opens the distribution of *reasons*, each with its supporting quote from the transcript. Reasons come
from a model against a constrained set of twelve values rather than from keyword heuristics — the
manager's explicit preference, since the interesting objections were the ones no keyword list
anticipated.

Honest coverage note, which was reported alongside the numbers: of 1,569 not-interested leads only
479 had a recording, so roughly 1,090 can never have a reason. A dashboard that hides that denominator
is lying by omission.

The transcription queue for this had its own lesson: **store the empty result.** A recording that
transcribes to nothing must still be written, flagged as empty, or it returns to the queue forever
and head-of-line blocks everything behind it.

## Outcome

Full coverage instead of a weekly sample. A per-setter daily report the setter can see and the
manager can act on, with the team ranking hidden from the setter by row-level security rather than by
convention. A score that the sales manager agreed with, arrived at by writing down the disagreements
instead of tuning until the complaints stopped.

---

**Related:** [The platform](01-platform.md) · [CLAUD.IA](07-claudia.md) · [Engineering notes §VII (LLMs in a pipeline)](../ENGINEERING-NOTES.md)
