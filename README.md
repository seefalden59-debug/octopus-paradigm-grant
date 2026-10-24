# The Octopus Paradigm

### Bounding the loss from a compromised hot-wallet backend
Public architecture documents and grant proposal

> **Status:** research prototype (v0.4.3) · source code in a private repository, reviewer access on request
> (see [Source code access](#source-code-access)) · not independently audited · documentation © 2026 Mahmoud Al-Nouri
> · documentation licensed under [CC BY 4.0](LICENSE)

> **Revision note.** This README replaces an earlier summary. Figures such as "≤ 0.7 ms" and "≤ 3 %" were design
> targets, not measurements, and some components it described (a Go layer, micro-VM sandboxes, key-shredding
> "autotomy") are not built. They have been withdrawn or restated. The complete list is the
> [claims register](docs/06-evaluation.md#4-claims-register-what-happened-to-the-earlier-statements).

## The problem

Large custodial thefts increasingly come through the withdrawal **authorization path** rather than stolen keys: the
attacker reaches the system that decides which withdrawals are legitimate and submits well-formed orders. A defence
that must *recognise* the attacker fails exactly when the attacker is good.

## The idea

An octopus survives by giving up one arm. The guard sits in front of the signer and enforces a hard **value ceiling
per time window**. When a pod or global ceiling is reached, that scope pauses until an operator resets it. The
maximum loss is capped by construction, not by the quality of a detector. Splitting the hot wallet into
independent pods then limits how much any single compromise can reach.

## What exists today

| Capability | Status |
|---|---|
| Value-weighted ceilings (global, pod, account) with a **proved window bound** | built, tested |
| Containment breaker; fail-closed on every internal fault | built, tested |
| Audit record written *before* signing, hash-chained and verifiable offline | built, tested |
| HMAC-authenticated API with timestamp window and replay protection; client IP and pod decided by the server | built, tested |
| **Shadow mode**: evaluate mirrored live traffic without enforcing or signing, and report how to size the ceilings | built, tested |
| **Tripwires**: honey-accounts/addresses bypass policy and the ceilings entirely (any amount) and trip containment immediately | built, tested |
| Real signer integration (HSM / MPC / EVM adapter) | **not built** (milestone M1) |
| Restart safety, validated shared-state backend, independent pods, dual control | **not built** (milestones M2-M3) |

## Evidence (reproducible; details in [docs/06-evaluation.md](docs/06-evaluation.md))

* **A measured flaw in a common technique.** The usual O(1) sliding-window counter admitted **190 units against a
  limit of 100** inside one window when an attacker chose the timing. The strict rule used here cannot exceed the
  limit, with a short proof ([docs/04](docs/04-blast-radius-guarantee.md)).
* **Bound.** With a 3 % ceiling and one-hour windows, emptying a hot wallet through this path takes at least about
  66 hours of uninterrupted, un-tripped abuse, under the conditions stated in the document.
* **Decision latency (in-process only).** About 42-45 µs at the median and 90-160 µs at the 99th percentile on a
  small VM, excluding network, TLS and a real signer. End-to-end latency is **not yet measured**.
* **Adversarial fuzzing.** 50,000 random sequences per property (10 million attempts each, reproducible by seed): the
  window bound was never exceeded, and the same campaign **found and fixed a flaw** (the one-clock rule reached
  1.5× the ceiling when workers' clocks differ), documented in [docs/04](docs/04-blast-radius-guarantee.md).
* **Real-hardware validation (2026-10-22).** The guard was run end to end for the first time outside the authoring
  environment, on a GitHub Codespace (Linux, Python 3.14): `python -m src.main` started successfully with the
  development stub signer, a health check returned a healthy response, and the full test suite ran with
  **125 tests, 123 passed, 2 skipped** (both skips require a live Redis server — none failed). The same 20,000-sequence
  fuzz campaign was re-run there and reproduced the original result exactly: the window bound held with zero
  violations, and the two documented flaws were still correctly detected. See
  [docs/06-evaluation.md §4c](docs/06-evaluation.md) for the full account.

## Fit with the BNB Chain ecosystem

Exchanges and custodians that hold BNB and BEP-20 assets carry the risk this guard addresses. The proposal's
concrete contribution is **M1**: an EVM reference signer adapter, built on an audited library, with a scripted demo
on BNB Smart Chain testnet. The design itself is chain-agnostic.

## Milestones

| # | Milestone | Acceptance |
|---|---|---|
| M1 | EVM reference signer adapter + BNB Smart Chain testnet demo | Scripted demo: approve within the ceiling, refuse beyond it, trip, reset. **End-to-end latency measured and published with hardware, whatever the value.** |
| M2 | Restart safety and a validated shared-state (Redis) backend | A crash loop cannot reset a ceiling; Redis contract tests pass under failure injection |
| M3 | Independent pods and dual control; **open-source release of the core library and documentation** | Compromising one pod's credentials cannot approve in another; reset needs two approvers; repository and license published |
| M4 | Verification and independent review | Property-based verification is already done (see Evidence, above); this milestone adds a model-checked proof of the window bound (a first, unexecuted model already exists) and an external review summary, published |

### Work plan (sequence only — budget and timeline are in the submitted application, not repeated here)

1. Integrate a production signer (HSM / MPC) and build the EVM reference adapter and testnet demo (M1).
2. Validate restart safety and the shared-state (Redis) backend under failure injection (M2).
3. Separate credentials/signing per pod and add dual control for admin actions; publish the core library under an
   open-source license (M3).
4. Run the existing TLA+ model and record the result; commission an independent security review (M4).

## Source code access

The implementation currently lives in a **private repository** while it is under active development and informal
review. This is a working choice for the current stage, not a refusal to share it: **reviewers — grant committee
members, security researchers, or potential technical partners — can be added to the private repository as
collaborators on request**, with read access to the full source, tests, and commit history. Please use the contact
details in the submitted application to request access.

Independently of any single reviewer's access, **publishing the core library (guard, limiter, engine, tests) under
an OSI-approved open-source license (MIT or Apache-2.0) is milestone M3** above — a committed deliverable, not a
vague intention. This documentation repository (design, proof, evaluation, roadmap, and this README) is already
public and licensed under [CC BY 4.0](LICENSE); note that this license covers the documentation only, not the
not-yet-published source code.

## Honest limitations

No production signer yet; ceilings hold only if every signature goes through the guard; state is in memory
(restart clears windows) and the Redis backend is unvalidated; pods are not yet independent control planes; a breaker
trip is a controlled denial of service; no independent audit. Full list:
[docs/07](docs/07-limitations-and-roadmap.md).

## Documents

| | |
|---|---|
| [01 Overview](docs/01-overview.md) | Problem, idea, contributions, terminology |
| [02 Threat model](docs/02-threat-model.md) | Adversaries, assumptions, compromised-backend walk-through |
| [03 Architecture](docs/03-architecture.md) | Request path, design decisions, shadow mode, pod independence |
| [04 Blast-radius guarantee](docs/04-blast-radius-guarantee.md) | Theorem, proof, conditions, what breaks it |
| [05 Deployment and operations](docs/05-deployment-and-operations.md) | Topology, configuration, roll-out, runbooks |
| [06 Evaluation](docs/06-evaluation.md) | Tests, measurements, claims register, real-hardware run |
| [07 Limitations and roadmap](docs/07-limitations-and-roadmap.md) | What is missing and in what order |
