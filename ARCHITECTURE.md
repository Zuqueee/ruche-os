# Architecture

How the Ruche OS was put together: what each layer was allowed to do, what the data model looked
like, and the path a single click took from an ad to an invoice.

---

## 1. Four layers, four different jobs

```
                   ┌──────────────────────────────────────────────┐
  Third-party      │ ClickUp · GoHighLevel · HotProspector         │
  systems of       │ Meta Ads · Google Ads · Make · Fathom         │  not ours; some of
  record           │ Stripe · Asaas · Plaid · banks                │  this data exists
                   └───────────────────────┬──────────────────────┘  ONLY in here
                                           │
                   ┌───────────────────────▼──────────────────────┐
  Ingestion and    │ n8n, queue mode (Redis), Docker Swarm         │  ~40 workflows
  automation       │ editor · webhook ×2 · worker                  │  schedules + webhooks
                   └───────────────────────┬──────────────────────┘
                                           │  RPC only — never a direct table write
                   ┌───────────────────────▼──────────────────────┐
  Source of        │ PostgreSQL (self-hosted Supabase)             │  the business rules
  truth            │ tables · views · RPCs · triggers · pg_cron    │  live HERE
                   │ row-level security on every table             │
                   └───────────────────────┬──────────────────────┘
                                           │  PostgREST + anon key (public by design)
                   ┌───────────────────────▼──────────────────────┐
  Presentation     │ React + TypeScript console, 10 roles          │  decides what to SHOW
                   │ CLAUD.IA natural-language agent               │  never what is ALLOWED
                   └──────────────────────────────────────────────┘
```

**Why ingestion never writes directly.** The PostgREST anon key ships inside the front-end bundle,
so it is effectively public. RLS therefore blocks anonymous `POST` outright, and every write path —
whether it comes from n8n, a script, or the console — goes through a `SECURITY DEFINER` function
that validates its own inputs. The practical benefit showed up on every schema change: the contract
was the function signature, not the table shape, so a column could move without breaking nine
workflows.

**Why the rules are in the database.** Four teams read the same numbers and each had an incentive to
define them favourably. A rule implemented in Postgres — a trigger, a `CHECK`, a view — is a rule
that produces the same answer in the console, in a workflow, in CLAUD.IA, and in a reconciliation
script. Two examples that earned their place:

- `normalize_client_lifecycle_status` — when a client's ClickUp task closes, a trigger sets
  `lifecycle_status = 'churned'`. Churn stopped being an opinion.
- `trg_ghl_leads_client_id` — the owning client of a lead is **frozen at insert**. Deriving it live
  looked cleaner and was wrong: when a client churned, their entire lead history became orphaned and
  historical reports silently changed.

---

## 2. The data model, in the shape the business actually had

### `clients` — the master record

Roughly a hundred columns, because this one row had to answer questions from five different teams.
It is synchronised from ClickUp (see §4) and partly editable inside the OS.

The fields that mattered: `clickup_task_id`, the human-facing `#NNN` number and its integer form,
`lifecycle_status` and `churn_date`, `ghl_location_id` (unique — one CRM sub-account per client,
enforced by trigger), `account_id_meta` (unique), the owning media buyer and CS squad,
`appointment_quantity` and `billing_type`, `timezone` and `company_state`, and the
`only_call_center` / `no_call_center` routing flags.

Two mechanisms made a shared master record survivable:

- **`locked_fields text[]`** — a human edit in the OS can pin a field so the next ClickUp sync
  cannot overwrite it. Without this, the robot always won, and the people who knew the client best
  stopped correcting data.
- **`COALESCE(NULLIF(new, ''), old)` across all 39 synced fields** — a value in ClickUp overwrites;
  an *empty* value does not erase. This is the difference between a sync and a data-loss event.

### `events` — one append-only stream for two things that look alike

Dials and booked appointments both describe "something happened to a lead at a time", so they share
a table with a discriminating `type` and a namespaced `external_id`: `hp_call:…` for dials from the
dialer, `ghl_…` for appointments from the CRM, `manual_…` for hand entry, `bf_…` for backfills.

`client_id` is `NOT NULL` and frozen. `stage_booked` carries the appointment's status. UTM fields
are attached per CRM contact. The namespaced external ID is what made backfills safe to re-run:
every row knows which process created it, so a bad import can be deleted by prefix.

### The rest

`ghl_leads` (CRM leads, with a view that strips internal test tags and soft-deleted records),
`call_audits` (transcript plus scored rubric), `ad_insights` (one row per client × date × platform ×
ad), `campaign_status`, `onboarding_tasks`, `receivables`, `finance_journal` and
`chart_of_accounts`, `client_touchpoints`, `ops_tickets`, `profiles` and `page_access_overrides`,
`niche_benchmarks`.

Roughly sixty views sat on top, one per screen, so the front end never assembled a business
definition out of raw tables.

---

## 3. Access control

`profiles.role` carried ten roles — admin, cc_manager, cs_manager, mb_manager, finance, cs_agent,
mb_analyst, cc_setter, viewer, client_user — plus three flags that cut across them: `is_admin`,
`is_ops`, `approved`.

Enforcement came in three tiers, deliberately redundant:

1. **Restrictive RLS policies on every table.** Two of them were pure lockouts: an unapproved user
   sees nothing at all, and a call-center setter is denied table access entirely — they read their
   call data only through `rpc_cc_call_data(p_from, p_to)`, which filters to their own rows.
2. **`SECURITY DEFINER` helpers** that the policies and RPCs both call: `is_admin()`,
   `is_approved()`, `current_role_name()`, `can_access(domain)`, and `current_user_access()` — which
   returns role, approval and admin flag in a single round trip, because the console was making
   three.
3. **Per-page registration in the front end** with per-user overrides in `page_access_overrides`.
   This layer is a convenience — it keeps people out of screens that would only confuse them. It is
   explicitly *not* the security boundary, and the code says so.

A fourth, softer layer sat in the read path: a single shared visibility filter hid archived clients,
test entries, external accounts and call-center-routing exclusions. Having it in one module instead
of thirty screens is the only reason the numbers on two different pages agreed.

---

## 4. The hardest integration: a two-way sync with ClickUp

ClickUp was where the commercial team already lived, so it stayed the system of record for the
client roster — which meant a bidirectional sync between a spreadsheet-shaped SaaS and a relational
database. Six lists were involved; two of them held the *same* client with the *same* custom fields,
one as the data source and one as the operational mirror.

What this taught, in order of how much it cost:

- **Map custom fields by ID, never by name.** A rename of "Squad CS" to "CSM" silently dropped the
  CS owner for 36 clients. The transform now maps by field ID with name as fallback only.
- **A closed task is not an absent task.** The nightly scan does not read closed tasks, so churn
  would never propagate through it; churn arrives through the real-time webhook instead. And when a
  task closes, the transform nulls out the CRM location and ad-account IDs — not to erase them (the
  `COALESCE` upsert prevents that) but to stop a churned client from contending for a `UNIQUE`
  constraint with the client who inherited that account.
- **Pagination that fails open is worse than pagination that fails.** A misspelled option name
  (`maxRequestsHere` instead of `maxRequests`) meant the sync quietly processed 100 of ~319 tasks
  for an unknown length of time. Nothing errored. The lesson generalised into a rule:
  *any paginated read needs a post-condition that asserts the count.*
- **A `200` response is not a success.** The RPC could return HTTP 200 with
  `{success: false}` in the body. A verification node now reads the body and throws, which routes
  the run to the shared error workflow.
- **Writing custom fields needs a different endpoint from writing the task.** `PUT /task/{id}`
  silently ignores `custom_fields`; each field needs its own `POST /task/{id}/field/{field_id}`,
  with a different payload shape per field type.

---

## 5. The path of one click

This is the sequence the whole system existed to make traceable.

```
 1  A contractor's ad runs on Meta or Google.
       ad-level spend lands in `ad_insights` the next morning, one row per ad per day

 2  Someone clicks, lands on a landing page, submits a form.
       the page captures fbc/fbp and fires Lead browser-side
       the form creates a contact in that client's CRM sub-account

 3  The lead reaches the OS.
       CRM → Google Sheet (append-only) → RPC `fn_upsert_leads` → `ghl_leads`
       owning client resolved once and frozen

 4  A call-center setter dials it.
       dialer webhook → `events` (type `dial`, `external_id = hp_call:…`)
       if there is a recording, the audit pipeline is triggered

 5  The call is transcribed and scored.
       Deepgram nova-3 → LLM with a strict-schema rubric → `fn_upsert_call_audit`
       the score is recomputed deterministically in SQL, not trusted from the model

 6  If the call booked a visit, an appointment event is written.
       Make → `events` (type `appointment_booked`, `stage_booked` = CRM stage)
       a booking message is pre-generated for the setter to send

 7  Appointment quantity meets the contract.
       Goal Tracker compares delivered appointments against `appointment_quantity`
       from go-live, in the client's own timezone

 8  The appointment becomes money.
       `receivables` → CRM invoice in the finance sub-account → dunning ladder over WhatsApp
       payment lands in one of 8 banks or gateways

 9  The money becomes a statement.
       reconciliation per source → `finance_journal` (USD functional) → P&L and cash-flow views
```

Every step in that chain has a failure mode that produced a wrong number at least once, and almost
all of them were *silent*. That is the subject of
[`ENGINEERING-NOTES.md`](ENGINEERING-NOTES.md).

---

## 6. What I would build differently

Honest version, written after the shutdown decision rather than during the build.

**Multi-tenancy from day one, or an explicit decision not to.** There is no `tenant_id` anywhere.
RLS partitions by *role*, not by organisation, and the client roster is coupled to fixed ClickUp
field IDs. That was the right call for an internal tool and it is exactly what made the system
unsellable as a product later. The cost of a `tenant_id` column on day one is one afternoon; the
cost of adding it to 30 tables and 60 views afterwards is a rewrite.

**Migrations as state, not as history.** Much of the SQL was run by hand in the Studio editor and
then committed, so file numbers were reused by parallel workstreams (62, 73, 79 and 80 each appear
twice) and some fixes exist only in the database. The repository is an honest log of intent and a
useless description of state. A real migration runner with a checksum table would have cost very
little and would have made the database reconstructible from git — which, at shutdown, it was not.

**A scheduled logical backup from the first week.** The front end had 1,347 commits in two places.
The database and the ~40 workflows existed in exactly one place: the servers. By the time anyone
asked for a dump, the honest answer was that none existed, and the cheapest faithful rescue was a
VM snapshot. Nothing in this system was hard to back up. It simply was never anybody's task.

**Secrets with one owner.** Tokens ended up in three habitats — an n8n credential store, hard-coded
values inside workflow code nodes, and a public config webhook that returned live API tokens to
anyone who called it. Each one arrived as a reasonable shortcut under deadline. Together they are
the part of this system I would most want to have done right the first time, and the part that
needed the most work before the servers could be safely turned off.
