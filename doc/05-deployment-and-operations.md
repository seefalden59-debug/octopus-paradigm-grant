# 05 · Deployment and operations

Author: Mahmoud Al-Nouri · © 2026 · Tags: ✅ implemented · 🔶 designed · ⛔ withdrawn

## 1. What "non-invasive" really means

The guard can be placed in front of an existing signing service without rewriting the exchange backend.
That is a small, specific change, not "zero change":

| Change | Who makes it |
|---|---|
| Backend sends withdrawal requests to the guard, signed with the API HMAC secret | Backend integration (client library or a few lines, see README) |
| **Signer accepts approvals only from the guard** (mutual TLS, network policy and a signed approval) | Signer / infrastructure team. **This is what makes the ceiling real.** |
| Ceilings, allowlist and pod map are configured | Security / risk team |

Without the second row the guard is a speed bump, not a ceiling (see 04 §5a).

## 2. Topology

```
 Backend ──TLS──► Guard (1 process) ──mTLS──► Signer (HSM/MPC)
                    │   ▲
        audit file ─┘   └─ Admin: separate secret, ADMIN_ALLOWED_CIDRS, operator network only
```

* Run **one** guard process per ceiling, or share state through Redis, or partition the budget (03 §4).
* Terminate TLS in front of the guard (reverse proxy) and list the proxy in `TRUSTED_PROXIES`, so only it
  may set `X-Forwarded-For`.
* Network policy: only the backend may reach the guard; only the guard may reach the signer; only the
  operator network may reach the admin paths.
* Run as a non-root user, read-only filesystem, no extra capabilities except `IPC_LOCK` (the provided
  `docker-compose.yml` does this) ✅.

## 3. Configuration (essentials)

Full list with comments: `.env.example`. The service **refuses to start** on inconsistent settings.

| Variable | Meaning |
|---|---|
| `API_HMAC_SECRET`, `ADMIN_HMAC_SECRET` | ≥ 32 bytes each, different; also accepted as `*_FILE` for mounted secrets |
| `HOT_WALLET_BALANCE` | Integer minor units; basis for every ceiling |
| `GLOBAL_LIMIT_BPS`, `POD_LIMIT_BPS`, `ACCOUNT_LIMIT_BPS`, `SINGLE_TX_BPS`, `HOLD_BPS` | Must satisfy HOLD ≤ SINGLE_TX ≤ ACCOUNT ≤ POD ≤ GLOBAL |
| `LIMIT_WINDOW_SECONDS`, `STRICT_WINDOW` | Window `W` and the guaranteed-bound mode (keep `true`) |
| `WINDOW_SKEW_TOLERANCE_MS` | Worst clock difference between workers that serve requests. 0 only for one process; **required (> 0) with Redis** (e.g. 1000). Sets the per-bucket cap `C/k` (04 §3b) |
| `REQUIRE_ALLOWLIST`, `ALLOWED_DESTINATIONS` | Production requires a non-empty allowlist |
| `POD_MAP`, `TRUSTED_PROXIES`, `ADMIN_ALLOWED_CIDRS` | Network trust decisions |
| `AUDIT_PATH`, `AUDIT_FSYNC` | Audit file; `fsync` per record trades speed for durability |
| `ALLOW_STUB_SIGNER` | Development only; production refuses to start with the stub |
| `SHADOW_MODE` | `true` = evaluate and audit mirrored traffic, never enforce, never sign (§4 phase 1). No signer needed; production allowlist not required while shadowing |
| `DECEPTION_ENABLED` | Research only; refused in production |

## 4. Roll-out (indicative, to be agreed per integration)

The earlier 10-week plan promised zero disruption. No roll-out can promise that; this one ties each step
to an exit criterion instead of a date.

| Phase | Goal | Exit criterion |
|---|---|---|
| 0. Sizing | Choose `g`, `W`, thresholds from historical withdrawals (04 §4) | Legitimate peak per window ≤ `⌊C/2⌋` with margin, reviewed by risk |
| 1. **Shadow** ✅ | Mirror live withdrawals to the guard with `SHADOW_MODE=true` (§5) | Every would-be refusal or hold is explained; decline rate agreed with operations; ceilings sized from the report |
| 2. Signer lock-down | Signer accepts guard approvals only | Direct requests to the signer are refused in a test |
| 3. Canary | Enforce for a small slice of withdrawals (e.g. one pod, low ceilings) | Stable latency, no unexplained refusals, breaker drill completed |
| 4. Ramp | Increase coverage to all pods | Operator runbooks exercised; audit anchoring running |

## 5. Shadow mode in practice ✅

**Integration.** The backend mirrors each withdrawal request to the guard *asynchronously* and ignores the
answer; the production signing path is untouched. Shadow answers are always HTTP 200 with
`{"status":"shadow","would_be":"approved|pending_review|declined","reference":...}` and never contain a
signature. The mirror call must be fire-and-forget: in shadow mode the guard's failures must never affect
production (this is the only place where "ignore the guard's errors" is correct).

**Reading the results.**
* Live: signed `GET /v1/admin/shadow` returns aggregates only (no accounts or addresses): counts and values
  by would-be verdict, decline reasons, peak demand per window (global and per pod), would-be trips, and the
  minimum basis points that cover observed demand.
* Offline: `python -m scripts.shadow_report /data/audit.jsonl --balance <B> --window <W> --global-bps <g>`
  verifies the audit chain first and refuses an edited log.

**Exit criteria.** Every decline or hold reason is understood; the decline rate is accepted by operations;
`GLOBAL_LIMIT_BPS`, `POD_LIMIT_BPS` and `SINGLE_TX_BPS` are set above the reported minimums with margin
(mirrored traffic that contained an attack inflates demand, so review outliers by hand); the allowlist
has been built from observed destinations and reviewed.

**Limits.** The report is only as representative as the traffic mirrored, and shadow mode proves nothing
about the signer lock-down (phase 2) or about behaviour under attack.

## 6. Operations

**Monitoring signals** (log lines at `WARNING`/`CRITICAL`): `containment tripped`, `auth rejected`
(reason code), `audit unavailable; refusing to sign`, `signer failure`, `redis limiter failure`.
Alert on any trip, and on a sustained rise in `auth rejected` or refusals.

**Runbook: containment tripped**
1. Treat it as a security incident first, a capacity question second.
2. Read `/v1/admin/status` (signed with the admin secret) for the mode, tripped scopes and the enforced
   usage (`global_usage`, with `window.global_bucket_cap` and `window.global_headroom`).
3. Review the audit trail around the trip; verify the chain (`src.audit.verify_file`).
4. Only after review, reset the scope with a signed `POST /v1/admin/containment/reset`
   (`{"scope": "pod-a"}`, `"global"`, or `"*"`). Every reset is itself audited.
   Remember that each reset re-arms one more ceiling (04 corollary 2).

**Audit anchoring.** The chain is tamper-*evident*. To make truncation detectable, copy the latest `head`
hash (from `/v1/admin/status`) on a schedule to storage the guard host cannot modify (WORM bucket, a second
machine, an external ledger).

**Restarts.** With the in-memory backend a restart clears the windows (04 §5c). Until persistence or a
start-up hold exists, restart under change control and review the audit trail for restart loops.

**Secret rotation.** Rotate the HMAC secrets on a schedule and after any suspected exposure; restart
picks up new values. Secrets never appear in logs, status output or responses.

## 7. Claims removed from earlier drafts ⛔

"Zero backend refactoring", "99.999 % uptime", "failover in under 50/100 ms", "fail over to the primary
MPC path" and "guaranteed zero disruption in 10 weeks" are withdrawn: none was measured, and the
fail-over described would bypass the guard. Availability comes from replicated guards with shared or
partitioned budgets, tested under failure, and is a roadmap item (07).
