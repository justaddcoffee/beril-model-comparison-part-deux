---
title: "Open model pilot benchmark"
description: "BERIL benchmark pilot — Opus 4.8 vs Kimi K3 vs GLM 5.2, 20 questions on the live lakehouse"
---

**2026-08-21 · 20 questions × 3 arms × 60 runs on BERDL JupyterHub**
Follow-up to [beril-model-comparison](https://justaddcoffee.github.io/beril-model-comparison/).
*Every figure below was recomputed from the raw run log by two rounds of independent adversarial review; corrections are folded in.*

## 1. Correctness — the main question

**Every question Kimi K3 finished, it got right — the same as Opus. GLM 5.2 did not.**

| arm | correct | wrong | never answered | **accuracy when it answered** | of all 13 asked |
|---|---|---|---|---|---|
| Claude Code + Opus 4.8 | **13** | 0 | 0 | **100%** (13/13) | **100%** |
| omp + Kimi K3 | **9** | 0 | 4 | **100%** (9/9) | 69% |
| omp + GLM 5.2 | 4 | 2 | 7 | **67%** (4/6) | 31% |

Denominator is the **13 machine-gradeable questions**: 20 minus 5 prose T4 items needing human
adjudication, minus 2 whose answer key is disputed (§4). Grading checks the value against the
key's stated tolerance (`±2%`, `±1.0pp`, `±0.3`); where a question asks *which* group wins, the
named entity must also be right; and a percentage key is matched on either the 0–100 or 0–1
scale, since `0.689` and `68.9%` are the same answer.

So the honest split is:

- **Kimi's weakness is finishing, not reasoning.** 0 wrong answers in 9 attempts.
- **GLM's weakness is both.** It answered fewer questions *and* got a third of those wrong.
- **Across all questions asked** the ranking is 100% / 69% / 31%, because to a user an
  unanswered question is no more useful than a wrong one.

Note the conditional figures are biased *toward* the open models: timeouts remove the hardest
questions from their denominators, and Opus loses nothing that way. Kimi's 100% is over 9
questions, not 13.

## 2. Completion — why the two readings diverge

Kimi failed to answer **4 of 13**, GLM **7 of 13**, Opus **0**. Every failure is an 1800 s
timeout, not an error or refusal. Failures cluster in T2 computation and T3 synthesis rather
than T1 retrieval.
