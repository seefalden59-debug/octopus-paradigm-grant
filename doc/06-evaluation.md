# 06 · Evaluation

Author: Mahmoud Al-Nouri · © 2026 · Everything here can be re-run from the private repository (reviewer access on request):
`python -m unittest discover -s tests -t .`, `python -m scripts.benchmark` and `python -m scripts.fuzz_limiter`.

## 1. Test suite

106 tests (reference run: 88 executed, 18 skipped because the sandbox had no FastAPI/httpx/Redis).
The skipped ones are written and need `pip install -r requirements.in httpx` (16 API tests) or
`REDIS_TEST_URL` (2 Redis contract tests). **Run them before relying on those components.**

| Area | Tests | What they establish |
|---|---|---|
| **Property-based / fuzz** | 9 | Strict bound for every random seed; independent-oracle equivalence; skewed workers with and without tolerance; documented flaws still found; checker catches a deliberately broken limiter; whole-engine random traffic never signs more than any ceiling |
| Window limiter | 8 + 5 | Amount-weighted; refusals cost nothing; all-or-nothing across scopes; fails closed when full; strict bound under greedy timing; clock stepping back does not reset usage |
| Request authentication | 8 | Valid, replay, tampered body, wrong secret, skew both directions, malformed headers, failed signature does not burn a nonce |
| Audit chain | 3 + 1 | Tampering and deletion detected; chain continues across restarts; broken file refused |
| Network trust | 7 | `X-Forwarded-For` ignored from untrusted peers; spoofed hops skipped; IPv6 grouped; most specific pod wins |
| Secret buffer | 3 | Wipe zeroes the live buffer; context manager; `repr` does not leak |
| Policy via engine | 8 | Sign / hold / deny; negative, zero, float, bool, string amounts refused; allowlist |
| Idempotency, limits, fail-closed | 2 + 4 + 3 | Retry consumes once; account ceiling does not trip breaker; pod ceiling trips only that pod; reset; audit or signer failure means refusal |
| **Compromised-backend scenario** | 4 | Allowlist stops it; large orders held; worst case (allowlist bypassed, every threshold respected, four pods) takes ≤ `C` and ends in containment; nothing more leaves after containment |
| **Shadow mode** | 16 | Signer never called, no signature ever returned, nothing refused or tripped; would-be verdicts match enforcement; ceilings simulated; demand measured independently of the simulated ceiling; retries not double-counted; report holds aggregates only; offline replay equals the live report and refuses an edited log; starts without signer or allowlist in production |
| Status accuracy | 3 | `/v1/admin/status` reports the *enforced* quantity (bucket total vs per-bucket cap in strict mode) |
| HTTP API (not run in reference) | 13 + 3 | 401/403/413/422/429 behaviour, no spoofing via headers, admin invisible without network config, admin secret separation; shadow responses always 200 and unsigned, report endpoint |

## 2. The window experiment (why strict mode exists)

| Experiment | Smooth window | Strict window |
|---|---|---|
| Burst of 100 at the end of a bucket, probe again 54 s later (limit 100) | **190** inside one 60 s span | ≤ 100 |
| Greedy attacker probing every 3 s for five windows, worst 60 s span (limit 100) | 110 | 60 |

The smooth estimator can be pushed close to twice the limit; the strict rule cannot exceed it
(04 §2–3). The smooth mode remains only for the per-IP request limiter.

### Fuzz campaign (`python -m scripts.fuzz_limiter --iterations 50000 --seed 100000`)

Each iteration is a random sequence of 200 attempts: amounts from tiny to the whole ceiling, times clustered at
bucket boundaries, ceilings of 10 to 1 000 001 (odd and tiny included). 50 000 sequences per property (10 million
attempts each), reproducible by seed:

| Property | Expectation | Result |
|---|---|---|
| P1 strict, one clock: any window ≤ ceiling | holds | **0** violations |
| P2 strict: decisions equal an independent oracle over several scopes | holds | **0** mismatches |
| P3 strict, skewed workers with tolerance configured | holds | **0** violations |
| P4 strict, skewed workers, **no** tolerance | known flaw | 1 685 violations, worst **1.50 ×** the ceiling |
| P5 smooth estimator | known flaw | 31 092 violations, worst **2.00 ×** the ceiling |

P4 was **discovered by this campaign**: the one-clock rule fails under any worker clock difference. It is fixed
by the skew-aware per-bucket cap (04 §3b), required for the Redis backend. Passing is evidence, not a proof; a
model-checked proof remains on the roadmap (R11).

## 3. Performance

`python -m scripts.benchmark`, small Linux VM, Python 3.12, x86-64, shared CPU, two runs:

| Quantity | Result |
|---|---|
| Decision (`engine.process`): policy + ceilings + hash-chained audit (in memory) + stub signature | p50 ≈ 42–45 µs · p95 ≈ 72–100 µs · p99 ≈ 90–160 µs |
| Tail | maxima in the tens of milliseconds (GC / scheduling noise), so **do not quote averages alone** |
| Throughput, one thread | ≈ 17 000–21 000 decisions/s |
| `SecretBuffer.wipe()` of 32 bytes | p50 ≈ 1.5 µs · p99 ≈ 3–5 µs |

**Scope of these numbers.** They cover the in-process decision only. They exclude network, TLS, JSON
parsing, a real HSM/MPC signer, file `fsync` and the Redis backend, and the wipe figure says nothing about
copies of a secret elsewhere in memory. End-to-end latency has **not** been measured and must be measured
in the target environment before any figure is published.

## 4. Claims register: what happened to the earlier statements

| Earlier statement (drafts 01–07) | Status | Note |
|---|---|---|
| O(1) rolling-window verification | ✅ | Integer bucket counters, O(1) per scope |
| Bound holds with several workers | ✅ found by fuzzing, fixed | The one-clock rule reached 1.5 × under skew; per-bucket cap `C/k` fixes it (04 §3b) |
| Blast radius ≤ 3 % | ✅ restated | Global ceiling `C = 3 % of B` in any window (proved, 04). The earlier "no pod holds more than 3 % of total" was arithmetically inconsistent with its own example (100/230 ≈ 43 %) and is withdrawn |
| `Cap = αμ + βσ + Δ` at run time | ⛔ → 🔶 | Replaced by a fixed fraction; the statistical model becomes offline sizing plus advisory alerts |
| `< 15 µs` lookup in a Go layer | ⛔ | No Go layer exists. Measured Python decision p50 is ≈ 42–45 µs |
| `0.7 ms` overhead | ⛔ | Never measured; no method was given |
| Passive "Shadow Mode" calibration phase (integration plan, phase 4) | ✅ | Implemented as `SHADOW_MODE` with live and offline reports (03 §7, 05 §5) |
| `< 15 µs` "time to containment" | ⛔ | A single lookup time is not a time-to-contain; withdrawn |
| Zero-residual memory wipe / "physical-level purge" | ⛔ | Only the wipe of an owned buffer is claimed, 1.5 µs for 32 B |
| `mlockall` | ✅ | Return value checked; needs `IPC_LOCK` in containers |
| Key shredding on detection ("autotomy") | ⛔ → ✅ | Replaced by the breaker (03 §3) |
| HTTP 202 decoy masking | ⛔ | 202 now means *pending review*; deception is an opt-in research switch |
| Decoys indistinguishable without the decoy key | ⛔ | Observers can verify signatures with the public key or look the hash up on-chain |
| Micro-VM / gVisor / decoy testnet / MEV-flag masking | 🔶 | Not built; "blast radius zero" language withdrawn |
| MPC quorum recovery, hardware attestation | 🔶 | Belongs to the signer domain |
| Immutable audit trail | ✅ restated | Tamper-*evident*; immutability needs external anchoring (05 §6) |
| `±50 ms` clock skew between pods, monotonic time | ✅ restated | Client timestamps ±30 s plus nonce; internal windows anchored to the monotonic clock |
| eBPF/XDP filtering, tarpits | 🔶 | Per-IP limiter exists in the application only |
| KYC level / behavioural trust scoring | 🔶 | Not built |
| Dual control for disabling the guard | 🔶 | Planned |
| 99.999 % uptime, `< 50/100 ms` failover | ⛔ | Not measured |
| Fail over to the primary MPC path | ⛔ | Fail-open; replaced by fail-closed |
| Zero refactoring / zero disruption in 10 weeks | ⛔ | Replaced by exit-criteria roll-out (05 §4) |
| Attack costs "years and millions" | ⛔ | Unsupported. Replaced by a measurable statement: minimum time to drain (04 Cor. 1) |
| Compliance and insurance "mapping" | ⛔ | No certification. Relevant *controls* are listed instead (audit chain, bounded exposure, fail-closed) |

## 5. Reproducing and extending

* Re-run tests and benchmark on your hardware and publish the machine description with every number.
* Property-based or model-checked verification of the limiter is the natural next step (07).
* An independent security review and a live load test (network included) are required before any
  production claim.
