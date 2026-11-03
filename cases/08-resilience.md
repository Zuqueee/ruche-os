# 8 · Infrastructure resilience

**A stateful automation engine that would not reconnect after a network blip, on a network that
blipped up to sixty times a day — recovered in ~3 minutes, with zero inbound webhooks lost.**

[← back to index](../README.md)

---

## Context

The automation layer — about forty workflows carrying every lead, dial, appointment, sync and
notification in the business — ran on one Hetzner host in queue mode, with its Postgres on a second
host. The Docker Swarm overlay network between them crossed the public internet.

That link flapped between five and sixty times a day. Short flaps were harmless. Past roughly twenty
seconds, n8n lost its database and parked in `waiting for database connection recovery`
**indefinitely**. Postgres recovered on its own; n8n did not.

Two incidents a week apart cost about two hours each of no webhooks and no scheduled jobs. During
that window, leads submitted on landing pages went nowhere: the webhook endpoint was refusing
connections, and the landing page had no retry.

The diagnosis mattered because the instinct was wrong. This was not memory and not disk — the
service was alive and idle, holding a dead connection. And the actual root cause, an overlay network
over the public internet between two hosts, was not something that could be fixed at the available
budget.

## What was built

Two independent layers, deliberately not one clever layer. The first restores the service; the
second makes the outage harmless while it lasts.

### Layer 1 — watchdog: restore in about three minutes

A shell script driven by a systemd timer on a sixty-second interval. It polls the readiness endpoint
of the editor service and of each webhook replica, and checks the worker's log for the stall
signature.

Three consecutive failures trigger a forced rolling restart — editor, then webhook, then worker, in
dependency order — with a ten-minute cooldown so a genuine platform outage cannot become a restart
loop.

The detail that makes it a system rather than a cron job: **each restart is filed as an incident in
the OS's own operations module.** A self-healing component that heals silently teaches you that your
infrastructure is fine. The incident record is how "it restarted itself eleven times last week"
becomes visible, which is the information that justifies fixing the real cause.

It also ships with a forced-failure test path, so the recovery can be exercised on purpose — because
a recovery mechanism nobody has triggered deliberately is a hypothesis, not a mechanism.

### Layer 2 — queuing relay: lose nothing

A small service in front of the public webhook hostname, which the reverse proxy routes to at high
priority.

While n8n is healthy it is **transparent**: same response body, same CORS headers, same latency,
verified by comparison rather than assumed.

When n8n is unreachable — connection refused, or 502/503/504 — the relay persists the request to a
local embedded database, answers `200 {"queued": true}` with correct CORS headers, and replays the
queue in order when n8n returns, tagging replays with a header so downstream workflows can tell.

The landing page is what makes this safe. It fires its browser-side conversion event with the **same
idempotency key** as the server-side event (see [case 6](06-conversion-tracking.md)), so a webhook
replayed twenty minutes later cannot double-count a conversion. Without that shared key, replay
would trade lost leads for inflated metrics.

## Why two layers instead of one

The watchdog alone still drops every webhook that arrives during the three-minute recovery, and a
lead form submission that returns an error is a lead the agency paid for and lost.

The relay alone keeps the data but leaves the service down until a human notices.

Together, the failure degrades from *"two hours of lost leads and silent stalled automation"* to
*"three minutes of queued requests that then process normally."* Neither layer fixes the network, and
that is the point: **when the root cause is outside your control or your budget, the engineering goal
is to make the failure boring.**

## The capacity problem, and the fix that was actually available

Eleven services shared 3.7 GiB of RAM on the automation host — n8n's four services plus a WhatsApp
gateway, a message broker, a secrets service, the reverse proxy and the container UI.

Worker concurrency went 8 → 2 → 6 → 4 → 3 over four months, every step forced by an out-of-memory
kill. The conclusion was written down once rather than re-litigated each time: *the host needs more
RAM; tuning the heap only changes which service dies.*

But the fix available at that budget was not concurrency — it was **payload size**. In queue mode,
every value passed between nodes is persisted to the queue database. Measuring stored execution data
per workflow found that two ClickUp sync workflows accounted for **76%** of all of it, and one
workflow was storing **13 MB per execution** because it passed entire task objects — 81 custom fields
each — from node to node, when it needed four of those fields.

**Select your fields at the first node, not the last.** That one change did more for stability than
any memory tuning, and it is the kind of problem that only appears when you measure where the bytes
are instead of where you assume they are.

## The monitoring that caught what the platform called success

The automation platform's own execution log only shows errors. A workflow that runs, processes zero
items, and exits is a success by that definition — and is the most common real failure mode in an
integration-heavy system.

A separate auditing script was built to find the classes the platform cannot see: runs that completed
silently with no output, scheduled workflows that did not fire at all, workflows whose processed
volume dropped to zero, and payloads large enough to threaten the queue database.

That inverts the usual alerting posture. **In a pipeline, the alarming condition is usually not an
error — it is the absence of work.** Nothing in the platform's UI will ever tell you that.

## Outcome

Unattended recovery in roughly three minutes instead of until-someone-notices, no inbound webhook
lost during an outage, each self-heal visible as an incident rather than invisible, and the queue
database's growth traced to its actual source. All of it on infrastructure that kept flapping,
because the flapping was never the part that could be fixed.

---

**Related:** [Conversion tracking](06-conversion-tracking.md) · [Engineering notes §VIII (infrastructure)](../ENGINEERING-NOTES.md) · [Engineering notes §VI (false success)](../ENGINEERING-NOTES.md)
