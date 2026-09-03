# x402 Payment Security Checklist

A public reference standard for auditing the **payment integration** of an [x402](https://github.com/x402-foundation/x402) resource server — the side that sells an API service over HTTP 402 Payment Required.

## What this is

- **`CHECKLIST.md`** — the v2 checklist: six failure categories + a liveness pre-filter, with detection questions, fixes, and real cases. Severity-calibrated against measured, reproducible leakage (up to ~100%).
- **`METHODOLOGY.md`** — how to run the audit: white-box (read the code), black-box (attack the endpoint), on-chain (verify the challenge).

## What this is not

- **Not an audit engine.** This repo contains no scanner, no verifier, no PoC harness. It is the *question set and method*, deliberately separated from the tooling that executes it.
- **Not a formal audit or security certification.** A checklist pass is first-pass triage over the measured failure classes — it flags, it does not certify.

## Why x402 needs its own checklist

x402 couples synchronous HTTP with asynchronous on-chain settlement. The failure modes are not the classic smart-contract top-ten: they live in the *gap* between the request, the payment proof, and the delivery. Independent measurements put the worst classes at ~100% resource leakage (the seller delivers output and never gets paid), and an ecosystem audit found the live half of endpoints dominated by **receiver-side** problems — honeypot `payTo`, receiver hijack, unverified receivers. This checklist is organized around those measured modes.

## Quick start

1. Run **L0** (liveness) on the endpoint.
2. Walk **Categories 1–6** with `CHECKLIST.md`, using `METHODOLOGY.md` to gather answers.
3. Label each finding with its victim direction (payer→resource or resource→payer).
4. Map findings to the severity table and triage.

## Corrections & honesty in the data

An early "76% of x402 endpoints are dead" figure was a prober bug (read payment requirements from the body after x402 v2 moved them to the `PAYMENT-REQUIRED` header). The corrected death rate is **38%**. Dead endpoints are an availability problem, not a payment-security flaw — hence the L0 pre-filter.

## License

MIT — the checklist and method are free to use, adapt, and cite. Attribution appreciated.
