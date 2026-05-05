# SOMAGRAPH — Protocol Whitepaper

> Version 1.0 · May 2026 · somagraph.bio

---

## Abstract

Biological age is the single most predictive metric of future health outcomes, yet it remains inaccessible to the vast majority of the global population. Premium longevity services gate the science behind subscription paywalls exceeding $500 per year. Mainstream medicine treats lab values as binary signals and ignores the optimization frontier where most people live. Somagraph closes that gap.

The Somagraph protocol accepts unstructured medical documents, extracts biomarker data through multimodal AI, computes biological age using the Levine PhenoAge formula and the Klemera-Doubal method, and returns a prioritized action plan in plain language. Each analysis costs roughly $5 or 1,000 $SOMA tokens. Every token payment is permanently burned. Every USDC payment triggers a 50% buyback-and-burn cycle. The result is a protocol where usage creates structural deflation.

This paper describes the scientific foundations, protocol architecture, token mechanics, threat model, and roadmap of the Somagraph system.

---

## 1. The Problem

### 1.1 The Broken Feedback Loop

A standard blood panel costs between $150 and $500. It measures dozens of biomarkers. The patient receives a multi-page PDF. The physician scans the document for values flagged as abnormal, addresses those values with binary advice ("watch your cholesterol"), and schedules a follow-up in six months. The patient leaves with no understanding of where their results sit relative to optimal ranges, how those results interact, or what concrete actions would shift their trajectory.

This feedback loop fails because it was designed for disease detection, not health optimization. The gap between "you have no disease" and "you are aging well" is where the vast majority of preventable decline occurs.

### 1.2 Existing Solutions and Their Limitations

The market has responded with premium services. InsideTracker charges $589 per year for a biomarker dashboard and personalized recommendations. Function Health charges $499 per year with a waitlist exceeding 100,000 users. Levels charges $200 per month for glucose monitoring alone. Each of these products serves a narrow demographic: wealthy, health-conscious Americans between 30 and 55.

The remaining population, numbering in the hundreds of millions of individuals who receive annual lab panels through employers, insurers, or national health systems, has no tool that interprets those results at the optimization frontier.

### 1.3 The Convergence Window

Three macro-trends converge between 2024 and 2026. The longevity narrative has entered mainstream culture through Netflix documentaries, bestselling books, and podcasts with audiences exceeding five million subscribers. The collapse of 23andMe has created demand for self-sovereign biological data ownership. Multimodal language models have reached the capability threshold required to parse unstructured medical documents with clinical-grade accuracy.

The window for an affordable, global, community-owned longevity intelligence layer is open. Somagraph fills that slot.

---

## 2. Scientific Foundations

### 2.1 Levine PhenoAge

The PhenoAge formula was published in _Aging Cell_ in 2018 by Morgan Levine and colleagues. It estimates biological age from nine routine blood biomarkers using a composite mortality risk score derived from Gompertz hazard modeling on NHANES cohort data. The formula has accumulated over 800 citations and has been validated across multiple independent cohorts.

The nine input biomarkers:

| Marker                      | Unit   | Coefficient |
| --------------------------- | ------ | ----------- |
| Albumin                     | g/L    | −0.0336     |
| Creatinine                  | µmol/L | +0.0095     |
| Glucose (fasting)           | mmol/L | +0.1953     |
| C-reactive protein (log)    | mg/dL  | +0.0954     |
| Lymphocyte percentage       | %      | −0.0120     |
| Mean cell volume            | fL     | +0.0268     |
| Red cell distribution width | %      | +0.3306     |
| Alkaline phosphatase        | U/L    | +0.0019     |
| White blood cell count      | K/µL   | +0.0554     |

The computation pipeline:

1. Compute linear combination of weighted biomarkers plus age term
2. Derive 10-year mortality risk via Gompertz baseline hazard
3. Invert the mortality model to extract PhenoAge

The implementation is deterministic. The same inputs produce the same output on every invocation. No AI sampling variance contaminates the score.

### 2.2 Klemera-Doubal Method

For biomarkers not captured by PhenoAge (lipids, hormones, vitamins, inflammatory markers), Somagraph applies the Klemera-Doubal method. Published in _Mechanisms of Ageing and Development_ in 2006, KDM estimates biological age offset by fitting observed biomarker values against age-dependent regression lines derived from population reference data.

Somagraph's KDM implementation covers: LDL cholesterol, HDL cholesterol, ApoB, total testosterone, TSH, vitamin D, and homocysteine. The composite biological age is a weighted blend: 70% PhenoAge (mortality-calibrated) and 30% KDM (broader marker coverage).

### 2.3 Separation of Computation and Interpretation

The scoring engine computes biological age through fixed mathematical formulas. The AI layer interprets and explains those scores in natural language. This separation is deliberate. The score is reproducible and auditable. The narrative is contextual and personalized. Neither depends on the other for correctness.

---

## 3. Protocol Architecture

### 3.1 Analysis Pipeline

The pipeline accepts three input formats: PDF documents (Quest, Labcorp, hospital discharge, employer checkup), photographs of paper lab results, and manual marker entry.

1. **Format Detection** — classify input as PDF, image, or structured JSON
2. **Vision OCR** — multimodal AI extracts every biomarker as structured JSON with name, value, unit, and reference range
3. **Normalization** — alias resolution (e.g., "WBC" → white blood cell count), unit conversion to canonical format, outlier flagging
4. **Scoring** — PhenoAge computation (all 9 markers required), KDM computation (any subset of 7 supplementary markers), composite blending
5. **Narration** — AI generates plain-English interpretation with top-3 prioritized interventions ranked by estimated biological age impact
6. **Attestation** — SHA-256 hash of normalized markers recorded on Solana as an immutable proof of analysis

Total pipeline latency: under 60 seconds from upload to result.

### 3.2 Privacy Architecture

No raw biomarker data is stored in plaintext anywhere in the Somagraph infrastructure. The privacy model operates across four zones:

- **Zone 0 (User Device):** Panel data encrypted with the user's wallet public key before transmission
- **Zone 1 (Edge Perimeter):** Data decrypted, processed, and re-encrypted in a stateless worker. No persistent storage of plaintext.
- **Zone 2 (Compute):** AI inference and scoring are ephemeral. No biomarker data persists after the response is generated.
- **Zone 3 (Storage):** PostgreSQL stores AES-256-GCM ciphertext. Only the user's wallet can decrypt.
- **Zone 4 (On-Chain):** Solana stores only SHA-256 hashes. No PII, no raw values.

Right to delete: a wallet-signed message triggers full purge of all encrypted records.

### 3.3 On-Chain Layer

The Somagraph Anchor program on Solana Mainnet provides four instructions:

1. `initialize_protocol` — one-time setup of treasury and configuration PDAs
2. `record_analysis` — immutable attestation linking wallet, panel hash, PhenoAge, longevity score, and timestamp
3. `burn_payment` — SPL Token-2022 burn of 1,000 $SOMA
4. `usdc_buyback_burn` — cron-triggered USDC → SOMAGRAPH swap via Jupiter, followed by burn

The program stores attestation records in PDAs derived from `[b"attestation", wallet, nonce]`. Each user's analysis history is enumerable on-chain without exposing any health data.

---

## 4. Token Mechanics

### 4.1 Supply Parameters

| Parameter        | Value                          |
| ---------------- | ------------------------------ |
| Total Supply     | 1,000,000,000                  |
| Decimals         | 6                              |
| Standard         | SPL Token-2022                 |
| Extensions       | BurnAuthority, MetadataPointer |
| Mint Authority   | Disabled at launch             |
| Freeze Authority | Disabled at launch             |

### 4.2 Distribution

| Allocation                     | Tokens      | Percentage | Vesting                       |
| ------------------------------ | ----------- | ---------- | ----------------------------- |
| Community (PumpFun fairlaunch) | 750,000,000 | 75%        | None — immediately liquid     |
| Protocol Treasury              | 150,000,000 | 15%        | 4-year linear, 6-month cliff  |
| Ecosystem & Integrations       | 75,000,000  | 8%         | Milestone-based release       |
| Team                           | 25,000,000  | 2%         | 4-year linear, 12-month cliff |

No presale. No insider allocation. No seed round. All community supply enters the market through PumpFun's bonding curve.

### 4.3 Dual-Burn Mechanism

Every paid analysis triggers one of two burn vectors:

**Vector 1 — Direct Token Burn:** User pays 1,000 $SOMA. The entire amount is permanently destroyed via SPL burn instruction.

**Vector 2 — USDC Buyback-and-Burn:** User pays $5 USDC. The protocol splits the fee 50/50. Half enters the protocol treasury (operations, compute, partnerships). Half is used to buy $SOMA on Jupiter every 24 hours, and the acquired tokens are burned.

The user always pays whichever option is cheaper in USD terms. At low token prices, token payment dominates (users get access at sub-$1 costs). At high token prices, USDC payment dominates (users pay a fixed $5, and the buyback maintains demand).

### 4.4 Modeled Deflation

At 100 analyses per day with 80% token / 20% USDC split:

- Direct burn: 80,000 $SOMA per day
- Buyback burn: ~50,000 $SOMA per day (at $0.001/token)
- Annual burn: ~47.5M tokens (4.75% of total supply)

At 1,000 analyses per day: ~475M tokens burned annually (47.5% of supply). The token is structurally deflationary when the product has usage.

---

## 5. Competitive Positioning

Somagraph occupies the intersection of three axes that no existing product covers simultaneously: affordability (under $5 per analysis), global access (no waitlist, no geographic lock), and scientific rigor (peer-reviewed formulas, not AI hallucination).

InsideTracker offers a comparable dashboard at 118x the per-analysis cost. Function Health offers broader marker coverage but requires a six-figure waitlist and their proprietary lab partner. ChatGPT offers free interpretation but with no tracking, no cohort comparison, no validated scoring formula, and no community.

The deepest long-term moat is the cohort dataset. After 10,000 analyses, Somagraph can report: "you are in the 78th percentile of 32-year-old males in our community." That insight does not exist anywhere else in a public, community-owned form.

---

## 6. Regulatory Posture

### 6.1 Three Hard Constraints

Somagraph does not diagnose. It does not treat. It does not replace a physician. Every result page carries a permanent banner: "Wellness insight, not medical advice. Discuss with your physician."

### 6.2 Geographic Restrictions (V0)

- **Allowed:** Southeast Asia, Latin America, EMEA (most jurisdictions), Australia
- **Limited:** EU (pending GDPR Article 9 compliance review)
- **Blocked:** United States (HIPAA), United Kingdom (Data Protection Act), China (Cybersecurity Law), OFAC-sanctioned countries

US access will be evaluated after entity formation and HIPAA-compliance audit.

---

## 7. Roadmap

| Phase            | Timeline    | Milestone                                                                                                                          |
| ---------------- | ----------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| V0 — MVP         | Weeks 1-2   | Landing page, PDF upload, PhenoAge scoring, slider playground, wallet connect, USDC payments, famous bio ages, Twitter share cards |
| V0.5 — Token     | Week 3      | $SOMA PumpFun launch, burn gate, live burn dashboard                                                                               |
| V1 — Cohort      | Weeks 4-8   | Cohort percentile comparison, panel history tracking, Telegram bot, Apple Health import                                            |
| V2 — Integration | Months 3-4  | Quest Diagnostics API, multi-language support, ethnicity-specific calibration                                                      |
| V3 — Research    | Months 5-12 | Anonymous cohort research participation, genetic risk overlay, longitudinal studies                                                |

---

## 8. Risk Disclosure

$SOMA is a utility token. It is not a security, not an investment contract, and confers no equity. Token holders own no claim on Somagraph protocol revenue. Past performance of similar projects does not predict Somagraph's outcome. The protocol does not provide medical advice. Smart contract risk is real: bugs, exploits, and economic failures are possible. Audit reports will be published before mainnet. Do not deploy capital you cannot afford to lose.

---

_Whitepaper version 1.0. Somagraph, May 2026._
