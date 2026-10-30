---
name: crypto-security-hardening
description: "Harden Web3 projects: deploy, AI sec, OpSec, post-quantum."
version: 1.0.0
author: Hermes Agent (distilled from monad.xyz/blog/tags/security)
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [crypto, web3, security, smart-contract, opsec, post-quantum, ai-security]
---

# Crypto Security Hardening

## When to Use
- Reviewing/auditing a protocol's deployment or upgrade-authority setup
- Choosing or configuring AI-assisted security tooling (AI audits, fuzzing, LLM code-gen)
- Auditing a team's or individual's operational security
- Planning post-quantum cryptography migration for accounts, consensus, or apps
- After any hack/exploit post-mortem, to map the vector to a control here

Distilled from the Monad Foundation's public security writing (monad.xyz/blog/tags/security). Use it when reviewing a protocol's deployment posture, choosing AI-assisted security tooling, auditing OpSec, or planning post-quantum migration. The single load-bearing idea: **web2 attack vectors (social engineering, key theft, supply chain) cause far more lost value than smart-contract bugs** — historically ~$3.3B from control-hijacking vs ~$983M from contract vulns. Don't let a clean audit lull you into ignoring the human/ops layer.

## 1. Threat model first
Attackers want three things: **your funds, your access, your data.** Before any review, do an internal audit: which accounts are critical? who/devices touch them? how do you authenticate? where is sensitive data? what is your incident-response path and who do you call? A project without a named incident line is already compromised-shaped.

## 2. Deployment security — the EOA/deployer-key trap
The most repeated real-world loss: a stolen **deployer key that still holds upgrade authority**.
- **Zoth (Mar 2025):** stolen deployer key upgraded the proxy → **$8.4M** drained.
- **Wasabi (Apr 2026):** same pattern, deployer-as-admin in the initializer → **$5M** across multiple chains.

A hardware wallet does NOT fix this — it makes the key harder to steal but the key still carries full upgrade authority.

Rules:
- **Admins MUST be a multisig**, never an EOA. Assign upgrade authority to a multisig *in the atomic deploy transaction*.
- **Deploy through a deploy contract, atomically.** A `forge`/Hardhat deploy script is a chain of independent txs (deploy impl → deploy proxy → initialize → configure → transferOwnership). The `initialize` step can be front-run (CPIMP — Clandestine Proxy In The Middle of Proxy) before you set state; later steps can revert or be half-rerun. A deploy contract bundles all config into one atomic call: config is settled in the repo, reviewed like code, verified in-contract at deploy time, and bad state halts the deploy.
- Do not assign the deployer any rights. Seed pools/markets via the deploy contract or an unprivileged treasury account.
- Best-in-class reference: Uniswap v4's `UniswapV4DeployerCompetition` (CREATE2, deterministically pre-computed address, no lingering admin).

## 3. AI-assisted security tooling (what actually works)
This is the newest and most actionable layer. Key findings from Monad's Bugfinder / Ultrafuzz / Mythos evaluations:

**Prompting beats generic checklists.** Same task, same model, three prompts:
- baseline (feature spec only): 21 validated findings
- generic OWASP-style "be secure" checklist: 20 total but **doubled the High-severity bugs** (4 vs 2) — worse on the axis that matters
- expert (feature spec + a **real threat model** for that specific app): **12 total, ~43% fewer findings**
→ When directing an LLM to write code, give it a concrete threat model (actors, trust boundaries, what's out of scope), never a generic "write secure code".

**Diverse review prompts find disjoint bugs.** Evaluating one frontier model ("Mythos 5.1") with four prompts (open security-review, scribble property-spec, roundme rounding, threat-hunt) against one target: 26 distinct issues, and **each run found at least one the others missed**. threat-hunt (one agent per modeled threat) found the most and led recall, but the plain open review had the best F1 at <half the cost. Lesson: run multiple different lenses; no single prompt type dominates.

**AI finds more complex bugs than static analyzers, at a fraction of cost** — including long-latent zero-days — but does not replace human review. Independent research cited: >half of 2025 blockchain exploits could have been run autonomously by current AI agents. Defenders should arm with the same tooling attackers use.

**Fuzzing finds different bug classes than manual review** — the two are complementary. Ultrafuzz (agentic Solidity fuzzing orchestrator: enumerate properties → generate fuzz + invariant tests → dedup/triage → report, one command) found **27 issues in a pilot protocol, >2× any single AI audit tool**, mostly novel despite prior scans. Cost ~$150–230 in API credits, 4–6h wall-clock, vs a 4–8 week manual engagement — run it as a pre-audit step.

**Where naive agentic pipelines fail (avoid these):**
- **Context rot in long single-context skills:** core directives (preserve failing tests as evidence, strict equality to catch off-by-one, never edit production code) get lost as the campaign grows. Break work into fresh-context steps.
- **Autoresearch/scoring loops game the metric:** without complete ground truth, prompts drift to produce many "new" (false-positive) findings over a few validated ones. Keep a held-out ground-truth set (known bugs) to score against.
- **Tightly-coupled clients create false positives:** Monad's C++ exec + Rust consensus are coupled, so a bug "real" in one is often mitigated by the other — triage must test the system, not one client in isolation.

## 4. Operational security (OpSec) essentials
The Crypto OpSec Bible — the "be paranoid / Don't Trust, Verify" layer:
- **2FA:** security key (FIDO2) is the only phishing-resistant option; keep 2 for redundancy. Never SMS 2FA. If TOTP, disable cloud sync and use a dedicated clean offline device.
- **Credentials:** password manager, no reuse, cold/dedicated email for critical accounts, keep 2FA separate from the password manager, never store/ transmit creds in cleartext, never save creds in the browser.
- **Link hygiene:** never act on a link from email/X/browser/social when you can type the destination URL directly. This kills the bulk of phishing (fake Docusign, fake exchange support, homoglyph/DNSTwist domains, malicious ads, fake platform integrations, recruitment phishing, malicious bot-verification, XSS frontends).
- **Signing:** stop blind-signing. Simulate before you sign (Blockscout, Pocket Universe, Wallet Guard / ScamSniffer-class tools). **permit() signatures are the highest-risk** — they pre-authorize transfers with no per-tx confirmation.
- **Wallets:** hardware wallet for large amounts; never store it online; keep the seed offline; **rotate keys** (nobody does, it matters). Whitelist addresses; bookmark critical sites.
- **Devices/org:** separate clean machine/browser exclusively for crypto; no personal activity on work devices or vice versa; EDR/AV on employee devices; auto-updates on; VPN for browsing; never download/execute files of unknown provenance (review via dangerzone.rocks); never run unknown code locally (sandbox if unavoidable).
- **Platform lockdowns:** X — audit connected apps + delegates, remove phone number, verify active sessions, enable extra password protection. Telegram — passcode lock, 2FA, revoke sessions. Same for Discord/Slack/Google/GitHub/cloud/multisig.
- **AI phishing is real and getting better:** live deepfakes, voice-matched vishing, stylometry-matched text. Verify identity out-of-band on any first contact, especially requests to move funds or grant access.
- **Physical:** no tailgating, never leave devices unattended/open, don't trust unknown chargers (evil-twin USB-C beacons) or public WiFi (MITM, evil twin, session hijack). High-profile holders: crypto-linked kidnapping of leaders *and family* is an uptrend; protect ingress/egress points of travel, alarm the residence/office.
- **Know the incident path:** there must be a direct line to a security expert, and everyone must know how to report.

**Auditing a host that holds keys/tokens:** run the masked command-level procedure in `references/host-secret-audit.md` — it checks permissions, encrypted-at-rest stores, and key-presence patterns without ever printing secret values.

## 5. Post-quantum readiness (the 2026 urgency)
Quantum break timelines are compressing: Gidney's optimized Shor estimate dropped RSA-2048 break to <1M noisy qubits (~20× fewer than prior); industry targets cluster 2027–2029 (Algorand 2027, XRPL 2028, Ethereum L1 ~2029, Google 2029). Roughly 2–3 safe years remain. Migration order by economic value:
- **P0 — Protect user funds first.** A single-key wallet with $100M can lose its whole balance in one key recovery; the attack machine is reusable across targets. Monad's *Flexible & Upgradeable Account Authentication*: account stores auth config in state; **address stays fixed while credentials/signature scheme change** — add a PQ scheme (ML-DSA listed), retire the old, no new address, no DeFi/address-book disruption. Two policies per account: a *signing policy* (what authorizes a tx) and a *reconfiguration policy* (what authorizes changing keys/policies) + proof-of-possession of new keys. Auth checks run before EVM execution on sender state only (fits parallel execution, no shared-state contention).
- **P1 — Protect consensus:** PQ version of Cadence in progress.
- **Then:** staking (reuses P0+P1), execution-layer crypto (PQ EVM — ecrecover, P-256, BN254/BLS, KZG), zkVM proof infra (STARK-based, Flock hash-based), and finally the encrypted mempool (BTX batch threshold encryption) — lower priority since forged account control / consensus failure / monetary integrity are worse outcomes.

## 6. Source documents (monad.xyz/blog)
- `crypto-opsec-bible` — full OpSec guide (accounts, phishing, physical)
- `say-no-way-to-eoa` — deployer-key / atomic deploy contract
- `monad-bugfinder` — AI-assisted vuln discovery + triage lessons
- `writing-secure-code-with-llms` — threat-model prompt vs generic checklist
- `ultrafuzz` / `open-sourcing-ultrafuzz` — agentic Solidity fuzzing
- `evaluating-mythos-5-1-smart-contract-security` — multi-prompt frontier-model benchmark
- `post-quantum-roadmap`, `flexible-and-upgradeable-account-authentication`, `quantum-cadence-part-1` — PQ migration

> These are engineering write-ups, not a substitute for an independent audit. Treat specific numbers (findings, costs, timelines) as single-sample research signals, not forecasts.
