# Bank Settlement Reconciliation — May & June 2026

This project checks whether the money the bank actually paid us matches the sales our own system
recorded, for two months in a row.

---

## What this is about, in everyday words

Our system keeps a list of every sale we made. The bank sends a statement of every payment it made
into our account. In a perfect world the two lists would be identical, and checking them would take
five minutes.

They are never identical. The bank does not pay the way our list is written:

* it sometimes pays **ten sales in one lump**,
* it sometimes **splits one sale into three payments** on three different days,
* it sometimes pays **late** — a May sale turns up in June,
* it sometimes pays **twice** by mistake,
* it sometimes keeps a **small fee** and pays slightly less,
* it sends money we **cannot identify at all**,
* and it almost always **mistypes our reference number** on the way.

"Reconciliation" is the job of lining the two lists up anyway, and being honest about whatever will
not line up. That is what the notebook in this folder does.

## What's in this folder

| File | What it is |
| --- | --- |
| `Bancapp_Reconciliation.ipynb` | The whole solution. Open it, run every cell top to bottom. |
| `summary_writeup.pdf` / `.md` | The 2-page write-up asked for in the brief. |
| `data/` | The four input CSV files, unchanged. |
| `output/reconciliation_output.csv` | The main result — every match, its status and its money difference. |
| `output/exception_report.csv` | Everything that needs a human, with a category and a suggested action. |
| `output/backlog_report.csv` | May's unfinished items, and what June did with each of them. |
| `output/input_validation_report.csv` | Every data-quality check we ran and what it found. |
| `output/summary_control_report.csv` | The headline numbers for both months, on one page. |
| `output/reconciliation_dashboard.html` | A one-page summary — double-click to open it in any browser. |

## How to run it

You need Python with `pandas` installed. Nothing else — no internet, no database, no extra libraries.

```bash
pip install pandas
jupyter notebook Bancapp_Reconciliation.ipynb
```

Then run every cell from top to bottom. It takes a few seconds and rewrites everything in `output/`.
The same inputs always give the same outputs — there is no randomness anywhere in the code.

## The four answers the report can give

Every item ends up in exactly one of these four buckets — the same four words the brief
asks for, just written in title case instead of shouting caps:

| Status | In plain words |
| --- | --- |
| **Matched** | The sale and the payment agree. Nothing to do. |
| **Partial Match** | We found the right payment, but some money is still missing. |
| **Open** | No payment has arrived for this sale yet. |
| **Exception** | Something is wrong — a double payment, money we cannot identify, or a case that needs a person. |

## The four shapes a match can take

The file keeps the standard 1:1 / 1:N / N:1 / N:M labels and adds a short gloss next to
each one, so nobody has to look the label up:

| Cardinality in the file | Meaning | Example from the data |
| --- | --- | --- |
| **1:1 - one payment, one credit** | One sale, one payment | The ordinary case — 209 in May |
| **1:N - one payment, many credits** | One sale, several payments | `PART SETTLEMENT PRMAY48951595-P1/-P2/-P3` |
| **N:1 - many payments, one credit** | Several sales, one payment | `SETTLEMENT BATCHMAY006` pays a whole batch |
| **N:M - many payments, many credits** | Several sales, several payments | `NET SETTLEMENT M1016 2026-05-19 1/2` and `2/2` |

The three leftover shapes are **1:0 - payment with no credit**, **0:1 - credit with no
payment** and **N:0 - entries that cancel out**.

## What the columns mean

The main file, `reconciliation_output.csv`, has thirteen columns and no jargon:

| Column | What it holds |
| --- | --- |
| Month | May 2026 or June 2026 |
| Group ID | The id of this unit of work, so you can trace it |
| Our Transaction IDs | The sale(s) from our own system |
| Bank Line IDs | The bank statement line(s) that paid them |
| Payment Reference | The reference the two sides are matched on |
| Cardinality | One of the shapes listed above |
| Our Amount | What our books say |
| Bank Amount | What the bank actually paid |
| Difference | Our amount minus the bank amount |
| Status | One of the four answers above |
| Days Old | Days from the sale date to the end of that month |
| Match Method | Which matching pass found this (payment reference, batch, part settlement, etc.) |
| Comments | One short sentence explaining the row |

`exception_report.csv` adds two columns to that idea: **Problem** (what went wrong) and
**What To Do** (the action, in one sentence). `backlog_report.csv` uses the two labels the
brief asks for — **Likely Settlement Lag** and **Genuine Exception** — in its
**Classification** column. `summary_control_report.csv` and `input_validation_report.csv`
follow the same plain-English style.

## How the matching works

**Step one is cleaning up the bank's text.** The bank hides our reference number inside a line of
narration, and mangles it on the way. Real examples from this data:

| What the bank wrote | The problem |
| --- | --- |
| `SETTLE/PRMA Y95565484/PAYOUT` | a space dropped into the middle |
| `SETTLE/prmay85386435/PAYOUT` | written in lower case |
| `SETTLE/NEFTPRMAY58438711/PAYOUT` | the channel name glued to the front |
| `SETTLE/PRMAY832847/PAYOUT` | the reference chopped short |

Making everything upper case and deleting every space fixes the first three. For the fourth, we look
for references in our own list that *start with* what the bank wrote — if exactly one does, that must
be it. If more than one does, we leave it alone rather than guess.

**Step two is matching, in a deliberate order — narrowest first.**

1. Sales that cancel against a reversal in our own books (no payment is ever coming for these).
2. Batch payments, where one credit pays many sales.
3. Instalments, where many credits pay one sale.
4. The ordinary reference match, one sale to one payment.
5. Merchant-day sweeps, where the bank netted a whole day for one merchant.
6. Whatever is left over: sales with no payment, and payments we cannot place.

The order is not arbitrary. Step 5 is greedy — it grabs *everything* for a merchant on a date. If it
ran earlier it would swallow sales that actually have their own payment. `TXNMAY00104` is exactly
that: it sits inside the M1010 sweep window on 19 May, but is genuinely paid on its own line
`BLMAY00104`. Because the sweep runs last, it can only pick up the true leftovers.

## The rule that stops anything being counted twice

This is the most important control, and it is deliberately simple.

The code keeps two lists: transactions already used, and bank payments already used. **Every** match,
in **every** step, has to go through a single function called `make_group()`. Before that function
creates anything, it checks whether any record involved is already on those lists. If even one of
them is, the whole match is refused and nothing changes. There is no way around it, because there is
no other way to create a match.

Here is what that buys us. The bank paid reference `PRMAY20770971` twice — ₹12,850.78 on two
different UTRs. The first payment settles the sale. The second one tries, is refused because the sale
is already used, and is written into the exception report as a duplicate credit. Without that check
we would have quietly recorded ₹12,850.78 of income that never existed. Eleven payments across the
two months were caught this way.

The notebook finishes by proving it: every one of the 1,148 records must appear in exactly one group.
If it ever does not, the notebook stops with an error instead of printing wrong numbers.

## Carrying May into June

June is not reconciled on its own. Everything still unpaid at 31 May is added to the June pool, so a
May sale can be cleared by a June payment. Of the 30 items still open at 31 May, **22 were paid
during June** (₹11,88,928) and 8 were not.

Every row records whether the sale came from May or June, so you can read three things off one file:
June sales paid in June, May sales cleared in June, and everything still unpaid at 30 June.

## Late, or actually a problem?

> **If a sale is still unpaid at month end but is 5 days old or less, the file calls it a
> normal delay — the bank is just running behind. Older than that, it says needs checking,
> and someone should look at it.**

Five days covers a normal settlement cycle plus a weekend. The data backs the rule up almost
perfectly: of May's 30 unpaid items, the 22 that were 5 days old or less **all** got paid in June,
and the 8 that were older **never** did.

## What we found

| | May 2026 | June 2026 |
| --- | ---: | ---: |
| Sales to settle | 328 | 318 (includes 30 carried from May) |
| Bank payments received | 252 | 250 |
| Matched | 226 | 228 |
| Partly matched | 3 | 2 |
| Still open | 30 | 32 |
| Exceptions | 15 | 12 |

87.6% of May's value and 85.3% of June's value matched with no difference at all. Of everything that
did match, the total unexplained difference is **₹941 in May and ₹667 in June** — a rounding error on
₹1.3 crore, and every rupee of it is listed by name.

Exceptions across both months:

| Problem | Items | Value | What to do |
| --- | ---: | ---: | --- |
| Unsettled transaction | 22 | ₹11,78,729 | Chase the acquirer. This payment is overdue. |
| Duplicate credit | 11 | ₹4,96,223 | Ask the bank to take the extra credit back. |
| Orphan credit | 9 | ₹4,22,677 | Ask the bank whose money this is. |
| Internal self-netting pair | 7 | ₹2,54,224 | No action needed. Keep for the audit trail. |
| Short settlement | 5 | ₹1,11,106 | Recover the shortfall. It is usually a bank fee. |

The short settlements are worth a second look. Five groups worth ₹1,11,106 came up short by a
combined ₹1,608 — under 1% each. That is what an undeclared bank fee looks like. We deliberately do
**not** round these away into "matched". Money is missing, so it stays visible until someone explains
it.

## Checking the inputs before trusting them

Before any matching happens, the notebook runs 46 checks across the four files: can each file be
read, are the columns we need there, are the dates and amounts real values, is any row missing its
id, are the ids unique, is everything in rupees, is every ledger row successful, is every bank row a
credit.

Result: **0 errors and 14 warnings**. All 14 warnings are the same thing — a reference number used by
two transactions — and every one turned out to be an expected sale-and-reversal pair. Nothing was
quietly dropped; all 46 results are written to `input_validation_report.csv`.

## Assumptions we made

1. A difference of **₹1 or less** counts as matched. That is rounding, not a real break.
2. Bank fees are added back **only when the bank says so in the narration** (`NET OF CHGS 330.26`).
   A shortfall the bank did not declare is reported, never absorbed.
3. Month end means the calendar month end — 31 May and 30 June — not the last date on the statement.
4. Every row in the ledger is marked SUCCESS, so all of them are expected to be paid. The check for
   unsuccessful rows is written and runs; it just has nothing to find in this data.
5. The merchant code inside a plain `UPI/SETTLE/...` narration is **not** used for matching. It
   disagrees with our own merchant id in places, so we only trust the reference number.
