# SOMAGRAPH — Threat Model & Security Posture

> Version 1.0 · May 2026

---

## 1. Scope

This document covers the security architecture, threat model, and mitigation strategy for the Somagraph protocol across all five trust zones: user device, edge perimeter, compute layer, persistent storage, and on-chain program.

The analysis follows STRIDE methodology and traces data flow across trust boundaries as prescribed by the security-auditor framework.

---

## 2. Asset Inventory

| Asset                      | Classification                     | Location                            | Sensitivity |
| -------------------------- | ---------------------------------- | ----------------------------------- | ----------- |
| Raw biomarker data         | PHI (Protected Health Information) | Zone 0 (client), Zone 1 (ephemeral) | CRITICAL    |
| Encrypted panel ciphertext | Encrypted PHI                      | Zone 3 (PostgreSQL)                 | HIGH        |
| Panel SHA-256 hash         | Non-reversible identifier          | Zone 4 (Solana)                     | LOW         |
| User wallet address        | Pseudonymous PII                   | Zone 1-4                            | MEDIUM      |
| PhenoAge / longevity score | Derived health metric              | Zone 1-4                            | MEDIUM      |
| Protocol treasury USDC     | Financial asset                    | Zone 4 (Solana multisig)            | CRITICAL    |
| $SOMA token balance        | Financial asset                    | Zone 4 (user wallets)               | HIGH        |
| AI inference prompts       | Operational data                   | Zone 2 (ephemeral)                  | LOW         |

---

## 3. Trust Boundary Diagram

```
┌────────────────────────────────────────────────────────────┐
│ ZONE 0 — USER DEVICE (untrusted)                          │
│   Browser / Telegram client                                │
│   Wallet signing keys                                      │
│   Client-side encryption (wallet pubkey → AES-256-GCM)     │
│                                                            │
│   BOUNDARY ═══════════════════════════════════════════      │
│                                                            │
│ ZONE 1 — EDGE PERIMETER (Cloudflare Workers)               │
│   TLS termination                                          │
│   Rate limiting (token bucket, 100 req/min/IP)             │
│   Wallet signature verification (Ed25519)                  │
│   Free-trial oracle (wallet age + email + IP fingerprint)  │
│   Geofence (OFAC + restricted jurisdictions)               │
│   Decrypt → process → re-encrypt (stateless, no persist)   │
│                                                            │
│   BOUNDARY ═══════════════════════════════════════════      │
│                                                            │
│ ZONE 2 — COMPUTE (ephemeral, no persistence)               │
│   AI Narrator: Cloud Function (stateless invocation)       │
│   PhenoAge Engine: embedded in Edge (pure math)            │
│   No biomarker data survives after response generation     │
│                                                            │
│   BOUNDARY ═══════════════════════════════════════════      │
│                                                            │
│ ZONE 3 — STORAGE (encrypted at rest)                       │
│   PostgreSQL: AES-256-GCM ciphertext only                  │
│   Decryption key: user wallet pubkey (never stored)        │
│   Right-to-delete: wallet-signed message → full purge      │
│                                                            │
│   BOUNDARY ═══════════════════════════════════════════      │
│                                                            │
│ ZONE 4 — ON-CHAIN (public, immutable)                      │
│   SHA-256(normalized_markers_json) only                    │
│   No PII, no raw biomarker values                          │
│   Attestation PDAs, burn records, treasury operations      │
└────────────────────────────────────────────────────────────┘
```

---

## 4. STRIDE Threat Analysis

### 4.1 Spoofing

| Threat                                               | Target            | Severity | Mitigation                                                                                                                    |
| ---------------------------------------------------- | ----------------- | -------- | ----------------------------------------------------------------------------------------------------------------------------- |
| S-1: Fake wallet signature on analysis request       | Edge Gateway      | HIGH     | Ed25519 signature verification against presented wallet pubkey. Replay protection via nonce.                                  |
| S-2: Sybil attack to farm free trials                | Free-trial oracle | MEDIUM   | Triple gate: wallet age ≥ 7 days, email verification, IP fingerprint. Rate limit 1 trial per wallet forever.                  |
| S-3: Impersonation of protocol authority for buyback | Solana Program    | CRITICAL | Authority pubkey hardcoded in ProtocolState PDA. Signer check in `usdc_buyback_burn` instruction. Multisig (3-of-5) required. |

### 4.2 Tampering

| Threat                                              | Target         | Severity | Mitigation                                                                                                                                   |
| --------------------------------------------------- | -------------- | -------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| T-1: Modified biomarker values before scoring       | Edge Gateway   | HIGH     | Client-side signing of marker payload. Hash verification at Edge before scoring.                                                             |
| T-2: Manipulated PhenoAge output                    | Scoring Engine | MEDIUM   | Engine is pure deterministic TypeScript. Same inputs always produce same output. On-chain attestation hash enables third-party verification. |
| T-3: Altered AI narrative to include medical claims | AI Narrator    | HIGH     | System prompt injection guardrails. Output filtered for banned phrases (see §6). Post-generation compliance check.                           |

### 4.3 Repudiation

| Threat                                  | Target               | Severity | Mitigation                                                                                      |
| --------------------------------------- | -------------------- | -------- | ----------------------------------------------------------------------------------------------- |
| R-1: User denies analysis was performed | On-chain attestation | LOW      | Immutable `AnalysisRecorded` event emitted on Solana. Transaction signature serves as receipt.  |
| R-2: Protocol denies burn was executed  | Burn mechanism       | LOW      | `TokensBurned` event with amount, cumulative total, and timestamp. Publicly auditable on-chain. |

### 4.4 Information Disclosure

| Threat                                            | Target       | Severity | Mitigation                                                                                                                                                                 |
| ------------------------------------------------- | ------------ | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| I-1: Biomarker data leaked from database          | PostgreSQL   | CRITICAL | AES-256-GCM encryption at application layer. Database encryption at rest. No plaintext biomarkers stored. Decryption requires wallet private key (never held by protocol). |
| I-2: AI inference logs expose patient data        | Vertex AI    | HIGH     | Disable logging on inference requests. Use data residency controls. Process in ephemeral Cloud Functions with no persistent storage.                                       |
| I-3: On-chain hash reversal to recover biomarkers | Solana       | LOW      | SHA-256 is one-way. Panel JSON includes sufficient entropy (50+ markers, floating-point values) to resist dictionary attacks.                                              |
| I-4: Side-channel leakage via timing analysis     | Edge Gateway | LOW      | Constant-time hash comparison. Rate limiting obscures per-request timing.                                                                                                  |

### 4.5 Denial of Service

| Threat                                       | Target             | Severity | Mitigation                                                                                                                                    |
| -------------------------------------------- | ------------------ | -------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| D-1: Volumetric attack on Edge Gateway       | Cloudflare Workers | MEDIUM   | Cloudflare DDoS protection (L3/L4/L7). Token bucket rate limiting (100 req/min/IP). WAF rules for bot detection.                              |
| D-2: AI inference cost attack (mass uploads) | Vertex AI          | HIGH     | Rate limit: 10 analyses per wallet per day. Cost cap: $500/day circuit breaker on inference spend. Queue-based processing with backpressure.  |
| D-3: Solana program spam                     | somagraph-core     | MEDIUM   | Each instruction requires SOL for rent + transaction fee. Attestation PDA creation requires user to pay rent. Economic disincentive for spam. |

### 4.6 Elevation of Privilege

| Threat                                            | Target            | Severity | Mitigation                                                                                                                                                                    |
| ------------------------------------------------- | ----------------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| E-1: Unauthorized buyback execution               | Solana Program    | CRITICAL | Authority signer check on `usdc_buyback_burn`. Authority is a 3-of-5 multisig. Program upgrade authority can be locked post-audit.                                            |
| E-2: Free-trial bypass via wallet recycling       | Free-trial oracle | MEDIUM   | Wallet age gate (7 days minimum). Email binding. IP rate limiting. Persistent anti-sybil log in PostgreSQL.                                                                   |
| E-3: Admin SDK bypassing database access controls | PostgreSQL        | HIGH     | Row-level security policies enforced at database level. Application-layer authorization checks redundant with DB-level policies. No service account with unrestricted access. |

---

## 5. Solana Program Security

### 5.1 Account Validation

All instruction contexts use Anchor's account validation constraints:

- PDA seeds enforce deterministic account derivation
- `has_one` constraints validate cross-account relationships
- Signer checks on all mutable operations
- Overflow protection via `checked_add` arithmetic

### 5.2 Known Attack Vectors and Mitigations

| Vector                | Mitigation                                                                                            |
| --------------------- | ----------------------------------------------------------------------------------------------------- |
| Reinitialization      | `initialized` boolean check in `initialize_protocol`. Fails if already set.                           |
| PDA collision         | Attestation PDAs derived from `[b"attestation", wallet, nonce]`. Nonce is monotonically incrementing. |
| Integer overflow      | All arithmetic uses `checked_add` / `checked_mul`. Custom `Overflow` error on failure.                |
| Unauthorized burn     | User must be signer of `burn_payment`. CPI to SPL burn requires token account authority.              |
| Front-running buyback | Buyback uses Jupiter with slippage protection. 24h cadence reduces MEV surface.                       |

### 5.3 Audit Plan

| Phase       | Partner               | Scope                           | Timeline            |
| ----------- | --------------------- | ------------------------------- | ------------------- |
| Pre-mainnet | OtterSec or Halborn   | Full Anchor program review      | Before token launch |
| Post-launch | Bug bounty (Immunefi) | Public vulnerability disclosure | Ongoing             |
| Quarterly   | Internal + community  | Dependency audit, Cargo audit   | Every 90 days       |

---

## 6. Content Safety — Banned Phrases

The AI Narrator output is filtered against a blocklist to prevent medical claims:

| Banned             | Replacement                 |
| ------------------ | --------------------------- |
| "guaranteed yield" | "performance-based reward"  |
| "passive income"   | "protocol participation"    |
| "cure cancer"      | "biomarker monitoring"      |
| "reverse aging"    | "biological-age tracking"   |
| "medical advice"   | "wellness insight"          |
| "diagnose"         | "flag for physician review" |
| "treatment plan"   | "action prioritization"     |

Post-generation regex scan rejects any output containing banned phrases. Rejected outputs are regenerated with stricter system prompt constraints.

---

## 7. Incident Response

### 7.1 Severity Classification

| Level         | Definition                                    | Response Time |
| ------------- | --------------------------------------------- | ------------- |
| P0 — Critical | Data breach, fund loss, program exploit       | 1 hour        |
| P1 — High     | Service outage, AI safety violation           | 4 hours       |
| P2 — Medium   | Degraded performance, partial feature failure | 24 hours      |
| P3 — Low      | Cosmetic issues, non-critical bugs            | 72 hours      |

### 7.2 Response Procedure

1. **Detect** — Monitoring alerts via Axiom + PagerDuty
2. **Contain** — Geofence expansion, rate limit reduction, program freeze (if P0)
3. **Investigate** — Root cause analysis with full audit trail
4. **Remediate** — Patch deployment with expedited CI/CD
5. **Communicate** — Public post-mortem within 48 hours of resolution
6. **Harden** — Updated threat model and regression tests

### 7.3 Program Emergency

The Solana program includes an upgrade authority held by the protocol multisig. In a critical exploit scenario:

1. Multisig signers convene (3-of-5 threshold)
2. Program is upgraded with a patch or paused
3. Affected users notified via Telegram and X
4. Post-mortem published with full timeline

After audit completion and 90 days of stable operation, the upgrade authority can be permanently locked (burned) via governance vote.

---

## 8. Compliance Considerations

| Framework            | Applicability                                 | Status                                             |
| -------------------- | --------------------------------------------- | -------------------------------------------------- |
| HIPAA                | Blocked (US geofenced in V0)                  | Future evaluation post-entity formation            |
| GDPR Article 9       | Limited (EU access restricted pending review) | Compliance review in progress                      |
| Singapore PDPA       | Applicable (SE Asia launch market)            | Compliant by design (encryption + right-to-delete) |
| Indonesia PP 71/2019 | Applicable (target market)                    | Compliant by design                                |

---

_Threat model version 1.0. Updated 2026-05-05. Review scheduled: 2026-08-05._
