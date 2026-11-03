# 1 · The platform

**One Postgres as the source of truth for an agency that was running on six disconnected SaaS tools.**

[← back to index](../README.md)

---

## Context

The agency sold booked appointments to U.S. home-service contractors. Roughly a hundred active
clients, each with their own CRM sub-account, their own ad accounts, their own time zone, and their
own contracted appointment quota. Four internal teams — media buying, customer success, call centre,
finance — each needed a different view of the same client.

What existed instead: the client roster in ClickUp, the CRM in GoHighLevel, dials in a HotProspector
dialer, spend in Meta and Google Ads, appointments in a Make-managed spreadsheet, invoices in a
separate CRM sub-account, and the books spread across eight banks and gateways in two currencies.
Four teams, four private spreadsheets, four different answers to "how many appointments did client
#437 get last month".

## What was built

A React console over a self-hosted Postgres, fed by about forty n8n workflows, with the business
rules implemented in the database rather than in any of the three places they could have gone.

| | |
|---|---|
| Console | 1,347 commits · 545 TS/TSX files · ~165k lines under `src/` |
| Database | 30+ tables · ~60 views · `SECURITY DEFINER` RPCs · triggers · `pg_cron` · 109 dated migrations |
| Roles | 10, with restrictive RLS on every table and per-user page overrides |
| Areas | Overview, Clients, Goal Tracker, Call Centre, Customer Success, Media Buying, Finance, Onboarding, Commercial, Ops, Users |

Stack: Vite, React, TypeScript, TanStack Router and Query, Tailwind v4, shadcn/ui, Recharts,
supabase-js. PostgREST over self-hosted Supabase. Deployment published on push to `main`.

## The design decision that mattered most

**Business rules went into Postgres, not into ClickUp and not into the front end.**

The pressure to do otherwise was real: rules in the front end are faster to write and easier to
demo. The reason not to is that four teams read the same numbers and each had an incentive to define
them favourably. A definition that lives in a view produces the same answer in the console, in a
workflow, in the natural-language agent, and in a reconciliation script. A definition that lives in
a component produces one answer per component.

Two rules that justify the whole approach:

- **Churn is a trigger.** When a client's ClickUp task closes, `normalize_client_lifecycle_status`
  sets `lifecycle_status = 'churned'`. Applied to all 218 clients in one pass when it shipped. Churn
  stopped being a per-team opinion.
- **Lead attribution is frozen at insert.** `trg_ghl_leads_client_id` resolves the owning client
  once, at write time, preferring non-churned clients. Deriving it live reads better and is wrong:
  when a client churned, every lead they had ever generated became orphaned and historical reports
  changed retroactively.

The corresponding discipline: **no external system ever writes to a table.** Every write path goes
through a `SECURITY DEFINER` function. The anon key is in the browser bundle and therefore public,
RLS blocks anonymous writes outright, and the contract that nine workflows depend on becomes a
function signature rather than a table shape — so columns can move without a coordinated release.

## Access control, in three deliberately redundant layers

1. **Restrictive RLS on every table.** Two policies are pure lockouts: an unapproved account sees
   nothing, and a call-centre setter is denied table access entirely — they read their own call data
   only through `rpc_cc_call_data(p_from, p_to)`.
2. **`SECURITY DEFINER` helpers** shared by policies and RPCs: `is_admin()`, `is_approved()`,
   `current_role_name()`, `can_access(domain)`, and `current_user_access()`, which returns role,
   approval and admin flag in one round trip because the console was making three.
3. **Per-page registration with per-user overrides.** This layer keeps people out of screens that
   would only confuse them. It is explicitly *not* the security boundary, and the code comment says
   so — because the next person to read it will assume it is.

A fourth, softer layer sits in the read path: one shared module hides archived clients, test
entries, external accounts, and call-centre routing exclusions. Having that in a single place
instead of thirty screens is the only reason two pages ever agreed with each other.

## The hardest part: a two-way sync with ClickUp

ClickUp stayed the system of record for the roster because the commercial team already lived there.
That meant a bidirectional sync between a spreadsheet-shaped SaaS and a relational database — six
lists, two of which held the *same* client with the *same* custom fields, one as source and one as
operational mirror.

Ranked by what each lesson cost:

**Map custom fields by ID, never by name.** Renaming a field from "Squad CS" to "CSM" silently
dropped the customer-success owner for 36 clients. Nothing errored; a column just went quiet. The
transform maps by field ID, with name only as a fallback.

**A closed task is not an absent task.** The nightly scan does not read closed tasks, so churn could
never propagate through it — it arrives via the real-time webhook instead. And when a task closes,
the transform deliberately nulls the CRM location and ad-account IDs. Not to erase them (the
`COALESCE` upsert prevents that) but to stop a departing client from contending for a `UNIQUE`
constraint with the client who inherits the ad account. Before that, one client sat in a stale status
for weeks because a single unique-key collision was failing the entire row.

**Humans need to be able to beat the robot.** `COALESCE(NULLIF(new, ''), old)` across 39 fields
means a value overwrites and an empty value does not erase — correct, and it also meant a human's
correction in the OS reverted at the next sync. `locked_fields text[]` lets an edit pin a field;
`fn_update_client(p_client_id, p_patch, p_lock)` writes and locks in one call, and
`fn_unlock_client_field` releases it. Without this, the people who knew each client best stopped
bothering to fix anything.

**Assert what you expected to read.** A misspelled pagination option (`maxRequestsHere` for
`maxRequests`) meant the sync quietly processed 100 of roughly 319 tasks, for an unknown period, with
no error. An unknown option name should be a hard failure; it was not, so the post-condition has to
be ours.

**A 200 is not a success.** The RPC could return HTTP 200 with `{success: false}`. A verification
node now reads the body and throws, routing the run to a shared error workflow.

**Writing a custom field needs a different endpoint from writing the task.** `PUT /task/{id}`
silently ignores `custom_fields`. Each field needs `POST /task/{id}/field/{field_id}` with a payload
shape that differs per field type — text, epoch milliseconds plus a `time: true` flag, label ID
arrays, dropdown order indexes, lat/lng objects — and clearing a value means an empty string, not a
zero.

## Outcome

One place to answer a question about a client, and the same answer wherever it was asked. The screens
each team actually used daily — the media-buying tracker, the call-centre tracker and call data, the
customer-success wallet and health score, the finance receivables and journal, onboarding progress,
the goal tracker — all read from views rather than assembling definitions locally.

The honest limitation, stated in full in [`ARCHITECTURE.md §6`](../ARCHITECTURE.md): there is no
`tenant_id` anywhere. RLS partitions by role, not by organisation. That was the right decision for an
internal tool serving one company, and it is precisely why the system could not later be sold as a
product. The decision was correct; not writing it down as a decision is what made it a surprise.

---

**Related:** [AI call auditing](02-ai-call-auditing.md) · [Engineering notes §II (RLS)](../ENGINEERING-NOTES.md) · [Engineering notes §V (upserts)](../ENGINEERING-NOTES.md)
