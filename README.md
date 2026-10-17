The Octopus Paradigm
Bounding the loss from a compromised hot-wallet backend
Public architecture documents and grant proposal
Status: research prototype (v0.4.1) · source code in a private repository, reviewer access on request ·
not independently audited · documentation © 2026 Mahmoud Al-Nouri
Revision note. This README replaces an earlier summary. Figures such as "≤ 0.7 ms" and "≤ 3 %" were design
targets, not measurements, and some components it described (a Go layer, micro-VM sandboxes, key-shredding
"autotomy") are not built. They have been withdrawn or restated. The complete list is the
claims register.

The problem
Large custodial thefts increasingly come through the withdrawal authorization path rather than stolen keys: the
attacker reaches the system that decides which withdrawals are legitimate and submits well-formed orders. A defence
that must recognise the attacker fails exactly when the attacker is good.
The idea
An octopus survives by giving up one arm. The guard sits in front of the signer and enforces a hard value ceiling
per time window. When a pod or global ceiling is reached, that scope pauses until an operator resets it. The
maximum loss is capped by construction, not by the quality of a detector. Splitting the hot wallet into
independent pods then limits how much any single compromise can reach.
What exists today
Capability
Status
Value-weighted ceilings (global, pod, account) with a proved window bound
built, tested
Containment breaker; fail-closed on every internal fault
built, tested
Audit record written before signing, hash-chained and verifiable offline
built, tested
HMAC-authenticated API with timestamp window and replay protection; client IP and pod decided by the server
built, tested
Shadow mode: evaluate mirrored live traffic without enforcing or signing, and report how to size the ceilings
built, tested
Real signer integration (HSM / MPC / EVM adapter)
not built (milestone M1)
Restart safety, validated shared-state backend, independent pods, dual control
not built (milestones M2-M3)
Evidence (reproducible; details in docs/06-evaluation.md)
A measured flaw in a common technique. The usual O(1) sliding-window counter admitted 190 units against a
limit of 100 inside one window when an attacker chose the timing. The strict rule used here cannot exceed the
limit, with a short proof (docs/04).
Bound. With a 3 % ceiling and one-hour windows, emptying a hot wallet through this path takes at least about
66 hours of uninterrupted, un-tripped abuse, under the conditions stated in the document.
Decision latency (in-process only). About 42-45 µs at the median and 90-160 µs at the 99th percentile on a
small VM, excluding network, TLS and a real signer. End-to-end latency is not yet measured.
Tests. 97 written; in the reference environment 79 executed and passed, and 18 that need FastAPI or Redis have
not yet been run there.

.
Fit with the BNB Chain ecosystem
Exchanges and custodians that hold BNB and BEP-20 assets carry the risk this guard addresses. The proposal's
concrete contribution is M1: an EVM reference signer adapter, built on an audited library, with a scripted demo
on BNB Smart Chain testnet. The design itself is chain-agnostic.
Milestones (revised)
#
Milestone
Acceptance
M1
EVM reference signer adapter + BNB Smart Chain testnet demo
Scripted demo: approve within the ceiling, refuse beyond it, trip, reset. End-to-end latency measured and published with hardware, whatever the value.
M2
Restart safety and a validated shared-state (Redis) backend
A crash loop cannot reset a ceiling; Redis contract tests pass under failure injection
M3
Independent pods and dual control; open-source release of the core library and documentation
Compromising one pod's credentials cannot approve in another; reset needs two approvers; repository and licence published
M4
Verification and independent review
Property-based or model-checked proof of the window bound; external review summary published
Amount and timeline: as stated in the submitted grant application. These milestones refine the ones in that
application; the reasons are given in the revision note at the top.
Openness and reviewer access
The source code is in a private repository during evaluation. Reviewers can request read access, or a technical
walkthrough, through the contact details in the submitted grant application. Grant-funded deliverables (M1-M4) will be developed under an open-source
licence: [DECISION: state the licence, e.g. Apache-2.0 or MIT].
Honest limitations
No production signer yet; ceilings hold only if every signature goes through the guard; state is in memory
(restart clears windows) and the Redis backend is unvalidated; pods are not yet independent control planes; a breaker
trip is a controlled denial of service; no independent audit. Full list:
docs/07.
Documents


01 Overview
Problem, idea, contributions, terminology
02 Threat model
Adversaries, assumptions, compromised-backend walk-through
03 Architecture
Request path, design decisions, shadow mode, pod independence
04 Blast-radius guarantee
Theorem, proof, conditions, what breaks it
05 Deployment and operations
Topology, configuration, roll-out, runbooks
06 Evaluation
Tests, measurements, claims register
07 Limitations and roadmap
What is missing and in what order

