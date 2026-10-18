# 03 · Architecture

Author: Mahmoud Al-Nouri · © 2026 · Tags: ✅ implemented · 🔶 designed · ⛔ withdrawn

## 1. Request path

```
POST /v1/withdrawals
  │
  ├─ admit(ip)            per-IP rate limit (IPv6 grouped by /64)             api.py / engine.admit
  ├─ read body ≤ 4 KB     streamed size limit                                 api.py
  ├─ authenticate         HMAC, timestamp window, nonce (after signature OK)  auth.py
  ├─ validate             strict schema, integer amounts, unknown fields refused
  │
  └─ engine.process (one lock; atomic decision)                               engine.py
       1. idempotency     same tx_id + same content → same decision
       2. containment     tripped pod/global → refuse
       3. policy          amount sanity · allowlist · per-tx cap · review threshold   policy.py
       4. ceilings        global · pod · account, all-or-nothing                limiter.py
       5. audit           approval written to hash chain BEFORE signing         audit.py
       6. sign            Signer boundary (HSM / MPC; stub in development)      signer.py
```

Responses: `approved` (200, with signature), `pending_review` (202), `declined` (403), `rate_limited` (429),
`unauthorized` (401), `invalid_request` (422). No response states the reason for a refusal.

## 2. Modules

| Module | Responsibility |
|---|---|
| `config.py` | Environment-driven settings; refuses unsafe or inconsistent configuration (e.g. production without an allowlist). |
| `limiter.py` / `limiter_redis.py` | Value-weighted window counters (in-memory; Redis/Lua 🔶 unvalidated). |
| `policy.py` | Pure rules, trivially testable. |
| `containment.py` | Breaker per pod and global; opt-in decoy helpers. |
| `engine.py` | Orders the layers, owns idempotency, audit-before-sign, fail-closed behaviour. |
| `auth.py`, `netutil.py` | Request authentication; client address and pod resolution. |
| `audit.py` | Hash-chained log. |
| `signer.py` | `Signer` protocol; development stub only. |
| `shadow.py` | Shadow-mode statistics and offline replay of the audit log. |
| `memory_lock.py` | Core-dump disable, non-dumpable flag, `mlockall` (all verified and reported); wipeable secret buffer. |
| `api.py`, `main.py` | HTTP layer and entry point. |

## 3. Design decisions and rationale

| Decision | Replaces | Why |
|---|---|---|
| **Fixed-fraction ceiling** (basis points of the wallet) | Runtime formula `αμ + βσ + Δ` | A ceiling that grows with recent variance can be inflated by the attacker's own traffic, and a sentiment term is neither deterministic nor auditable. A fixed fraction gives a bound that does not move under attack. |
| **Breaker trip, reset by operator** | Wiping regional keys on detection | Wiping is a self-inflicted outage that a single misclassified request can cause, and in a managed runtime wiping cannot be guaranteed anyway (copies). The breaker pauses the same scope without destroying anything. |
| **Fail-closed on every internal fault** | "Fail over to the primary MPC path" | A fallback that bypasses the guard turns *crash the proxy* into *remove the ceiling*. Availability must come from replicas of the guard, not from skipping it. |
| **Identity from the connection** | `region` and `source_ip` read from the request body | Anything a client can write, a compromised client can forge. |
| **Audit before signature** | Best-effort asynchronous logging | If logging can fail silently, the strongest evidence disappears when it is needed most. |
| **Idempotent `tx_id`** | none | Retries are normal in payment systems; without idempotency a retry either double-spends capacity or gets a different answer. |
| **Opaque refusals** | HTTP 202 "accepted" masking | A distinct status code is itself a tell; and clients should not learn which control to evade. |
| **Signer behind an interface** | Hand-written MPC and "multisig" hashing | Cryptography is not re-implemented here. The guard decides *whether*; an audited signer decides *how*. |

## 4. Regions, pods and shared state

Geographic pods limit contagion between regions. Two ways to keep the *global* ceiling meaningful:

| Option | How | Cost |
|---|---|---|
| **Shared counter** | One authoritative store (Redis, atomic Lua script) holds all scopes | Regions depend on it; it becomes critical infrastructure |
| **Static budget partition** | Give pod *i* a ceiling `Cᵢ` with `ΣCᵢ ≤ C` | No cross-region dependency; unused budget in one region cannot be borrowed by another |

Running several guard instances **without either** multiplies the ceiling by the instance count
(each instance enforces its own `C`). See 04 §5. Several workers also need `WINDOW_SKEW_TOLERANCE_MS`
(04 §3b), because clocks differ and a one-clock window rule can then let through up to 1.5 `C`.

### Independence: what makes a pod a real compartment

Splitting the hot wallet into several vaults (the "ten safes, each with its own lock" idea) limits loss
only when the vaults are **independent control planes**. Geography helps operationally and legally, but the
property that matters is that compromising one pod gives no access to another:

| Axis | Independent when… | Today |
|---|---|---|
| Authorization path | each pod has its own order source and approval path | 🔶 one guard, one API secret |
| Signing keys | each pod has its own HSM / MPC quorum | 🔶 one `Signer` |
| Credentials | per-pod API and admin secrets | 🔶 shared |
| Operations | separate accounts, networks, on-call | deployment choice |
| Ceilings and breakers | per-pod budget, per-pod trip | ✅ implemented |

| Situation | Loss bound |
|---|---|
| Pods are independent; `k` of them are compromised | about `k × Cpod`, capped by the global ceiling `C` |
| Pods share a control path (today's deployment of this repository) | `C` (global ceiling only): everything falls together |

This is why the global ceiling exists: it covers the shared-control-path case that partitioning alone cannot.
Partitioning also has costs: liquidity is fragmented (each vault needs enough for its own withdrawals),
rebalancing between vaults is itself an attack path, and more keys mean more operations to get right.
Per-pod signers and credentials are roadmap item R18.

## 5. Statistical model 🔶 {#statistical-model}

The earlier drafts used historical mean μ and deviation σ to set the cap at run time. The project keeps
the idea, with a different role:

* **Offline sizing**: μ and σ of legitimate withdrawal volume help humans choose `g` and `W` so that
  `⌊C/2⌋` exceeds the legitimate peak per window (04 §4).
* **Advisory alerts** 🔶: deviation from the historical profile raises an alert to the SOC. It never
  changes a limit by itself.

## 6. Latency in context ✅

The guard sits on the **withdrawal authorization path**, not in the order-matching path, so it adds no
latency to matching. Measured in-process decision time (policy + ceilings + audit + stub signature, no
network) was about 40–45 µs at the median on a small VM (06 §3). Network hops, TLS and a real HSM/MPC
signer will dominate in production and have **not** been measured.

## 7. Shadow mode ✅

`SHADOW_MODE=true` runs the same policy and ceiling logic on mirrored traffic but changes what the guard is
*able* to do:

| Property | Guarantee (tested) |
|---|---|
| No signing | The signer is never called and no response carries a signature. It does not even need a signer to start. |
| No enforcement | Every request receives HTTP 200 `{"status":"shadow","would_be":...}`. Nothing is refused, held or tripped. |
| Faithful simulation | Approvals consume simulated capacity; refusals cost nothing; per-pod and global ceilings are tracked. |
| Would-be trips | Recorded once per scope per window instead of pausing anything. |
| Evidence | Every decision is written to the hash-chained audit log (including decision time), so the report can be rebuilt offline and an edited log is refused. |
| Demand measured independently | Requests that pass policy count as demand even if the simulated ceiling would refuse them; otherwise the ceiling would hide the demand it should be sized against. |

The report gives, per window, peak demand globally and per pod, and the *minimum* basis points that would
cover it (`2 × demand / balance` in strict mode, because strict mode admits half the ceiling per bucket), plus
the share of requests that enforcement would have declined or held, with reasons.
