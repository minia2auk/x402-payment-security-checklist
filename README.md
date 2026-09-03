# x402 Payment Security Checklist

![status](https://img.shields.io/badge/status-v2%20live-2ea44f) ![license](https://img.shields.io/badge/license-MIT-blue) ![categories](https://img.shields.io/badge/failure%20categories-6%20%2B%20L0-important)

A public reference standard for auditing the **payment integration** of an [x402](https://github.com/x402-foundation/x402) resource server — the side that sells an API service over HTTP 402 Payment Required. Built from running an x402 gateway in production; **this is a living standard, exercised by paid checks every day.**

## Contents

- [What this is](#what-this-is)
- [What this is not](#what-this-is-not)
- [Why x402 needs its own checklist](#why-x402-needs-its-own-checklist)
- [Used by (this standard is live)](#used-by-this-standard-is-live)
- [Quick start](#quick-start)
- [Corrections & honesty in the data](#corrections--honesty-in-the-data)
- [License](#license)

## What this is

- **`CHECKLIST.md`** — the v2 checklist: six failure categories + a liveness pre-filter, with detection questions, fixes, and real cases. Severity-calibrated against measured, reproducible leakage (up to ~100%).
- **`METHODOLOGY.md`** — how to run the audit: white-box (read the code), black-box (attack the endpoint), on-chain (verify the challenge).

## What this is not

- **Not an audit engine.** This repo contains no scanner, no verifier, no PoC harness. It is the *question set and method*, deliberately separated from the tooling that executes it.
- **Not a formal audit or security certification.** A checklist pass is first-pass triage over the measured failure classes — it flags, it does not certify.

## Why x402 needs its own checklist

x402 couples synchronous HTTP with asynchronous on-chain settlement. The failure modes are not the classic smart-contract top-ten: they live in the *gap* between the request, the payment proof, and the delivery. Independent measurements put the worst classes at ~100% resource leakage (the seller delivers output and never gets paid), and an ecosystem audit found the live half of endpoints dominated by **receiver-side** problems — honeypot `payTo`, receiver hijack, unverified receivers. This checklist is organized around those measured modes.

## Used by (this standard is live)

The same six categories power minia2a's paid x402-payment security checks — every call exercises this standard:

| Check | What it runs | For whom |
|---|---|---|
| [`x402-payment-audit`](https://minia2a.uk) — **$5** | Black-box probe: forged txHash / forged signature / replay → `SAFE`/`RISK`/`NOT_X402` | Buyers verifying an endpoint **before** paying it |
| [`x402-integration-audit`](https://minia2a.uk) — **$50** | White-box deep review of x402 integration source against categories 1–6 | Sellers auditing their own gateway |
| [`minia2a-payment-audit`](https://www.npmjs.com/package/minia2a-payment-audit) — npm | Client for the $5 check (`trustCheck(url)`), pay-per-call in USDC | Any agent/developer, programmatically |
| [Methodology page](https://minia2a.uk/x402-payment-audit-methodology.html) | Public write-up of the method + live lessons | Reference |

## Quick start

1. Run **L0** (liveness) on the endpoint.
2. Walk **Categories 1–6** with `CHECKLIST.md`, using `METHODOLOGY.md` to gather answers.
3. Label each finding with its victim direction (payer→resource or resource→payer).
4. Map findings to the severity table and triage.

Or, if you just want an endpoint checked before you pay it: `npm i minia2a-payment-audit && minia2a-payment-audit check <endpoint-url>`.

## Corrections & honesty in the data

An early "76% of x402 endpoints are dead" figure was a prober bug (read payment requirements from the body after x402 v2 moved them to the `PAYMENT-REQUIRED` header). The corrected death rate is **38%**. Dead endpoints are an availability problem, not a payment-security flaw — hence the L0 pre-filter.

## License

MIT — the checklist and method are free to use, adapt, and cite. Attribution appreciated.
