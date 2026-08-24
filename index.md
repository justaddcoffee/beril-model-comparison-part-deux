---
title: "Open model pilot benchmark"
description: "SUPERSEDED — BERIL benchmark pilot — Opus 4.8 vs Kimi K3 vs GLM 5.2, pilot questions on the live lakehouse"
---

> ## ⚠️ Superseded — and its headline numbers were wrong
>
> **This page is obsolete.** It was a 20-question pilot whose real purpose was proving
> the plumbing worked: three arms running concurrently, per-question pairing, honest
> grading. Only **13 questions** were machine-scoreable.
>
> It has since been superseded by a full run of all **174 questions × 3 arms**, and two
> problems with the pilot came to light:
>
> 1. **The checkout was contaminated.** The benchmark questions were written from prior
>    BERIL research that ships in the repo, and the agents were reading it. On an
>    unpruned checkout, **90 of 96** scoreable answers are findable on disk.
> 2. **n = 13 was far too small** to separate the arms.
>
> On the full run, with contaminated questions screened out, the three arms score
> **64.8% / 64.8% / 64.6% — indistinguishable.** In particular the "GLM 5.2 is the weak
> arm" conclusion below does **not** survive: GLM matched the others, and cost about a
> tenth as much.
>
> Those results are held privately with the BERIL team, because the write-up quotes
> benchmark questions and disputed answer-key values. Ask Justin for access.
>
> *Kept online for provenance only. Do not quote the numbers below.*

---

**2026-08-21 · pilot questions × 3 arms on BERDL JupyterHub**
Part two of [beril-model-comparison](https://justaddcoffee.github.io/beril-model-comparison/).

## 1. Correctness — the main question

**Kimi K3 has not yet produced a wrong answer. Opus is perfect. GLM 5.2 is not.**

| arm | correct | wrong | never answered | accuracy when it answered |
|---|---|---|---|---|
| Claude Code + Opus 4.8 | 13 | 0 | 0 | **100%** (13/13) |
| omp + Kimi K3 | 12 | **0** | 1 | **100%** (12/12) |
| omp + GLM 5.2 | 9 | 3 | 1 | 75% (9/12) |

Denominator is the **13 machine-gradeable questions**: 20 minus 5 prose T4 items needing human
adjudication, minus 2 whose answer key is disputed. Grading checks the value against the key's
stated tolerance (`±2%`, `±1.0pp`, `±0.3`); where a question asks *which* group wins, the named
entity must also be right; and a percentage key matches on either the 0–100 or 0–1 scale.

Across this run and an earlier one, **Kimi K3 has answered 21 questions and got 21 right**. Its
failure mode is running out of time, never being wrong. GLM's is both.

## 2. Completion

Kimi and GLM each failed to answer 1 of 13. Both remaining failures are 1800 s timeouts, and
both cluster in the tiers that need long chains of work — by tier, Kimi is 4/0/1 (T1), 5/0/0
(T2), 4/1/0 (T3); GLM is 4/1/0, 4/1/0, 1/2/2 as correct/wrong/timeout. **GLM's weakness is
specifically T3 synthesis.** Counting the unscored prose tier, T4 defeated both arms more often
than any other — three of Kimi's timeouts and two of GLM's fall there, and one T4 question timed
out for both at the full cap.

## 3. GLM's wrong answers came under the wire

GLM's failures are mostly time-related, not reasoning-related, and the accuracy column above
overstates the difference in capability. Its wrong answers took **three times longer** than its
correct ones — mean 1,292 s against 411 s:

| question | tier | verdict | elapsed |
|---|---|---|---|
| AFMSA-T1-02 | T1 | wrong | 1,593 s — 88% of the cap |
| AFMSA-T2-01 | T2 | wrong | 1,742 s — 97% of the cap |
| ESSG-T3-01 | T3 | wrong | 540 s |

Two of the three were produced with the clock nearly out, which is the signature of an agent
finishing because it must rather than because it has converged. `AFMSA-T2-01` is the sharpest
case: Kimi answered it correctly in 269 s; GLM spent 1,742 s and got it wrong. Only `ESSG-T3-01`
looks like a clean reasoning failure — unhurried, and wrong on both the entity and the magnitude.

Counting the two outright timeouts, **four of GLM's five non-T4 failures are time-related**. It
grinds: 38–61 API calls and 41–76 tool calls on the questions it loses, against 26–36 on those it
wins. The counter-example is `ESSG-T3-02` — 713 s, 53 API calls, and correct — so slow does not
automatically mean wrong.

The open experiment is to re-run GLM's five failures at a 3600 s cap. If the two near-cap wrongs
and the two timeouts convert, the honest description is "GLM is slower, not less capable."

## 4. The first attempt measured the wrong thing

An earlier run of this same benchmark had Kimi failing 4 of 13 and GLM 7 of 13. That was almost
entirely an artifact of Azure quota, not model capability, and it is worth recording because the
failure was invisible until the traces were read.

The workshop Azure resource had been provisioned with:

| deployment | capacity |
|---|---|
| `claude-opus-4-8` | 2000 |
| `FW-Kimi-K3` | **100** |
| `FW-GLM-5.2` | **100** |

**Opus had 20× the throughput of the open models.** Running two questions at a time across two
omp arms put four concurrent large-context requests against a 100-unit deployment; the
transcripts contain 22 records reading `429 … exceeded rate limit. retry-after-ms=13784`.
Measured directly, both models answer in **1.3–5.9 s** at 40 K context, and **~90% of each API
cycle was spent before the first token** — waiting, not generating. Azure quotas are
per-deployment, so Opus never competed for the same budget and never timed out.

Raising the two deployments to 1000 and 750 removed it: the same four concurrent requests that
previously returned three `429`s now return four `200`s in 2.6–9.1 s, and timeouts fell from
**7 and 9 to 1 and 1**. Everything in §1 and §2 is from the run after that fix.

The residual timeouts are the interesting part: they are concentrated in T4 and did **not**
disappear with the quota, so that share is genuine difficulty rather than throttling.

## 5. What this cannot tell you

- **n = 13 scored.** Opus and Kimi are one question apart; nothing here separates them.
- **T4 is unscored** — 5 prose questions per arm await human adjudication.
- **Harness is confounded with model.** Opus ran under Claude Code, the open models under omp.
  Nothing separates "Opus is better" from "Claude Code is better."
- **Opus's numbers predate the quota fix.** It was never throttled, but its run happened under
  different cluster load, so its timings are not strictly comparable.
- **The grader accepts a match on any number in the final answer line**, so verbose answers get
  more chances than terse ones. No confirmed false pass, but the exposure is real.
- **87 of the benchmark's 174 answers appear verbatim in public repo documents** the agent can
  read. No arm was observed exploiting it.

## 6. How the traces were analysed

Timings were parsed twice by independent code. The second pass used the omp transcript reader
from [evalome](https://github.com/justaddcoffee/coscientist-bench), which read every transcript
with **zero malformed lines**, recovered the model ids from the records rather than being told
them, and reproduced the per-step figures to within ~7%. Agreement between two parsers is why
the timing claims are stated as measurements.

evalome's `collect` pipeline was not usable: it grades BERIL research projects and needs a
directory holding a plan, notebooks and a report. A benchmark run is one-shot question answering
with no such directory, so the reader was used as a library. Running the full pipeline on
benchmark traces would need a new adapter beside the existing `beril` and `koros` ones.
