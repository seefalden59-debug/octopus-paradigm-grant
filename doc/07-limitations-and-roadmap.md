# 07 · Limitations and roadmap

Author: Mahmoud Al-Nouri · © 2026 · Tags: ✅ implemented · 🔶 designed · ⛔ withdrawn

## 1. Limitations (read these first)

| # | Limitation | Consequence | Mitigation path |
|---|---|---|---|
| L1 | **No production signer.** Only a development stub exists. | Nothing here can guard real funds yet. | R1 |
| L2 | **Trust-domain dependence.** The ceiling holds only if every signature goes through the guard (04 §5a). | A direct path to the signer removes the bound. | R1, deployment review |
| L3 | **In-memory state.** A restart clears windows; several instances multiply the ceiling. | Crash loops and scale-out weaken the bound. | R2 |
| L4 | **Shared-state backend unvalidated.** The Redis/Lua path is written but untested against a live server. | Do not use for multi-instance until validated. | R3 |
| L5 | **Controlled denial of service.** Reaching a pod or global ceiling pauses that scope. | Availability cost during attacks and mis-sized ceilings. | Sizing (04 §4), runbooks, R4 |
| L6 | **Strict mode halves steady capacity** per bucket. | Legitimate peaks must fit in `⌊C/2⌋`. | Sizing; finer-grained windows (R9) |
| L7 | **Held withdrawals have no review workflow.** | Held orders are recorded, not released. | R5 |
| L8 | **Single admin secret.** One operator credential can reset a breaker. | Insider risk (A5). | R6 |
| L9 | **Audit is tamper-evident, not immutable.** | A host-level attacker can truncate the tail. | External anchoring (05 §6), R7 |
| L10 | **Memory hygiene is best effort.** `SecretBuffer` zeroes only its own buffer. | Copies elsewhere are not covered. | Keep keys in the HSM/TEE |
| L11 | **Single-process throughput.** ≈ 17–21 k decisions/s in-process; a slow remote signer would block the decision lock. | Needs a queue for remote signing. | R8 |
| L12 | **Single asset unit.** | Multi-asset valuation needs a trusted price source. | R10 |
| L13 | **Not independently audited; no formal verification.** | Treat as a research prototype. | R11, R12 |
| L14 | **Shadow reports are only as good as the mirrored traffic.** Attack traffic inflates demand; shadow mode says nothing about signer lock-down. | Mis-sized ceilings if outliers are not reviewed. | Manual review (05 §5), R17 |
| L15 | **Pods are not yet independent control planes.** They have separate ceilings and breakers, but share one guard, one API secret and one signer. | Pod compartmentalisation does not yet reduce loss beyond the global ceiling. | R18 |
| L16 | **Skew tolerance costs capacity.** With several workers the per-bucket cap is `C/3` (not `C/2`), and skew above `W/2` needs a still smaller cap. | Legitimate peaks must fit in `⌊C/k⌋` per bucket. | Sizing; R9 (finer buckets) |

## 2. Roadmap

Priorities: **P0** blocks any real use · **P1** needed for a pilot · **P2** strengthens the case.

| ID | Item | Priority | Output / acceptance |
|---|---|---|---|
| R1 | **Real `Signer` integration** (HSM, TEE or audited MPC/TSS) with mutual authentication; signer accepts only guard approvals | P0 | Direct-to-signer request is refused in an automated test |
| R2 | **Restart safety**: persistent shared state, or a start-up hold of one window | P0 | Crash-loop test cannot reset a ceiling |
| R3 | **Validate Redis backend** against a live server, including failure injection | P0/P1 | `tests/test_limiter_redis.py` passes; fail-closed on outage verified |
| R4 | ~~Shadow mode~~ ✅ **done** (`SHADOW_MODE`, `/v1/admin/shadow`, `scripts/shadow_report.py`) | done | Roll-out phase 1 (05 §4) can run |
| R17 | Shadow-report extensions: destination-novelty and per-account statistics, outlier filtering, replay against alternative ceilings | P2 | Sizing recommendations that attack traffic cannot inflate |
| R18 | **Independent pods**: per-pod signer/HSM quorum, per-pod API and admin secrets, per-pod audit stream | P1 | Compromising one pod's credentials cannot approve withdrawals in another (automated test) |
| R19 | **EVM reference signer adapter** built on an audited library, with an end-to-end demo on public test networks (BNB Smart Chain testnet, an Ethereum testnet); testnet only | P1 | Reproducible demo: guard approves within the ceiling, refuses beyond it, trips; documented limits |
| R20 | Governance files: `LICENSE`, `SECURITY.md` (disclosure policy), contribution guide | P0 for any public release | Present and reviewed |
| R5 | **Manual-review workflow** for held orders; release passes through the same ceilings | P1 | Release cannot exceed the window bound (04 §5g) |
| R6 | **Dual control** for admin actions (two approvers) | P1 | Reset needs two distinct credentials |
| R7 | **Audit anchoring** job and a verifier tool | P1 | Truncation detected within one anchoring period |
| R8 | **Queue-based signing** for slow remote signers | P1 | Decision lock is never held across a network call |
| R9 | Finer-grained windows (N sub-buckets) to recover steady capacity while keeping the bound | P2 | Proof extended; property tests |
| R10 | Multi-asset valuation with trusted pricing and staleness limits | P2 | Ceiling expressed in a reference unit, price manipulation analysed |
| R11 | **Verification of the limiter**: randomised property tests and a fuzzer ✅ done (`scripts/fuzz_limiter.py`, found the clock-skew flaw); **model checking (e.g. TLA+) 🔶 still open** | P2 | Window bound checked exhaustively for small parameters |
| R12 | **Independent security review** and live load test including network and TLS | P1 before production | Findings closed; end-to-end latency published with hardware |
| R13 | Metrics export and dashboards (trips, refusals, auth failures, queue depth) | P1 | SOC alerts defined |
| R14 | **Statistical advisory model**: μ, σ based alerts and ceiling-sizing reports; never changes limits | P2 | Alert precision/recall measured on replayed traffic |
| R15 | **Tripwires first, decoys second**: honey accounts/addresses that raise high-confidence alerts and trip containment; decoy replies only as opt-in research | P2 | Detection rate and false-alarm rate measured; distinguishability of decoys studied on-chain |
| R16 | Edge filtering (eBPF/XDP) and layer-7 flood protection in front of the guard | P2 | Flood test against the per-IP limiter and edge |

## 3. Earlier "Phase VII" items, re-mapped

| Earlier item | Now |
|---|---|
| Monotonic time, clock-skew rejection | ✅ monotonic-anchored engine clock; ±30 s client window plus nonce (cross-pod millisecond timestamps are not used) |
| eBPF/XDP, tarpit quotas | 🔶 R16 (tarpits consume the defender's resources and are not planned) |
| Polymorphic decoys | 🔶 R15 (research) |
| MPC re-keying ceremony, attestation | 🔶 belongs to the signer domain (R1); the guard's recovery is an operator reset (R6) |

## 4. Open research questions

1. How much capacity does strict mode cost against finer-grained bounded windows (R9)?
2. What is the right ceiling for a given loss-tolerance and withdrawal profile, and how should it be
   re-estimated without becoming attacker-controllable?
3. Can a breaker be made less of a denial-of-service lever (graduated throttling, per-pod fairness)
   without weakening the window bound?
4. How distinguishable are decoy responses on a public chain, and does deception add value beyond the bound?
5. What additional evidence (formal model, independent audit, pilot data) would a custodian or insurer
   need to take the bound into their risk model?

## 5. Related-work map (to be completed with verified citations)

Topics to survey before claiming novelty. No references are listed here on purpose: each must be found
and checked by the authors.

* Rate-limiting algorithms (token bucket, leaky bucket, fixed/sliding window counters, window-boundary error)
* Circuit breakers and bulkheads in distributed systems
* Withdrawal velocity limits and policy engines in institutional custody (commercial products)
* MPC/TSS wallet infrastructure and signing policies
* Tamper-evident logging (hash chains, transparency logs)
* Honeypots and deception technology; hot/cold wallet architectures; post-incident analyses of exchange hacks
