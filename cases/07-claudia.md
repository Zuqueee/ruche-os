# 7 · CLAUD.IA — natural-language queries over the operation

**Ask the database a question in plain language and get an answer that came from a query, or no
number at all.**

[← back to index](../README.md)

---

## Context

The OS had a screen for every team and about sixty views behind them. It still could not answer the
long tail: "how many clients in Florida churned this quarter", "which media buyer has the most
accounts with paused campaigns", "what's the average appointment count for clients who went live in
August". Each of those is a five-minute query for someone who knows the schema and an impossible
request for everyone else — so they became Slack messages to the two people who did.

## What was built

A chat agent, reachable from a widget inside the console, that answers questions about any area of
the operation by querying the database through a constrained tool surface.

```
user question (Portuguese or English)
      │
      ▼
agent, running on a small model
  │  system prompt carries the DATASET CATALOG inline
  │  max 2 model calls per question, capped output
  │
  ├─► claudia_catalog      what exists
  ├─► claudia_query        read a registered dataset with filters
  ├─► claudia_count        count rows
  └─► claudia_valores      distinct values of a column
      │
      ▼
registered views only — the view name comes from the catalog table, never from the model
      │
      ▼
answer + conversation log
```

Coverage comes from a **table**, not from code: a catalog of registered datasets across the
operation's areas — clients, call centre, customer success, media buying, finance, onboarding,
commercial. Adding an area to the agent's knowledge is one `INSERT`. The agent cannot query anything
that is not registered, and the view name it hits is read from the catalog row rather than generated
by the model, which removes SQL injection from the threat model entirely: the model chooses a
*dataset key*, not a query.

## Design decision: the catalog goes in the prompt, not behind a tool call

The obvious design gives the agent a `list_datasets` tool and lets it discover what exists. With a
two-call budget, that spends the first call discovering the schema and leaves the actual question
unanswered.

So the catalog is injected into the system prompt. The cost is prompt size; the benefit is that the
agent's single tool round trip is spent on the user's question.

The refinement that made it work: a scheduled function populates each catalog row with the **valid
values** of its enum-like columns — any column with a small, bounded set of distinct values. Without
those hints, the model invents plausible values. It will filter on `status = 'cancelled'` when the
column contains `churned`, get zero rows, and report that there are none.

## Design decision: a filtered zero is not "there are none"

This was the agent's most dangerous failure and it is worth stating as a general principle.

A query with a filter value that does not exist returns zero rows. Zero rows is indistinguishable
from a true zero, and the agent reported it as fact: *"there are no clients in that status."*
Confident, fluent, and wrong in a way the user cannot detect.

The fix is in the RPC, not the prompt. **When a filtered query returns zero rows, the function also
returns the set of values that do exist for that column.** The agent can then self-correct and
answer honestly: "there are none *with that status* — the statuses that occur are active, onboarding,
churned, and paused."

The general form: **an empty result should carry enough context to distinguish "nothing matches" from
"you asked wrong."** That is a data-layer responsibility. No amount of prompting fixes a function
that returns an unqualified empty set.

## Design decision: no tool call, no number

The agent runs on a small, cheap model. The single most important line in its system prompt is that
it may not state a number it did not retrieve.

Without it, a small model will answer "how many clients churned last month" directly, from nothing,
in a confident sentence. It is not hedged or hallucination-flavoured; it reads exactly like the
correct answers. This is the clearest illustration in the whole system of the trade-off between model
cost and prompt discipline: **the cheaper the model, the more of the correctness has to be structural
rather than instructed.**

Two more hard-won details:

- **Return only the tool call you executed.** When the model emits multiple tool calls and the
  implementation runs one, the conversation sent back must contain only that one — otherwise the API
  rejects the follow-up for having unanswered calls.
- **The automation platform's code sandbox cannot read environment variables.** The agent was broken
  for several days because an API key was read from the environment inside a code step, which the
  platform blocks. Network access and credentials have to go through the platform's HTTP node with a
  proper credential. A platform constraint, discovered the slow way.

## What it is honest about

Two limitations were documented rather than papered over:

**A view returning empty for a non-obvious reason.** One churn view returned nothing despite 52
recorded churn events in the table. Until that was understood, the documented instruction was explicit:
*do not trust a churn figure from the agent.* An agent that is right about nine areas and silently
wrong about the tenth is more dangerous than one that is wrong everywhere, because it has earned
trust it does not deserve across the board.

**It logs what it was asked.** Conversations are stored — for debugging and for seeing which
questions people actually have, which turned out to be the better input to the product roadmap than
asking them. The first version of that log table shipped with row-level security disabled and writes
open to the public anon key. Caught and closed in a later migration, and recorded in
[`ENGINEERING-NOTES.md §II`](../ENGINEERING-NOTES.md) as what it was: a new table that skipped the
default.

## Outcome

The long tail of data questions stopped being a Slack message to an engineer. The pattern that made
it maintainable is the catalog table: the agent's capability is data, so extending it does not
require touching the agent.

---

**Related:** [AI call auditing](02-ai-call-auditing.md) · [The platform](01-platform.md) · [Engineering notes §VII (LLMs in a pipeline)](../ENGINEERING-NOTES.md)
