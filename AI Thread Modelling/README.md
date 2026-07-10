# 🎯 AI Threat Modelling — Study Notes

Notes on modelling threats to AI/ML systems: the new assets they introduce, the data supply
chain, and a three-layer assessment method — **STRIDE-AI → MITRE ATLAS → OWASP LLM Top 10** —
worked through a running "MegaCorp" example.

> Reference notes for future revision. Summarised from a hands-on lab — not official documentation.

## Table of Contents

- [1. AI-Specific Assets & What Makes AI Different](#1-ai-specific-assets--what-makes-ai-different)
- [2. The AI Data Supply Chain](#2-the-ai-data-supply-chain)
- [3. Why STRIDE Alone Falls Short](#3-why-stride-alone-falls-short)
- [4. STRIDE Adapted for AI](#4-stride-adapted-for-ai)
- [5. MITRE ATLAS](#5-mitre-atlas)
- [6. OWASP LLM Top 10 — Component Mapping](#6-owasp-llm-top-10--component-mapping)
- [7. How the Frameworks Layer Together](#7-how-the-frameworks-layer-together)
- [Quick Self-Check](#quick-self-check)

---

## 1. AI-Specific Assets & What Makes AI Different

AI introduces asset types that don't map onto traditional categories (databases, source code,
API keys). Missing these during a threat assessment means missing whole categories of risk.

| Asset                      | What it is                                          | Why it matters                                                      |
| -------------------------- | --------------------------------------------------- | ------------------------------------------------------------------- |
| **Training Data**          | Datasets that teach the model its behaviour         | Poisoning corrupts outputs at the source; damage is baked into the model |
| **Model Weights / Params** | Numerical values encoding what the model learned    | These *are* the model — stolen = a functional copy of your AI       |
| **Embedding Vectors**      | Numerical representations for similarity/retrieval  | Used in RAG, recommendations, fraud detection; poisoning alters what the model "sees" |
| **System Prompts**         | Instructions defining behaviour, constraints, persona | Leaking reveals guardrails & business logic — a roadmap to bypass them |
| **Feature Stores**         | Preprocessed data feeding real-time inputs          | Tampering changes model inputs at inference without touching the model |
| **Model Registry / Artifacts** | Stored versions of trained models for deployment | A compromised registry lets an attacker swap in a backdoored model  |

> **Key contrasts:** a stolen *database* → rotate a credential and move on; a stolen *model* is a
> fundamentally different loss. Weights = the learned behaviour; system prompts = the control
> roadmap; poisoned training data doesn't trigger the alerts a modified DB record would.

**Two AI traits that change threat modelling:**
- **Non-deterministic behaviour** — the same input can produce different outputs, making testing,
  auditing, and incident reproduction much harder.
- **The black-box problem** — deep models lack step-through explainability, so defenders reason in
  terms of input/output behaviour and failure modes rather than code paths.

---

## 2. The AI Data Supply Chain

AI inherits traditional software supply-chain risk **and** adds a separate data supply chain.
Five stages, each a point of compromise:

1. **Data Collection** — scraping, purchased datasets, internal DBs, user content. Influence any
   source = a foothold.
2. **Cleaning & Labelling** — compromised labels teach wrong associations; corruption is invisible
   (a mislabelled set doesn't *look* corrupted).
3. **Model Training** — surviving poison gets baked into **weights**; may require a full retrain.
4. **Validation & Packaging** — a compromised **model registry** lets an attacker swap in a
   backdoored model that passes validation (trigger inputs are absent from the validation set).
5. **Inference** — for LLMs, RAG retrieval adds another injection point at query time.

> **Key difference = time.** A bad npm package is reverted in hours; a poisoned dataset may not
> surface for weeks or months, only after retraining, validation, and deployment.

**MegaCorp example:** the fraud-detection model retrains monthly. An attacker drip-feeds crafted
transactions over several cycles, gradually shifting decision boundaries until a specific fraud
pattern stops being flagged — approving fraud for weeks before anyone notices.

---

## 3. Why STRIDE Alone Falls Short

STRIDE (Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of
Privilege) is still valuable, but has documented gaps against AI:

- **Training-data integrity isn't first-class** — poisoning effects are diffuse, delayed, and
  nearly invisible; no error is thrown.
- **Adversarial manipulation blurs categories** — one attack can be part Tampering, part Spoofing,
  part Elevation of Privilege.
- **Privilege scope has expanded** — a model with tool access makes its *entire* tool-permission
  set the attacker's capability.
- **Model IP theft is a different disclosure** — the stolen asset is a trained intelligence, not a
  dataset.

Conclusion: **adapt STRIDE, don't replace it.**

---

## 4. STRIDE Adapted for AI

| STRIDE Category        | Primary AI Manifestation                       | MegaCorp Example                                       |
| ---------------------- | ---------------------------------------------- | ------------------------------------------------------ |
| **Spoofing**           | Data source impersonation (RAG injection)      | Fake policy docs injected into chatbot knowledge base  |
| **Tampering**          | Data poisoning                                 | Crafted transactions shift fraud model's boundaries    |
| **Repudiation**        | Lack of decision audit trails                  | Can't explain why fraud model approved a transaction   |
| **Info Disclosure**    | Model extraction / stealing                    | Competitor reconstructs recommendation engine via API  |
| **Denial of Service**  | Inference cost exploitation (Denial of Wallet) | Chatbot API flooded; bill spikes ~12× ($15k→$180k)     |
| **Elevation of Priv.** | Jailbreaking / guardrail bypass                | Jailbroken chatbot queries customer PII via DB tools   |

**Other threats per category:**
- **Spoofing:** model impersonation, adversarial identity attacks (fooling facial/voice auth).
- **Tampering:** model manipulation (weight/registry swap), prompt injection, feature manipulation.
- **Repudiation:** prompt/context volatility, model version ambiguity.
- **Info Disclosure:** training-data extraction, system-prompt leakage, embedding inversion.
- **DoS:** GPU exhaustion, sponge examples, training-pipeline disruption.
- **EoP:** excessive agency, tool-use exploitation, cross-plugin escalation.

> ⚠️ **Prompt injection is context-dependent:** maps to **Tampering** when altering the model's
> input, but **Elevation of Privilege** when the goal is bypassing guardrails.

**STRIDE still misses:** adversarial examples (span multiple categories), model bias/fairness
(a failure, not an "attack"), and emergent behaviours (no traditional parallel) — hence the need
for supplementary frameworks.

---

## 5. MITRE ATLAS

**ATLAS** = Adversarial Threat Landscape for Artificial-Intelligence Systems — the AI-focused
counterpart to **MITRE ATT&CK**. As of early 2026: ~16 tactics, 155 techniques, 35 mitigations,
52 case studies (check atlas.mitre.org for current counts).

**Hierarchy:** Tactic (*why*) → Technique (*how*) → Sub-technique (*specific variant*) → Mitigation.

**Key techniques:**

| Technique                      | ID          | STRIDE mapping                    |
| ------------------------------ | ----------- | --------------------------------- |
| Data Poisoning                 | AML.T0020   | Tampering                         |
| Backdoor ML Model              | AML.T0018   | Tampering                         |
| Model Extraction               | AML.T0024   | Information Disclosure            |
| Infer Training Data Membership | AML.T0025   | Information Disclosure            |
| Evade ML Model                 | AML.T0015   | Tampering + Spoofing + EoP        |
| LLM Prompt Injection           | AML.T0051   | Tampering (direct + indirect)     |

- **Direct injection** = malicious input typed into the chat; **indirect injection** = malicious
  instructions embedded in retrieved content (e.g. RAG documents) — the primary MegaCorp vector.

**Case studies:** *ShadowRay (AML.CS0023)* — real compromise of Ray AI infrastructure;
*Morris II Worm (AML.CS0024)* — self-replicating prompt-injection worm spreading via RAG email,
extracting PII with no user interaction.

**Workflow:** STRIDE (*what could go wrong*) → ATLAS (*how, + case studies*) → apply ATLAS
mitigations (e.g. for Data Poisoning: data provenance tracking, anomaly detection on training
inputs, drift monitoring).

---

## 6. OWASP LLM Top 10 — Component Mapping

Published by the OWASP GenAI Security Project. Its power here is mapping each risk to **where it
lives**, turning an architecture diagram into an assessment tool. Read it both ways:

- **Risk → Component:** "Prompt injection — where?" → inference endpoint + RAG pipeline.
- **Component → Risk:** "We're adding a vector DB — what does it inherit?" → LLM01, LLM08, LLM09.

**Highest-risk components (MegaCorp):**

| Component                    | OWASP entries it appears in                                   | Count |
| ---------------------------- | ------------------------------------------------------------ | ----- |
| **LLM Inference Endpoint**   | LLM01, LLM02, LLM05, LLM06, LLM07, LLM09, LLM10              | **7** |
| **Vector DB / RAG Pipeline** | LLM01 (indirect injection), LLM08 (embeddings), LLM09 (stale sources) | **3** |
| **Training Pipeline**        | LLM02 (sensitive data), LLM03 (supply chain), LLM04 (poisoning) | **3** |

The inference endpoint carries the highest risk concentration and needs the most hardening.

---

## 7. How the Frameworks Layer Together

Not competitors — three zoom levels of one assessment:

| Layer                | What it does                          | When you use it                          |
| -------------------- | ------------------------------------- | ---------------------------------------- |
| **STRIDE-AI**        | Categorises threats by type           | Initial ID — "what could go wrong"       |
| **MITRE ATLAS**      | Documents specific attack techniques  | Enrichment — "how exactly would they"    |
| **OWASP LLM Top 10** | Maps risks to components, prioritises | Scoping — "where does it live, how bad"  |

**STRIDE = wide-angle view · ATLAS = technical detail · OWASP = where to point the camera.**

This workflow is repeatable: run it on every new AI system, model update, or added agentic
capability. The frameworks evolve, but the methodology stays consistent.

---

## Quick Self-Check

<details>
<summary>Click for answers</summary>

- Asset that defines the model's learned behaviour → **Model Weights / Parameters**
- Asset targeted for a roadmap of your guardrails → **System Prompts**
- Two AI traits that complicate threat modelling → **non-determinism** + **black-box (low explainability)**
- Critical difference between AI and software supply-chain compromise → **time** (delayed effects)
- STRIDE category for data poisoning → **Tampering**
- STRIDE category for model extraction → **Information Disclosure**
- STRIDE category for Denial of Wallet → **Denial of Service** (OWASP LLM10)
- STRIDE category for jailbreaking → **Elevation of Privilege** (OWASP LLM06)
- AML.T0020 → **Data Poisoning**; AML.T0024 → **Model Extraction**; AML.T0051 → **LLM Prompt Injection**
- Direct vs. indirect prompt injection → typed input vs. malicious instructions in retrieved content
- Component appearing in 7 of 10 OWASP entries → **LLM Inference Endpoint**
- The three assessment layers in order → **STRIDE → ATLAS → OWASP**

</details>

---

*Personal learning notes for revision — not official documentation.*
