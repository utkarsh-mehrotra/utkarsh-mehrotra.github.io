# Independent Work: Staff-Grade Fintech Project Specs

Private working notes (not deployed). Each project must clear the **staff bar** below before its
"In progress" label comes off the portfolio.

## The staff bar (applies to every project)

What separates a staff-level project from a senior one is not size. It is a **hard, named problem**, a
**precise failure model**, and **evidence** that the design holds under that model.

| Requirement | Concretely |
|---|---|
| A named hard problem | One sentence a staff interviewer recognises as hard (e.g. "hot-account contention under strict serializability"). Not "a payment system". |
| Failure model written down | What can fail (crash, partition, duplicate, reorder, clock skew, partial write) and what the system guarantees under each. |
| Correctness evidence | At least one of: a TLA+/P model, deterministic simulation testing (DST) with fault injection, or a linearizability/serializability checker (Porcupine, Elle). Property-based tests for invariants. |
| Performance evidence | A reproducible benchmark with a stated workload (skew, mix, size), hardware, p50/p99/p99.9 and throughput, and a "knee" chart. Numbers you can defend line by line. |
| Design doc + decision log | Alternatives considered and why rejected. This is what an interviewer actually reads. |
| A write-up | One long-form article per project on the portfolio, in the same format as the existing three. |
| Scope discipline | A small core done rigorously beats a broad system done loosely. No CRUD, no UI work, no "microservices for the sake of it". |

Signals to avoid: inflated READMEs ("planet-scale", "FAANG-level"), unmeasured latency badges, LLD
interview exercises, committed `venv/`, emoji-heavy READMEs.

---

## 1. Bitemporal loan servicing engine  ·  *Lending (ties to Samsung)*

**Hard problem.** Retroactive corrections. A payment dated 3 March arrives on 20 March (late
settlement file, reversal, or a back-valued rate change). Interest accrual, late fees, delinquency
buckets and every downstream event since 3 March must be recomputed, deterministically. The old view
must stay reproducible for audit ("what did we believe on 15 March?").

**Design core.**
- Event-sourced loan ledger with **two time axes**: *valid time* (when it happened financially) and
  *transaction time* (when we learned it). Queries are `as_of(valid) × as_at(known)`.
- Pure, deterministic servicing function: `state = fold(schedule, events ≤ valid_t known ≤ known_t)`.
  Replay is the recompute mechanism, so incremental recompute from the earliest affected point is the
  optimisation, verified against full replay.
- Correction propagation emits **compensating events** downstream (fees reversed, bucket changed) with
  idempotency keys derived from `(loan, valid_t, correction_id)`.
- Day-count conventions (ACT/365, 30/360), rounding policy with `BigDecimal`/fixed-point, and
  holiday calendars per market as versioned config.

**Evidence to produce.**
- Property tests: for any event permutation with the same valid times, final state is identical;
  incremental recompute == full replay; Σ(compensations) reconciles to the delta.
- Bitemporal query benchmark: recompute cost vs. correction depth (1 day → 2 years back) on 1M loans.
- Write-up: "Backdated payments are the hardest problem in lending software."

**Not in scope.** Origination, KYC, UI.

**Effort.** 6–8 weeks part-time.

---

## 2. High-contention double-entry ledger with deterministic simulation testing  ·  *Payments core*

**Hard problem.** Strictly serializable transfers when a small set of accounts (fee pool, settlement
account, a merchant's wallet on sale day) receives most of the traffic. Naive row locking collapses
throughput under Zipfian skew, and sharding breaks atomic multi-account transfers.

**Design core.**
- Single-writer-per-partition state machine with **batched commit** (group commit of transfers into
  one log write), so hot accounts are a CPU problem, not a lock problem.
- Cross-partition transfers via a deterministic 2PC on the replicated log, or a documented
  restriction plus a routing layer. Pick one and defend it in the decision log.
- Invariants: Σ debits = Σ credits always; balances never below configured limits; every transfer
  exactly once under client retries (idempotency keys with bounded retention).
- **Deterministic simulation testing**: the whole system (network, disk, clock) runs on a seeded
  simulator. Inject crashes, torn writes, packet loss and reordering; any failing seed is a perfect
  repro. This is what makes it staff-grade, not the ledger itself.

**Evidence to produce.**
- Elle/Porcupine-checked histories under fault injection; millions of simulated seeds per CI run.
- Throughput vs. skew curve (uniform → Zipf 0.99) comparing row-lock baseline vs. batched commit.
- TLA+ model of the transfer + idempotency protocol (small, model-checked).
- Write-up: "Hot accounts: why most ledgers fall over on sale day."

**Not in scope.** Multi-currency FX, a query API beyond balances and history.

**Effort.** 8–10 weeks part-time. Highest ceiling of the set.

---

## 3. Point-in-time correct streaming fraud features, active-active  ·  *Fraud (ties to WU and the 50K article)*

**Hard problem.** Make the "What breaks at 50K events/sec" thought experiment real, at a smaller
scale, on the two hardest parts:
1. **Training/serving parity.** Features computed online must equal features recomputed offline for
   the same event at the same instant, including late and out-of-order events.
2. **Velocity counters across two regions** without cross-region coordination on the hot path
   (the CRDT design from the article: G-counters + PN-counters for window expiry), with a bounded,
   measured error.

**Design core.**
- Event-time windows with watermarks in Flink (or a small custom stream processor). Each feature
  value is stamped with the watermark it was computed under.
- Offline backfill with the same feature definitions (one DSL, two executors), and a **parity
  checker** that replays production-shaped traffic and diffs online vs. offline per event.
- Two-region deployment (two local clusters plus a simulated WAN with latency and partitions);
  measure counter divergence and convergence time under partition.

**Evidence to produce.**
- Parity rate (target: 100% for on-time events; documented, bounded deviation for late events).
- Counter error vs. partition duration chart; p99 feature-lookup latency at sustained load.
- Write-up: the "built it" follow-up to the 50K thought-experiment article.

**Not in scope.** The ML model itself (use a fixed rule or logistic regression as a stand-in).

**Effort.** 6–8 weeks part-time.

---

## 4. Deterministic backtesting engine (existing, `nse-engine`)  ·  *Trading*

Already the strongest repo: 15.7K LOC, 521 passing tests, byte-for-byte reproducibility, written
decisions. To make it clearly staff-grade:
- **Market-impact and fill realism.** Replace close-based fills with a calibrated slippage/impact
  model, and quantify how much backtest P&L each realism layer removes (this is the interesting result).
- **Leakage proofs.** A look-ahead probe that mutates future data and asserts zero change in past
  decisions, run on every strategy (generalise the existing probe).
- **Performance.** Benchmark: N strategies × M years of F&O chains, wall time and memory; profile and
  publish the two biggest wins.
- **Write-up.** "Why most backtests lie, and how to make one that can't."
- Clean up before linking anywhere: README tone, remove `CLAUDE.md`/`PROGRESS.md` from the public tree
  or rewrite them as a design log.

**Effort.** 3–4 weeks part-time.

---

## Suggested order

1. **Backtesting engine polish (4)**: closest to done; first item that can lose the "In progress" tag.
2. **Bitemporal loan servicing (1)**: strongest tie to your current role; great interview story.
3. **Ledger with DST (2)**: highest ceiling; start once 1 is in write-up stage.
4. **Fraud features (3)**: pairs with the existing article; optional if time is short.

Keep **at most three** on the portfolio at once. Two done well beats four "in progress".
