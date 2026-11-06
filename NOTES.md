# BankGPT knowledge system — notes

## What this is
A knowledge system for Dupaco Community Credit Union that answers member
questions from five website snapshots (2026-05-12 → 2026-09-28), resolving
conflicts and saying "I don't know" when the site doesn't publish an answer.

## Files
- `solution.py` — the whole system. `build()` returns an object with
  `ingest(snapshot)`, `answer(question, as_of=None)`, `conflicts()`,
  exactly as `harness/contract.py` requires.
- No other files are needed. No `requirements.txt`: **Python standard
  library only** (`re`, `sys`, `pathlib`). No API keys, no network calls,
  no LLM — nothing to configure, nothing that can leak a key.

## How it works
1. **Ingest parses the rate sheet into a versioned fact store.** The
   markdown tables on `/rates` (and `/business-rates`) are parsed per
   snapshot into facts keyed by (product, term, tier, metric) — e.g.
   17-month special certificate APY, 30-year mortgage rate/APR by loan
   amount, auto APR by model-year tier and term, Share Savings APY by
   balance tier, credit-card APR ranges. Prose facts ("6-month
   certificate APY: 2.95%") are parsed only where no table exists.
   A fact is carried forward to a later snapshot only if its source
   page is missing from that snapshot (a crawl gap). If the source page
   is present but the row is gone, that is a real removal (e.g. the
   25-month special replaced by the 17-month special) and the fact is
   **not** resurrected.
2. **"Current" is snapshot-dated.** `answer(question, as_of=None)` uses
   the newest snapshot with `taken_on <= as_of` (newest overall when
   `as_of` is None; earliest when `as_of` predates the first snapshot).
   A date mentioned in the question ("as of May 2026") is honoured too.
3. **Source hierarchy.** The official rate sheet and product pages beat
   blog posts, press releases, promo landing pages and the frozen
   mortgage table on `/homebuying-guidance` (which still shows
   6.00%/6.037% for a 30-year mortgage). Expired promos (3.00%
   checking, 2.95% certificate promo) never beat the current sheet.
   `/business-rates` is parsed first and `/rates` overwrites shared
   keys, because the business page lagged (it still showed the old
   17-month special in Aug 2026).
4. **Answers are short and tier-exact.** Only the requested tier's
   value is returned — never a whole table. Repeating a stale or
   wrong-tier number zeroes a question in this scorer, so hedging by
   listing every number is deliberately avoided. Dividend rate and
   APY are both given when the question asks for the dividend rate.
5. **Honest refusals.** If the site does not publish the requested
   value — 7/1 ARM rate, construction-loan rate, 10-year mortgage,
   60-month jumbo certificate, a branch with no published hours
   (Mullan Avenue), a lost-debit-card fee — the answer is
   `confident=False` with no numbers in the text at all.
6. **`conflicts()`** is computed from the versioned fact store (values
   that changed across snapshots) plus cross-page disagreements found
   by re-scanning the ingested pages (stale guidance-page mortgage
   table, 3.00% checking on promo pages, the Traditional IRA 10-month
   jumbo row still showing 3.90% vs Roth 4.00%, branch count 22 vs
   23 vs 24, Forbes fourth vs fifth, Achieve promo 3.66% vs 3.25%
   after the intro period). Each conflict names its values, sources,
   the winning value and why.

## Scores (measured with the kit's own scorer)
- Starter solution: **0.093** final score on the 40 public questions.
- This solution: **0.969** final score (accuracy 0.969, consistency
  1.0, speed 1.0, p95 latency 0.0s, ingest 2.6s). The only public
  question scoring 0 is `dup-005` (certificate list), whose
  `must_include` list is empty, so the scorer gives it 0.0 for any
  answer — it is unwinnable under the published scoring code.
- Harness self-test (toy snapshots + conflicts gold): **1.0**.
- A 43-question paraphrase / as-of / new-topic probe I wrote myself
  (rewordings, May/June/Aug 2026 as-of dates, HELOC, home equity,
  motorcycle, IRA certificates, other branches, dividend-rate
  wording, removed products) passes 41/43; the other two are
  questions with no published answer, where refusal is correct.

## How to run
```bash
python harness/run_eval.py --submission <this folder> \
    --snapshots <kit>/snapshots \
    --eval <kit>/eval/public_eval.jsonl
```
Snapshots and the eval file are **not** included in this repository
(they are the supplied kit data, shared for this event only).
