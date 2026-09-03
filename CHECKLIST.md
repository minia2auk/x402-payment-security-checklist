# x402 Payment Integration Security Checklist — v2

A reference checklist for auditing the **payment integration of an x402 resource server** — the side that *sells* an API service over the x402 protocol (HTTP 402 Payment Required).

**Version:** v2 · 2026-09-03
**Status:** public reference standard. This repo contains a checklist and methodology only — no scanner, no verifier, no engine code. See `README.md`.

---

## Why this exists

x402 moves money synchronously with an HTTP request: a buyer agent receives a `402` challenge (price + `payTo` address), pays in USDC, the resource server verifies, then delivers. When the resource-side integration is wrong, one of two things happens:

- **The seller gets free-ridden** — forged payment headers, replayed transactions, delivery races, or allowance overdraft let an attacker get the service without paying (or pay once and consume N times).
- **The buyer gets cheated** — a honeypot `payTo`, receiver hijack, or bait-and-switch pricing takes their money without delivering.

Independent measurements put the worst classes at **~100% resource leakage** (the seller delivers output and never gets paid). An ecosystem audit of 5,077 tracked x402 endpoints found the live half riddled with **receiver-side problems** — of five problem categories, three are `payTo`/receiver security. The checklist below is organized around those measured failure modes.

> **Correction note (honesty in the data):** an early widely-cited "76% of x402 endpoints are dead" figure was wrong — the author's prober read payment requirements from the response body after x402 v2 moved them into the `PAYMENT-REQUIRED` header. The corrected death rate is **38%**. Dead endpoints are an availability/liveness problem, not a payment-security flaw — see L0.

---

## Severity model

| Level | Meaning | Empirical anchor |
|---|---|---|
| **Critical** | Deterministic money loss — attacker always wins | floating authorization (100/100 reproduced); honeypot `payTo` (paid, no delivery) |
| **High** | High-probability or default-triggered loss | allowance overdraft (97.76% leakage); denial-of-settlement (86.95% leakage); receiver hijack |
| **Medium** | Probabilistic or bounded loss | duplicate-settlement race (~6% of rounds); memory-only replay lock (lost on restart) |
| **Low / Info** | Hardening or documentation gap | weak web↔chain consistency; missing refund guards |

---

## L0 — Liveness & reachability (pre-filter)

Run this **before** any of the six categories. A dead endpoint has no exploitable payment flow — it never returns a well-formed `402` — so it fails here and never reaches a security review.

- [ ] Does the endpoint return a syntactically valid `402` (price + `payTo` present and parseable)?
- [ ] Is `payTo` a non-zero address (not `0x0…`, not the payer's own wallet)?
- [ ] Is the challenge amount `> 0` and denominated in a real settlement asset?
- [ ] Does the endpoint actually respond to a valid request within a sane timeout?

**Why L0 matters:** ~38% of tracked endpoints are unreachable. Before asking "is the payment integration secure?", ask "is it even alive and really taking payment?". This is the same verify-first discipline that separates a live, honest marketplace from a directory of dead links.

---

## Category 1 — Payment verification authenticity

**Risk.** The server trusts a payment *header* (`PAYMENT-SIGNATURE` / `X-PAYMENT` / a `txHash` field) without verifying the on-chain transfer → an attacker forges the header and gets the service free.

**Detection**
- [ ] On a request carrying payment credentials, does the server actually call chain/facilitator verification (read receipts, call `/verify`)?
- [ ] Does a failed verification refuse delivery — no fallback to "return the result anyway"?
- [ ] Can the verification path be bypassed (a different endpoint, a different header name, a "trusted internal" branch)?
- [ ] Which network/chain is being verified — can it be pointed at a testnet or worthless chain?

**Live case (minia2a).** The gateway runs two verification paths — Path A: facilitator `verify`/`settle` (two-step), Path B: on-chain receipt inspection that iterates USDC transfer logs and confirms a transfer to `payTo`. Verification failure returns `402`/`502`, no delivery.

**Fix.** Complete on-chain verification (receipt logs or facilitator) before delivery; fail closed on verification failure; keep verification in one place so it can't be bypassed.

**Related paper class:** *cross-resource substitution* — a "floating authorization" signed for resource `r_a` re-attached to any same-priced `r_b` because the authorization isn't cryptographically bound to the resource. Reproduced 100/100. Protocol-level fix: EIP-712 context binding, sign `H(Method‖URI‖Body)`.

---

## Category 2 — Replay protection (nonce / txHash abuse)

**Risk.** The same payment (txHash / signature) is reused → pay once, get the result N times.

**Detection**
- [ ] Can the same txHash / payment signature call the service repeatedly?
- [ ] Is replay protection persisted (DB / on-chain), or memory-only (lost on restart)?
- [ ] What is the replay key's granularity — global / per-service / per-wallet — and can it be bypassed by reusing the key across services?
- [ ] After a refund / partial refund, can the original txHash still be consumed?

**Live case (minia2a).** Dual replay protection: a `replay_lock` DB table plus a gateway in-memory cache, key = `path + txHash`. **Pitfall hit:** after consuming a txHash in test, the gateway had to be restarted to clear the memory cache — DB deletion wasn't enough. The memory cache is a single point of failure on restart; the DB is the durable line of defense.

**Fix.** Persist the replay key to a DB with a unique constraint; include the service ID in the key to prevent cross-service reuse; mark txHash consumed on refund.

**Related paper class:** *duplicate-settlement race* — verification and settlement are decoupled, so N concurrent requests with the same one-time authorization all clear verification before any reaches on-chain consumption. Reproduced duplicate delivery in ~6% of rounds. Fix: stateful nonce linearization (atomic check-and-lock at ingress).

---

## Category 3 — Delivery timing & web↔chain races

**Risk**
- **Delivery before verification** — concurrent/async paths deliver before verification completes → free ride.
- **web↔chain desync** — the web layer records "paid" while the chain hasn't confirmed, or the chain confirms while the web state is tamperable.
- **Race** — the same order/session consumed concurrently → double-spend of the service resource.

**Detection**
- [ ] Is delivery (returning the result) strictly after verification succeeds — any async path that delivers early?
- [ ] Can the web-layer payment state be tampered with by the client (e.g. injected `order=paid` parameter)?
- [ ] Does a concurrent call on the same payment/order deliver multiple times (no atomic lock)?
- [ ] Is server-side billing state (credits/quota) updated atomically (no read-modify-write race)?

**Live case (minia2a).** The "Free Shopping" fix: **settle before deliver** — confirm payment → record it → then call the service. A liveness probe runs before delivery; if the service is unavailable, settlement is refused (no money taken, no delivery made).

**Fix.** Deliver only atomically after verification + accounting succeed; use DB transactions / unique constraints for order state; never trust web-layer state — the chain is the source of truth.

**Related paper classes:** *duplicate-settlement race* (above) and *denial of settlement* (Category 6). A secondary case: one SDK gated settlement on a 2xx status, so a `302` redirect was served without settling at all.

---

## Category 4 — Allowance / permit2 management

**Risk**
- The buyer-side integrator grants the receiver a **max `uint256` allowance** → the receiver (or a compromised receiver) can drain the full allowance.
- permit2 authorization with unclear expiry/scope → long-lived exposure.
- The receiver changes `payTo`/collection logic after authorization → funds redirected.

**Detection**
- [ ] Does the integration code request a max `uint256` allowance? Is it tightened after use?
- [ ] Does the permit2 authorization have an expiry / single-use ceiling?
- [ ] Can the receiver change the collection address during the authorization window (mutable `payTo`, no notice)?
- [ ] Does a revocation path exist and work?

**Live case.** x402 uses **permit2-exact** (authorize + single transfer). minia2a challenges advertise `assetTransferMethod: permit2-exact` — precise single-use authorization beats max approval.

**Fix.** Use permit2 single-use (exact) authorization, never max; clear/expire after use; collection-address changes require governance + notice.

**Related paper class:** *allowance overdraft* — under the `upto` dynamic-pricing scheme, verification is a non-binding snapshot of remaining allowance, so a burst of concurrent requests all pass verification and stream output while only the first few settle. A 50-request burst delivered **47,277 wei** but settled only **1,057 wei** — a **97.76% leakage ratio**; the long-context variant approaches 100%. Fix: reserve-commit two-phase locking (escrow the cap, refund the difference).

---

## Category 5 — Funds & payTo security

**Risk**
- `payTo` misconfiguration (wrong address, or self-send) → money sent to the wrong place or everything rejected.
- Platform wallet private-key insecurity / single point of failure.
- Can't move funds after receipt (no gas / no operating permission).
- Refund-logic flaws (triggered multiple/incorrect refunds).

**Detection**
- [ ] Does the challenge's `payTo` point at the correct receiver? Can it be injected/tampered?
- [ ] How are the collection wallet's keys managed (cold/hot separation, permissions)?
- [ ] Is the settlement chain funded with enough gas to move funds out?
- [ ] Can refund conditions be abused (unpermissioned trigger / double refund)?
- [ ] Does changing the collection address require authorization (prevent attacker rewriting `payTo` to hijack collections)?

**Live case (minia2a).** Hit **`self_send_not_allowed`** — `payTo` was configured as the same address as the payer wallet, and the CDP facilitator rejected all self-sends, halting payments. Lesson: `payTo` must be an independent receiver, never the payer-side wallet. Also: **the refund chain = the actual payment chain** (not a handler default), preventing wrong-chain refunds.

**Fix.** Independent + permission-gated `payTo`; layered wallet-key custody; gas reserved; refunds on the actual payment chain with authorization and idempotency.

**Ecosystem data:** this is the **highest-frequency real-world class**. In the live half of the ecosystem audit, three of five problem categories were receiver-side — honeypot receivers (zero/malformed `payTo`), receiver hijack, and unverified receivers (no on-chain USDC history). A buyer-facing audit should start here.

---

## Category 6 — Denial of settlement (rate-limit asymmetry) — NEW in v2

**Risk.** Verification ingress is provisioned far higher than settlement egress (default ~10 tx/s). An attacker floods valid requests: every one is verified and served, but settlement requests beyond the limit are rejected with `429`. The seller streams output and never gets paid.

**Detection**
- [ ] Is the settlement rate limit provisioned at the same scale as request ingress?
- [ ] Does the server verify + reserve billing capacity **before** forwarding to the (expensive) service engine, refusing excess with `429` up front?
- [ ] Does a settlement failure stop delivery (fail closed), or does the service keep streaming?

**Measured:** a 50 req/s burst against a 10 tx/s settlement limit streamed all 50 requests but settled only 5 — an **86.95% leakage ratio**, rising to 100% when the limit blocks everything. A client-driven variant terminates the TCP connection before the settlement callback fires, consuming streamed output without paying.

**Fix.** Failure-closed design: couple the rate limiter and billing to request **ingress** — reserve billing capacity before execution. The principle to internalize: *reliability problems should become denial of service, not denial of payment.* The paper's bounded-loss streaming (checkpoint every Δ tokens, settle cumulative cost) caps worst-case leakage per interruption at ~$0.00047.

**Related paper class:** *denial of settlement* — one of the four measured flaw classes, and a deployment default that silently leaks revenue.

---

## Victim-direction note

The six categories are not all one direction. Label each finding:

- **payer → resource** (buyer attacks seller): Categories 2, 3, 4, 6 — free-riding, replay, allowance overdraft, denial of settlement. Silent seller-side loss.
- **resource → payer** (seller cheats buyer): Category 1 (bait-and-switch pricing), Category 5 (honeypot `payTo`, receiver hijack). Direct buyer-side theft.
- **bidirectional:** Category 5 — a hijacked `payTo` cheats buyers; a misconfigured `payTo` (self-send) breaks the seller.

This maps to two different audit products: a **buyer-facing audit** leads with Category 5 + pricing authenticity (what the ecosystem actually shows); a **seller-facing audit** leads with Categories 3, 4, 6 (the silent leakage the paper measured).

---

## Real ecosystem cases (for calibrating severity)

| Category | Case | Source | One-liner |
|---|---|---|---|
| 1 | Floating authorization: signature not bound to resource, replayed against any same-priced endpoint | arXiv systematic analysis | 100/100 reproduced |
| 1 | Bait-and-switch: quoted price ≠ charged price | ecosystem audit | violates value-consistency invariant |
| 2 | `RequirePaymentAndSettle` with no nonce dedup (verify→execute→settle) | production x402 middleware (fixed) | N concurrent requests → N deliveries |
| 3 | Flask adapter gates settlement on 2xx → `302` served without settling | arXiv systematic analysis | delivery before settlement confirmation |
| 4 | 50-request burst: 47,277 wei delivered / 1,057 wei settled = 97.76% leakage | arXiv systematic analysis | `upto` pricing + TOCTOU |
| 5 | `self_send_not_allowed`: `payTo` = payer wallet, CDP rejects all self-sends | minia2a (live) | `payTo` must be an independent receiver |
| 5 | Honeypot receiver: `payTo` points to a dead/fake wallet | ecosystem audit | highest-frequency live class |
| 6 | 50 req/s vs 10 tx/s settle limit → 5/50 settled = 86.95% leakage | arXiv systematic analysis | insecure deployment default |

---

*This checklist is a living reference. It encodes hard-won lessons from running a production x402 gateway plus independently reproduced findings from the academic and ecosystem audits. It ships with `METHODOLOGY.md` (how to run the audit) and `README.md` (scope and boundaries).*
