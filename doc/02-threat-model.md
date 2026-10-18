# 02 · Threat model

Author: Mahmoud Al-Nouri · © 2026 · Tags: ✅ implemented · 🔶 designed · ⛔ withdrawn

## 1. System and trust domains

```
 Users ─► Exchange backend ─► [ GUARD ] ─► Signer (HSM / MPC) ─► Blockchain
              (assumed to be            │         accepts ONLY requests the
               compromisable)           │         guard approved
                                        └─ Admin API: separate secret + allowed network
```

The guard is useful only because it lives in a **different trust domain** from the backend it protects.
If an attacker who owns the backend can also reach the signer directly, or can modify the guard,
no ceiling holds. This is the single most important deployment requirement (see 05 §2).

## 2. Assets

| Asset | Why it matters |
|---|---|
| Hot-wallet balance `B` | The value an attacker wants. |
| Signing capability | Turns an order into a spendable transaction. |
| Guard secrets (API/admin HMAC) | Let a caller impersonate the backend or the operator. |
| Audit chain | Evidence for investigation, compliance and insurance. |

## 3. Adversaries

| ID | Adversary | Capability | In scope |
|---|---|---|---|
| A1 | External API abuser | Sends many or malformed requests, spoofs headers, replays captured requests | ✅ |
| **A2** | **Compromised backend** (the 2026-class incident) | Holds valid API credentials, can submit any authenticated request, forges account data, picks timing | ✅ **central case** |
| A3 | Network attacker | Observes or tampers with traffic between components | ✅ with TLS in front (05) |
| A4 | Host-level attacker on the guard machine | Reads memory, edits files, changes the clock, restarts the process | ◐ partly (clock, log tampering, restart: see 04 §5); full host compromise is out of scope |
| A5 | Malicious operator holding the admin secret | Can reset a tripped breaker | 🔶 dual control planned |
| A6 | Compromise of the signer / HSM | Signs arbitrary data | ❌ out of scope: protect it with its own controls |

## 4. Security properties

| ID | Property | Status |
|---|---|---|
| P1 | **Bounded outflow.** Approved value in any window of length `W` is at most `C = ⌊B·g/10⁴⌋` (default g = 300, i.e. 3 %). Proof and conditions in [04](04-blast-radius-guarantee.md). | ✅ |
| P2 | **No signature without an audit record**, written first. | ✅ |
| P3 | **Fail-closed.** Audit, signer or limiter failure means refusal, never a bypass. | ✅ |
| P4 | **Clients cannot choose identity or region.** Source address and pod come from the connection and operator configuration. | ✅ |
| P5 | **Replay resistance.** Authenticated requests are bound to a timestamp window and a single-use nonce; a repeated `tx_id` cannot consume capacity twice. | ✅ |
| P6 | **Opaque refusals.** Clients learn that a request was declined, not why. | ✅ |
| P7 | **Tamper evidence.** Editing, deleting or re-ordering any audit record is detectable offline. | ✅ |

## 5. Walk-through: compromised backend (A2)

The attacker can authenticate and send plausible orders. Each layer narrows what they can achieve:

| Step | Attack attempt | Layer that responds | Test |
|---|---|---|---|
| 1 | Withdraw to an attacker address | **Destination allowlist** refuses unknown recipients | `test_attacker_destination_is_not_on_the_allowlist` |
| 2 | Use few, large orders | **Per-transaction cap** refuses; amounts above the review threshold are **held**, not signed | `test_large_orders_are_held_for_review` |
| 3 | Add their address to the allowlist and stay below every threshold | **Global / pod / account ceilings**; total approved ≤ `C` in any window | `test_worst_case_allowlist_bypassed...` |
| 4 | Spread over accounts and pods | Global ceiling is shared; **breaker trips** and everything pauses | same test, `containment` asserted |
| 5 | Keep going after the trip | Refused as *contained* until an operator resets | `test_after_containment_nothing_more_leaves...` |
| 6 | Replay captured requests | Nonce cache, timestamp window, idempotent `tx_id` | `AuthTests`, `Idempotency` |
| 7 | Make the audit store fail to force a bypass | Fail-closed: no audit, no signature | `FailClosed` |

Step 3 is the worst case: the attacker has defeated every *recognition-based* control. The ceiling still
holds because it does not depend on recognising anything.

## 6. Out of scope and known gaps

* Compromise of the guard process itself, the host OS or the signer.
* An attacker who can *both* reach the signer directly *and* bypass the guard (deployment error).
* Multi-asset valuation: ceilings are in one common minor unit; a manipulated price feed would widen
  the real loss.
* Cold-wallet and treasury movements are not governed by this guard.
* Denial of service: an attacker able to push traffic to a ceiling can force a pause (04 §6).
  This is the intended failure mode but it is still an availability cost.
* Supply chain: dependencies are pinned with hashes in the build (05), but no SBOM or signing yet.
