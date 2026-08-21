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

## 3. Cost per answer delivered

Per answer *actually produced* — the denominator that matters, since a timed-out question still
consumes the full budget:

| arm | s / answer | fresh tokens / answer |
|---|---|---|
| Opus 4.8 | **287** | **28k** |
| Kimi K3 | 1,876 (6.5×) | 110k (4.0×) |
| GLM 5.2 | 2,523 (8.8×) | 80k (2.9×) |

**"Fresh tokens" here means `input + output` only.** Opus additionally logged **79.3M cache
reads and 4.4M cache writes**, against ~17M cache reads for each omp arm. Counting cache writes
as billable work would raise Opus's per-answer figure from 28k to roughly 247k — so this
denominator flatters Opus, and the comparison should be read with that in mind. Azure returns no
per-call price, so these are token counts, not dollars.

This still inverts the earlier page's per-token framing: per *answer*, the open models are not
cheaper here.

## 4. Two answer-key entries are disputed

On `AFMSA-T3-01` all three arms independently returned **11.7%** against a key of **3.8**;
on `AFMSA-T3-02` both completing arms returned **~12.06** against **10.83**. The benchmark's own
appendix warns its figures are "point-in-time COUNTs over BERDL snapshots" from ~13 August, so
drift is plausible.

**I have not re-derived these values from the lakehouse**, so this is a flag, not a finding.
Cross-arm agreement is weaker evidence than it looks: all three arms share the same data, the
same repo, and similar query strategies, so they can converge on the same wrong answer.

Excluding both raises every arm — **GLM most (+16.7pp), then Opus (+13.3pp), then Kimi
(+10.0pp)** on conditional accuracy. Including them under the current key gives Opus 87%, Kimi
90%, GLM 50%. **These two keys should be re-derived before the benchmark is used again**, and the
same drift may affect questions no arm reached.

## 5. What this cannot tell you

- **n = 13 scored.** Opus and Kimi are indistinguishable here; the Opus-vs-GLM gap is larger but
  still small-sample. Nothing supports a ranking between Opus and Kimi on accuracy.
- **T4 is unscored** — 5 prose questions for Opus, but only 3 each for Kimi and GLM, which timed
  out on the other 2. The open models have *fewer* prose answers awaiting review, not the same.
- **Harness is confounded with model.** Opus ran under Claude Code, the others under omp.
  Nothing separates "Opus is better" from "Claude Code is better."
- **Pairing is partial** — all three arms ran concurrently for 11 of 20 questions; for the other
  9, Opus's runs came from an earlier attempt under different cluster load.
- **The grader accepts a match on any number in the final answer line**, so a verbose answer gets
  more chances than a terse one. No confirmed false pass was found in this data, but the exposure
  is real.
- **From run logs, not derivable here:** an omp `tools.format` misconfiguration crippled a first
  attempt and was fixed before this run; 87 of the benchmark's 174 answers appear verbatim in
  public repo documents the agent can read.

## Recommendation

For BERIL work today, Claude Code + Opus — it answered everything, got everything right, and did
so 6–9× faster.

But **Kimi K3 got every question right that it managed to finish**, and that is the result worth
following up. Whether a longer cap would convert its 4 timeouts into correct answers is *not
tested here* — they may be questions it would never finish, or would finish wrongly. That is a
cheap experiment: re-run those 4 with a 3600 s cap.

GLM 5.2 has both problems and is not currently a candidate.

Next: add omp + Opus to break the harness confound, re-derive the two disputed keys, adjudicate
T4 by hand, and re-run the timeouts at a longer cap to separate "cannot" from "not yet".