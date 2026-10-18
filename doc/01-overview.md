# 01 · Overview

**The Octopus Paradigm: bounding the loss from a compromised hot-wallet backend**

Author: Mahmoud Al-Nouri · © 2026 · Status: prototype v0.4.2, not independently audited · Source code: private repository, reviewer access on request

> **How to read this documentation.** Every capability is tagged:
> ✅ implemented and tested in this repository · 🔶 designed, not built · ⛔ withdrawn from earlier drafts.
> Evidence for each ✅ is listed in [06-evaluation](06-evaluation.md).

## 1. Problem

Centralised custodians keep a fraction of customer assets in *hot* wallets so that withdrawals can be fast.
Recent large thefts show that the weak point is often not the key but the **authorization path**:
the attacker gets into the system that decides which withdrawals are legitimate and submits orders that
look perfectly well formed. In the 2026 incident at a major exchange (publicly reported loss of roughly
US$388 M; details differ between sources, verify against the primary disclosure before citing),
the signing infrastructure did what it was asked to do.

Detection will always lag some intrusions. A defence that depends on *recognising* the attacker therefore
fails exactly when the attacker is good. The question this project asks instead is:

> **If the backend is fully compromised, what is the maximum value that can still leave, and how long does
> it take to reach that maximum?**

## 2. Idea

An octopus survives an attack by giving up one arm. The guard does the same with availability:
it sits in front of the signer, enforces a hard value ceiling per time window, and when a ceiling is
reached it *trips*: withdrawals in that scope stop until an operator resets them. The loss is capped by
construction, not by the quality of an anomaly detector.

## 3. Contributions ✅

| # | Contribution | Where |
|---|---|---|
| C1 | **Value-weighted ceilings** (global, pod, account) with a *proven* bound: the total admitted in any window is at most the ceiling | `src/limiter.py`, [04](04-blast-radius-guarantee.md) |
| C2 | **Containment breaker**: reaching a pod or global ceiling pauses that scope until an authenticated operator resets it | `src/containment.py` |
| C3 | **Policy-before-signature, audit-before-signature, fail-closed**: no audit record, no signature; any internal failure means refusal | `src/engine.py` |
| C4 | **Server-decided identity**: HMAC-authenticated requests with timestamp window and nonce replay cache; client IP and pod are never taken from the request body | `src/auth.py`, `src/netutil.py` |
| C5 | **Tamper-evident audit chain** (hash-linked records, verifiable offline) | `src/audit.py` |
| C6 | **Measurement tooling** so every number can be reproduced | `scripts/benchmark.py`, `tests/` |
| C7 | **Shadow mode**: the full pipeline runs on mirrored live traffic and reports what enforcement *would* have done and how to size the ceilings, while being unable to sign, refuse or trip anything | `src/shadow.py`, `scripts/shadow_report.py`, [05](05-deployment-and-operations.md) |

## 4. Not in this release 🔶

* A production **signer** (HSM, TEE or audited MPC/TSS service). Only a development stub exists.
* Validation of the **shared-state (Redis) backend** against a live server.
* **Manual-review workflow** for withdrawals above the review threshold.
* A statistical **advisory model** for ceiling sizing and alerts (see [03](03-architecture.md#statistical-model)).
* Deception (decoy responses) beyond an opt-in research switch; micro-VM sandboxes; edge packet filtering.

Details and priorities: [07-limitations-and-roadmap](07-limitations-and-roadmap.md).

### Terminology: shadow mode is not the "ink"

| Term | Meaning | Status |
|---|---|---|
| **Shadow mode** | Observation and calibration: the guard evaluates mirrored live traffic, never signs, never refuses, and reports how to size the ceilings | ✅ |
| **Decoys** ("ink") | Fake "approved" replies to confuse an attacker | opt-in research switch `DECEPTION_ENABLED`, refused in production; value unproven (observers can check signatures or look hashes up on-chain) |
| **Tripwires** | Honey accounts or addresses nobody legitimate uses; any touch raises a high-confidence alert and trips containment | 🔶 roadmap R15; their value is *detection*, not deception |

## 5. Document map

| File | Question it answers |
|---|---|
| [02-threat-model](02-threat-model.md) | Who is the attacker, what do we assume, what is out of scope? |
| [03-architecture](03-architecture.md) | How does a request flow, and why was each design choice made? |
| [04-blast-radius-guarantee](04-blast-radius-guarantee.md) | What exactly is guaranteed, under which conditions, and what breaks it? |
| [05-deployment-and-operations](05-deployment-and-operations.md) | Where does it run, how is it configured, rolled out and operated? |
| [06-evaluation](06-evaluation.md) | What was tested and measured, and what happened to the earlier claims? |
| [07-limitations-and-roadmap](07-limitations-and-roadmap.md) | What is still missing, in what order should it be built? |
