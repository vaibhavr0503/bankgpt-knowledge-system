# BankGPT — Knowledge System That Knows What's True *Now*

My submission for the interface.ai (BankGPT) Software Engineer II build
assignment: a system that reads five snapshots of Dupaco Community Credit
Union's website, answers member questions with the **current**,
correctly-tiered value and a source, resolves conflicts, and says
"I don't know" when the site doesn't publish the answer.

- `solution.py` — the whole system (stdlib only, no API keys, no network)
- `NOTES.md` — design, source hierarchy, and measured scores

Public eval (kit scorer, 40 questions): **0.969**, up from the starter's
**0.093**. Harness self-test: **1.0**.

```bash
python harness/run_eval.py --submission . \
    --snapshots <kit>/snapshots \
    --eval <kit>/eval/public_eval.jsonl
```
