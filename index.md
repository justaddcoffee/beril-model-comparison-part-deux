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
| omp + GLM 5.2 | 4 | 3 | 6 | **57%** (4/7) | 31% |

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

## 3. Why the open models ran out of time

**Azure rate limiting, not model speed** — and it was an artifact of how this pilot was run.

The transcripts contain 22 records reading `429 … exceeded rate limit. retry-after-ms=13784`.
Called directly, both models answer in **1.3–5.9 s** at 40 K context. Inside the run, **77% of
wall clock sat inside API request cycles and ~90% of each cycle was before the first token** —
waiting, not generating. Actual decode ran at 85–132 tok/s on 200–400-token outputs, so real
generation was 2–4 s per turn.

The cause was the harness: two questions in flight × two omp arms sent **four concurrent
large-context requests at one deployment**. Azure quotas are per-deployment, so Opus — on a
separate deployment — never competed for the same budget and never timed out. **The completion
gap in §2 therefore measures the test setup at least as much as it measures the models.** Two
or three timeouts were separately caused by slow lakehouse queries rather than throttling.

**Known defect:** one GLM run scored as a timeout actually kept running 17 minutes past the cap
and produced an answer — the harness failed to kill the detached process. That answer was wrong,
and the table above now counts it as wrong rather than unanswered. Re-running the open models
sequentially, one request per deployment at a time, is the experiment that would separate model
capability from quota contention.

## 4. How the traces were analysed

Timings above were parsed twice by independent code. The second pass used the omp transcript
reader from [evalome](https://github.com/justaddcoffee/coscientist-bench), which read all 16
transcripts with **zero malformed lines**, recovered the model ids from the records rather than
being told them, and reproduced the per-step figures to within ~7% of the first pass (38.9 s vs
36.2 s per step for GLM; 36.5 s vs 39.7 s for Kimi). Agreement between two parsers is why the
timing claims here are stated as measurements rather than estimates.

evalome's `collect` pipeline was **not** usable: it grades BERIL research projects and requires a
project directory holding a plan, notebooks and a report. A benchmark run is one-shot question
answering with no such directory, so the reader was used as a library instead. Running the full
pipeline on benchmark traces would need a new adapter alongside the existing `beril` and `koros`
ones.

Two defects in the trace set are worth recording for anyone reproducing this: two of the sixteen
files are byte-identical duplicates, so there are **14 unique traces, not 16**, and the run that
overshot its cap means concurrency during the pilot was higher than the harness intended.
