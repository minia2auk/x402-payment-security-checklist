# Methodology — how to audit an x402 payment integration

Three lenses, run in order. `CHECKLIST.md` is the question set; this is how to *get* the answers.

## 1. White-box (read the code)

Read the resource server / gateway / integrator code and check each checklist item against the actual implementation.

- Trace the payment-verification path from header to delivery. Where does it verify? Is verification in one place or scattered (scattered = bypassable)?
- Find the replay key. Is it persisted or in-memory? Does the key include the service/resource ID?
- Find the delivery point. Is it strictly after verification + accounting?
- Find the allowance/permit2 grant. `exact` or `max`?
- Find `payTo` and its change path. Is it configurable at runtime by an untrusted party?

Output: a list of checklist items marked PASS / FAIL / UNVERIFIED, with file/line references.

## 2. Black-box (attack the live endpoint)

Send crafted requests to the running x402 endpoint and observe whether it can be free-ridden, replayed, raced, or hijacked.

- **Forged payment header** — send a fake `X-PAYMENT`/`PAYMENT-SIGNATURE`/`txHash`; is the result delivered anyway? (Category 1)
- **Replay** — reuse an already-consumed txHash/signature; does it deliver again? (Category 2)
- **Concurrent same-order** — fire N parallel requests for one payment; how many deliver? (Category 3)
- **Delivery timing** — does a `302`/redirect or early-stream path deliver before settlement? (Category 3, 6)
- **Settlement flood** — burst valid requests above the settle rate limit; what fraction settle? (Category 6)
- **`payTo` inspection** — is the advertised `payTo` a non-zero, non-self address with on-chain USDC history? (Category 5, L0)

Output: observed behavior per category, with the request/response used.

## 3. On-chain (verify what the challenge claims)

Verify the chain-side facts independently of the web layer.

- Does the challenge's `payTo` match the actual on-chain receiver? Any on-chain USDC history (a receiver with zero history is a red flag)?
- What is the allowance amount actually granted to the facilitator/receiver?
- For a real payment: does the transfer settle on the correct chain, to the correct address, for the correct amount?

Output: on-chain confirmation or contradiction of the web-layer claims.

## Ground rules

1. **L0 first.** A dead/malformed endpoint fails at L0 and never reaches a deep review.
2. **The chain is the source of truth.** The web layer's "paid" state is an opinion; the receipt is the fact.
3. **Label victim direction** on every finding (payer→resource vs resource→payer). It changes who the audit is for and how it's prioritized.
4. **Honest scope.** A static checklist pass is *not* a formal audit — it's a first-pass triage that flags the measured, reproducible failure classes. Deep verification (PoC reproduction) is a separate, heavier step and is intentionally out of scope for this reference.
