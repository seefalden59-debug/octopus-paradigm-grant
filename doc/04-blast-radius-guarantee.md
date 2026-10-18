# 04 · The blast-radius guarantee

Author: Mahmoud Al-Nouri · © 2026 · Status: ✅ implemented and tested (strict mode)

This document states precisely what is guaranteed, proves it, lists the conditions it depends on,
and says what breaks it. Read §5 before quoting any number.

## 1. Definitions

| Symbol | Meaning | Default |
|---|---|---|
| `B` | Hot-wallet balance, integer minor units | configured |
| `g` | Global ceiling in basis points of `B` | 300 (3 %) |
| `W` | Window length | 3600 s |
| `C` | Ceiling: `C = ⌊B·g / 10 000⌋` | 3 % of `B` |
| bucket `Tᵢ` | Aligned interval `[iW, (i+1)W)` | |
| `δ` | Worst clock difference between the workers that serve requests (`WINDOW_SKEW_TOLERANCE_MS`) | 0 (one clock) |
| `k` | Bucket divisor: `k = 2 + ⌈2δ / W⌉` | 2 with one clock; **3** for `0 < δ ≤ W/2` |
| `A(I)` | Total value of **approved** (signed) withdrawals in interval `I`. Held, refused and decoy responses are not counted. | |

**Strict mode** (`STRICT_WINDOW=true`, the default): a request of value `a` arriving in bucket `Tᵢ` is
approved only if `Sᵢ + a ≤ ⌊C/k⌋`, where `Sᵢ` is the approved total already in that bucket. The same
rule is applied to the pod and account ceilings. Each is an *additional* restriction; none relaxes the
global one.

## 2. Theorem (window bound)

> For every interval `I` of true length `W`, `A(I) ≤ C`, provided every worker's clock is within `δ` of true time
> and the limiter is configured with that `δ`.

**Proof.** The limiter places a request in the bucket of the time its worker believes. If the clocks are
within `δ`, the requests approved inside a true interval of length `W` carry believed times that all fit in an
interval of length at most `W + 2δ`. Such an interval intersects at most `⌈(W + 2δ)/W⌉ + 1 = 2 + ⌈2δ/W⌉ = k`
aligned buckets. Each bucket holds at most `⌊C/k⌋`. Hence `A(I) ≤ k·⌊C/k⌋ ≤ C`. ∎
With a single clock `δ = 0`, `k = 2`, and the argument reduces to "an interval of length `W` touches two buckets".

**Corollary 1 (sustained drain rate).** Over `n` consecutive buckets the total is at most `n·⌊C/k⌋`.
To drain the whole balance an attacker therefore needs at least `B / ⌊C/k⌋ ≈ k·10⁴ / g` windows.
With the defaults (`g = 300`, `W = 1 h`, `k = 2`) that is **at least about 66 hours** of uninterrupted,
un-tripped abuse (about 100 hours with `k = 3`), which is the time budget defenders get before the wallet
could be emptied through this path.

**Corollary 2 (breaker).** Once a request is refused by the global or a pod ceiling, that scope signs
nothing until an operator resets it. Loss until the reset is at most what was admitted before the trip
(≤ `C`). Each reset can re-arm at most one more ceiling's worth.

## 3. Why the "sliding-window counter" is not used for ceilings

The common O(1) approximation estimates window usage as
`previous·(W − elapsed)/W + current`. It is smooth and fine for *rate limiting*, but an attacker who
chooses timing can move almost `2C` inside one window: burst `C` at the end of one bucket, then
`≈ 0.9C` shortly after the boundary. **Measured** in this repository: 190 units against a limit of 100
(`test_sliding_mode_overshoots...`). The strict rule above gives up some steady-state capacity
(`⌊C/2⌋` per bucket) in exchange for a bound that holds against any timing
(`test_strict_mode_never_exceeds_the_limit_in_any_window`).
The per-IP request limiter keeps the smooth mode because it is a rate limit, not a loss ceiling.

## 3b. Clock skew between workers (found by fuzzing, then fixed)

The first version of the strict rule assumed one clock (`k = 2`). Random adversarial testing showed that
**any** clock difference between the workers serving requests breaks it, because an interval of true length
`W` can then touch three buckets, and three half-ceilings make 1.5 `C`:

| Skew between workers | `k = 2` (one-clock rule) | `k = 3` (`WINDOW_SKEW_TOLERANCE_MS` set) |
|---|---|---|
| 0 ms | ≤ 1.00 × ceiling | ≤ 0.67 × ceiling (the cost of the smaller cap) |
| 50 ms to 30 s | **exceeds** the ceiling in about 3 % of random sequences, up to **1.50 ×** | never exceeds it (0 of 50 000 sequences) |

(Measured with `scripts/fuzz_limiter.py`, 50 000 sequences of 200 attempts, skews 50 ms, 1 s, 10 s and 30 s.)
So with more than one worker, set `WINDOW_SKEW_TOLERANCE_MS` to at least the worst clock difference
(`k = 3` covers any `δ ≤ W/2`; larger skews need larger `k`, and the configuration computes it).
The Redis backend **refuses to start** without it. The price: steady capacity per bucket falls from `C/2` to `C/3`.

## 4. Sizing the ceiling

Strict mode halves the usable steady capacity, so size against the *legitimate* peak:

```
legitimate peak outflow per bucket  ≤  ⌊C/k⌋   (otherwise honest users are refused, and a
                                               pod/global refusal trips the breaker)
```

Historical mean and deviation of withdrawals (μ, σ) are the right input for choosing `g` and `W`,
reviewed by people. They are not a run-time input, because anything the attacker can influence at run
time can be inflated by the attacker.

| Wallet `B` | `g` | Ceiling `C` per window | Sustained maximum | Minimum time to drain `B` (`W` = 1 h) |
|---|---|---|---|---|
| 100 M | 300 (3 %) | 3.0 M | 1.5 M / window | ≈ 66 h |
| 100 M | 100 (1 %) | 1.0 M | 0.5 M / window | ≈ 200 h |
| 100 M | 50 (0.5 %) | 0.5 M | 0.25 M / window | ≈ 400 h |

(Units are illustrative. The table assumes no breaker reset and no bypass.)

**Measuring the legitimate peak.** Shadow mode (05 §4) records the highest per-window demand on mirrored
traffic and prints the minimum `g` that covers it. `/v1/admin/status` reports the *enforced* quantity: in
strict mode the current-bucket total against the per-bucket cap `⌊C/k⌋` (`window.global_bucket_cap`,
`global_headroom`), not a smoothed estimate.

## 5. Conditions, and what breaks the guarantee

The theorem is about **approved (signed) value**. It holds only if all of the following hold:

| # | Condition | If violated |
|---|---|---|
| a | **Every** signature is issued through the guard; the signer refuses anything else | A direct path to the signer removes the bound entirely. Deployment, not code, enforces this. |
| b | **One authoritative counter.** One process, or one shared store with atomic updates | `N` independent instances each allow `C`, so the real ceiling is `N·C`. Partition the budget (`ΣCᵢ ≤ C`) or share the counter. |
| c | **State survives** or the guard starts safe | The in-memory state resets on restart, so a crash loop is a ceiling reset. Use a persistent shared store, or hold withdrawals for one window after start 🔶. |
| d | **Clock error is within the configured `δ`.** | One process: the engine's clock is anchored to the monotonic clock and the limiter never lowers a bucket index (both tested). Several workers: set `WINDOW_SKEW_TOLERANCE_MS ≥ δ` (§3b); without it the bound can reach 1.5 `C`. Skew larger than `W/2` needs a larger `k`, which the configuration derives. |
| e | **The guard is not itself compromised** | Out of scope (A4/A6). |
| f | **Amounts are in one trusted unit** | Multi-asset deployments need a trusted valuation; a manipulated price widens the real loss. |
| g | **Release of held withdrawals consumes capacity** 🔶 | The manual-review workflow is not built; when it is, a release must pass through the same ceilings. |
| h | **Pod ceilings assume independent compromise.** | If pods share an authorization path or signer, only the global ceiling bounds the loss (03 §4). |

## 6. The price of the guarantee

* **Controlled denial of service.** An adversary able to push traffic up to a pod or global ceiling can
  force a trip. The ceiling is the maximum loss; the pause is the cost. This is the intended failure mode
  for custody, but operators need a runbook and monitoring (05 §6).
* **Reduced steady capacity** in strict mode (half the ceiling per bucket).
* **Capacity consumed by an approval whose signing then fails is not refunded.** The ceiling can then
  only be reached sooner, never later.

## 7. How the guarantee is tested

| Claim | Test (`tests/`) |
|---|---|
| Any window ≤ ceiling under greedy adversarial timing | `WindowGuaranteeTests.test_strict_mode_never_exceeds...` |
| Boundary burst cannot double the limit | `...boundary_burst_cannot_double_the_limit` |
| Smooth mode *can* overshoot (documented weakness) | `...sliding_mode_overshoots...` |
| Clock stepping back does not reset usage | `...clock_stepping_backwards...`, `DefaultClock` |
| Refusals consume nothing; all-or-nothing across scopes | `LimiterTests` |
| **Randomised adversarial sequences**: bound holds for every seed (one clock; skewed workers with tolerance) | `tests/test_properties.py`, `scripts/fuzz_limiter.py` (P1, P3) |
| Decisions equal an independent oracle across several scopes | fuzz property P2 |
| Documented flaws are still found (the fuzzer keeps its teeth) | fuzz properties P4, P5; `test_the_checker_catches_a_limiter_without_the_per_bucket_cap` |
| Random traffic through the whole engine never signs more than any ceiling; nothing approved after a trip | `EngineProperties` |
| End to end: attacker below every threshold takes ≤ `C`, ends in containment | `CompromisedBackendScenario` |
