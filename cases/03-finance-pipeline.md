# 3 · Finance pipeline

**Eight banks and payment gateways, two currencies, one append-only journal, and a monthly close
that went from days of manual work to a reviewable pipeline.**

[← back to index](../README.md)

---

## Context

A company operated from Brazil, billing U.S. clients, holding money in both currencies. Revenue
arrived through Stripe (settling in BRL for charges presented in USD), a Brazilian payment gateway,
and direct transfers. Costs left through two Brazilian banks with current accounts and credit cards,
an investment account, a U.S. bank account, and two international payment processors.

Ten sources. Two currencies. The monthly close was a person exporting statements, classifying lines
by hand in a spreadsheet, and reconciling totals — days of work, and the kind of work where a single
mis-keyed decimal point survives to the P&L.

## What was built

A multi-currency journal in the same Postgres as the rest of the OS, with a classification layer
per source and a reconciliation check per source.

```
Stripe · Asaas · BoA · C6 (acct + card) · Unicred (acct + card + invest) · PayPal · Payoneer
      │
      │  per-source parser: each bank's own CSV/XLS quirks
      ▼
  classification against a 12-group chart of accounts
      │
      │  BRL → USD, with the FX rule depending on the source (see below)
      ▼
  finance_journal  (append-only, USD functional currency, SHA-1 dedup key)
      │
      ├─► per-source tie-out  ──►  reconciliation record
      │
      └─► P&L by accrual period · cash flow by account and period
```

Core objects: `chart_of_accounts` (12 groups — revenue, deductions, COGS, fixed costs, operating
expenses, depreciation, through to transfers and loans), `finance_journal` (the master journal),
`finance_import_jobs`, and `finance_reconciliation`. Output views for P&L by month, cash flow split
operating/investing/financing, and account-by-month drill-down.

The journal is **append-only** and keyed by a SHA-1 dedup key, so re-running an import is safe and a
correction is a new row rather than an edit. A financial record that can be silently edited is a
financial record nobody can audit.

## The decision that defines the design: two different "dates"

Every transaction carries both a **transaction date** and an **accrual period**, and they are
routinely different.

A client paying an implementation fee in a single charge in August is not August revenue — the
service is delivered over following months, so one charge becomes several journal lines, one per
period it is earned in. A credit-card purchase hits the bank when the statement is paid, but belongs
to the period it was spent in.

This is ordinary accrual accounting and it is the thing a naïve pipeline gets wrong, because the only
date in the source data is the one the money moved. The consequence is that **P&L runs on accrual
period and cash flow runs on transaction date and account**, from the same journal — and when the two
disagree, that disagreement is the information, not a bug.

## The problem that made the whole project worth doing

The manual journal that the pipeline was validated against turned out to contain its own corruption:
a handful of entries had lost a decimal place, turning a few thousand in local currency into a few
million. The automated parse read the source statement correctly and disagreed with the spreadsheet.

That inverted the project's premise. The pipeline was supposed to reproduce the manual work;
instead, reconciling line by line against the source statements showed the pipeline was right and
the baseline was wrong in specific, findable places. The reconciliation layer went from "prove the
automation matches" to "find where either side is wrong", which is what it should have been from the
start.

**A reconciliation you only run until it agrees is not a control.** The one worth building reports
the difference and makes you explain it.

## Design decision: the FX rule depends on the source

The functional currency is USD. Brazilian-currency amounts convert at the central bank's official
daily rate, fetched from its public API — correct for a local bank transfer or a local fee.

It is *wrong* for the payment processor. U.S. clients were charged in USD; the processor settled in
BRL. The real USD figure is the presented amount the client was actually charged, which the processor
reports on the transaction. Converting the settled BRL back to USD at the central bank rate produces
a number that is close, plausible, and not what anyone paid — and the giveaway was that the manual
journal's USD figures were round numbers while the derived ones never were.

**Take the FX rate from the system that performed the conversion.** Derive it only where nothing
performed one.

## Design decision: the reconciliation is per source, because the sources differ

There is no single tie-out that works for ten sources. Each has the check its data actually supports:

- The payment processor zeroes out **by payout** against an itemised report — every charge, fee and
  refund inside a payout must sum to the deposited amount.
- The U.S. bank reconciles on **total credits and total debits** for the period.
- The investment account reconciles on **gross balance movement**.
- Credit cards reconcile **per statement**, on the cash model, since that is when the money leaves.

Building these as separate, documented procedures per source — rather than one generic importer — is
what made each one trustworthy. The parsing differences are not incidental either: every bank exports
a different dialect of CSV or XLS, with its own date format, decimal separator, column naming, and
header junk. That is unglamorous work and it is most of the reliability.

## Two incidents worth recording

**A plausible fix that would have double-counted costs.** It was suggested that the payout fee be
booked as a cost. Investigated against the processor's own data: the transaction fee was *already*
booked as a cost under its own account code, and the paired entries around the payout are an internal
settlement that sums to zero. Booking both would have inflated costs by the full fee amount every
month. The right answer was to change nothing, show the ledger entries, and explain why — and then
write down the reasoning, because a "missing cost" is exactly the kind of thing that gets re-raised
every quarter.

**A spreadsheet filter cost six build cycles.** An intermediate stage wrote to a Google Sheet. A
filter left active on the destination tab hid rows, and the append wrote *past* the end of the hidden
data — so each run appended below the real last row and the totals drifted. Nothing errored. The rule
that came out of it: **never leave a filter on a tab that an automated process appends to**, and clear
tabs programmatically rather than by hand.

## An honest note on sequencing

The first implementation wrote to the OS database directly. It was then deliberately pointed at a
spreadsheet instead, and the database path was paused — because the numbers needed to be validated
against the manual baseline by a person, in a tool that person could inspect, before anything landed
in the system of record.

Then that intermediate stage was itself retired once per-source reconciliation procedures became the
source of truth. Two rewrites of the same pipeline, which looks like churn and was not: the thing
being built was *trust in the numbers*, and that could not be shortcut by writing to the right table
sooner.

## Outcome

A monthly close that produces a reviewable journal with a per-source tie-out instead of a
spreadsheet nobody could audit. A canonical journal of 1,336 transactions across ten sources,
reconciled to the cent for the validation period, with accrual periods spreading deferred
implementation revenue across the months it was actually earned. P&L and cash-flow views in the same
database as client, lead and appointment data — which is what makes per-client profitability a query
rather than a project.

Specific figures from the agency's accounts are not published here; the systems are the subject, not
the company's financials.

---

**Related:** [The platform](01-platform.md) · [Engineering notes §IV (time zones)](../ENGINEERING-NOTES.md) · [Engineering notes §IX (migrations)](../ENGINEERING-NOTES.md)
