<div align="center">

# 📂 AI Models & Data — Cheatsheet

### Training data, supply-chain integrity, and the black-box problem

<img src="https://img.shields.io/badge/Domain-AI_Security-6366f1?style=flat-square&labelColor=0d1117" />
<img src="https://img.shields.io/badge/Focus-Supply_Chain-f43f5e?style=flat-square&labelColor=0d1117" />
<img src="https://img.shields.io/badge/Standard-ML--BOM-38bdf8?style=flat-square&labelColor=0d1117" />
<img src="https://img.shields.io/badge/Source-TryHackMe-2ecc71?style=flat-square&labelColor=0d1117" />

<i>Where model data comes from, how optimization silently breaks safety, and why you inherit what you didn't build.</i>

</div>

---

## 📑 Table of Contents

1. [Where AI Data Comes From](#1--where-ai-data-comes-from)
2. [The Dominance of Common Crawl](#2--the-dominance-of-common-crawl)
3. [Data Provenance & the ML-BOM](#3--data-provenance--the-ml-bom)
4. [Security & Privacy Implications](#4--security--privacy-implications)
5. [Building the Model: Key Training Concepts](#5--building-the-model-key-training-concepts)
6. [The Inheritance Problem](#6--the-inheritance-problem)
7. [The Black Box Problem & Model Auditing](#7--the-black-box-problem--model-auditing)
8. [Executive Summary: Five Threat Pillars](#-executive-summary-five-threat-pillars)
9. [Quick Glossary](#-quick-glossary)

---

## 1. 🪣 Where AI Data Comes From

LLMs need massive text. Developers pull from four "buckets," each with a different risk profile:

| Source | Description | Trust Profile |
| :--- | :--- | :--- |
| **Web Scraping** | Automated crawls of the public internet (Common Crawl) | 🔴 **Low** — no version control, content changes, high noise |
| **Licensed Datasets** | Purchased data (e.g. Reddit, Meta posts) | 🟠 **Medium** — unclear terms, no original-user consent |
| **Synthetic Data** | AI-generated content used to train newer AI | 🟡 **Variable** — fast-growing; risks "model collapse" over time |
| **Internal Corpora** | Company KBs, support transcripts | 🟢 **Higher** — direct control, but high liability if leaked |

---

## 2. 🌐 The Dominance of Common Crawl

**Common Crawl** — a free, public archive of web data — is the backbone of nearly every major model:

- **GPT-3:** ~60% of tokens came from a filtered version of Common Crawl.
- **DeepSeek-V3:** trained on 14.8 trillion tokens from this source.
- **LLaMA 4:** scaled to ~40 trillion tokens using similar pipelines.

---

## 3. 🧾 Data Provenance & the ML-BOM

**Data Provenance** verifies the "pedigree" of data by answering: *Where did it come from? When was it collected? Has it been modified since?*

### The ML-BOM (Machine Learning Bill of Materials)

Just as **SBOMs** became standard after the SolarWinds attack, the **ML-BOM** is the emerging standard for AI.

- **Definition:** a documented inventory of dataset sources, licenses, PII categories, and filtering decisions.
- **The reality:** an audit of 1,800 datasets found **70%** had "Unspecified" licenses and **66%** of labeled ones were categorized incorrectly.

---

## 4. 🔓 Security & Privacy Implications

Unaudited web scraping bakes "toxic" data directly into the model's weights.

### A. The PII Problem (GDPR Conflict)
Medical records, private emails, and sensitive forum posts get swept into crawls. This clashes with GDPR: regulation demands **data minimization**, but AI training thrives on **data maximization**.

### B. "Secrets" in the Weights
- Scans of Common Crawl found nearly **12,000 live API keys and passwords**.
- An attacker can craft prompts to coax a model into revealing these credentials verbatim — and there is **no patch** once the model is trained.

---

## 5. 🏗️ Building the Model: Key Training Concepts

### A. Training & Validation
- **Epoch:** one complete pass of the algorithm through the entire dataset (models train over many epochs).
- **Overfitting:** training too long so the model **memorizes** data instead of learning patterns.
  - 🛡️ **Security risk:** an overfit model reproduces sensitive training data (API keys, PII) verbatim when prompted.
- **Validation Set:** data held back from training to detect overfitting. Skipping it means shipping un-audited, unpredictable behavior.

### B. Post-Training Compression — Pruning vs. Quantization
Third-party packagers often compress models to run on weaker hardware:

| Technique | What It Does | Security Implication |
| :--- | :--- | :--- |
| **Pruning** | Removes low-contribution parameters to shrink the model | Modifies behavior post-training; changes rarely documented |
| **Quantization** | Reduces weight precision (e.g. 32-bit → 8-bit floats) | **Silently degrades safety mechanisms** — backdoor defenses built for full precision may miss triggers in quantized versions |

### C. Federated Learning — Privacy vs. Integrity
A decentralized approach: devices train locally and send **only weight updates**, not raw data.

```
Centralized:   Raw Data  ──►  Central Server   (high privacy risk)
Federated:     Weights   ──►  Central Server   (lower privacy risk)
```

- ✅ **Privacy win:** greatly reduces direct data leaks (e.g. hospitals never share raw records).
- ⚠️ **Integrity failure:** training can't be verified. Malicious participants can submit **poisoned updates (manipulated gradients)** to corrupt the global model — nearly impossible to detect at aggregation.

---

## 6. 🧬 The Inheritance Problem

Building from scratch is prohibitively expensive, so most orgs adopt a **pre-trained model** and customize it via **fine-tuning**.

- **Pre-trained Model:** a base (LLaMA, GPT) already trained on web-scale data.
- **Fine-Tuning:** further training on a smaller, task-specific dataset (medical journals, case law).

> ⚠️ **The catch — "you inherit what you didn't build."** Fine-tuning changes tone and domain knowledge but **does not sanitize the base weights.**

Three resulting vulnerabilities:

1. **Safety Alignment Erosion** — alignment can break with as few as **10 adversarial examples (under $0.20)**. Even benign fine-tuning shifts probability weights and gradually obscures the safe paths.
2. **Narrow Focus = Bigger Attack Surface** — specialization reduces resilience to unexpected tokens, making the model **more susceptible to prompt injection** (e.g. a finance-tuned model is highly compliant to prompts framed in financial jargon).
3. **Version-Tracking / Supply-Chain Gap** — fine-tuning is locked to a base checkpoint. If that checkpoint is later found to contain backdoors, every downstream derivative inherits them permanently.

---

## 7. 🕳️ The Black Box Problem & Model Auditing

Unlike traditional software (which you can disassemble and audit), an AI model is a **black box** — billions of floating-point **weights** with no human-readable record of how they were shaped.

### Sampling vs. Auditing
- Red teaming, benchmarking, and adversarial prompts are **sampling**, not auditing.
- They reveal behavior on the inputs you *tried* — not on inputs you haven't conceived.
- 🔑 **Golden rule:** trusting a model means trusting the exact pipeline that produced it.

### Model Cards — the "Nutritional Label"
Introduced by Google researchers in 2019; the closest thing to a standard AI transparency format.

| Section | What It Discloses |
| :--- | :--- |
| **Training Data** | Sources, filtering methods, known gaps/biases |
| **Intended Use** | What it was designed for — and what it *wasn't* |
| **Evaluation Results** | Performance across conditions and demographics |
| **Known Limitations** | When the model is expected to fail |
| **Bias Assessment** | Where data/evaluation may have introduced skew |
| **License** | What you're legally permitted to do |

> ⚠️ Model cards are **voluntary and unregulated**, so they're often sparse or absent. A missing/incomplete model card is a warning sign of an **unvetted asset**.

### Practical Model Auditing (Hugging Face Risks)
Public repos let anyone upload models — a major **AI supply-chain risk**. Red flags to check:

1. **Missing model cards** → lack of transparency/evaluation.
2. **Unclear licensing** → legal liability.
3. **No provenance data** → un-audited base data (PII / hardcoded-credential risk).
4. **Suspicious serializations** → malicious files (e.g. pickled Python objects) embedded in the weights format itself.

🚩 *Lab flag:* `THM{A_m0del_Stud3nt}`

---

## 🏁 Executive Summary: Five Threat Pillars

Before deploying any third-party model, evaluate these five pillars:

1. **Unaudited Pipelines** — models trained on web scrapes (Common Crawl) carry live API keys and PII that can't be patched out.
2. **Silent Safety Degradation** — optimizations like quantization can quietly break safety mechanisms.
3. **Federated Learning Vulnerability** — private, but exposed to local update / gradient poisoning at aggregation.
4. **The Inheritance Tax** — fine-tuning doesn't clean base weights; backdoors and biases are inherited whole.
5. **The Black Box Reality** — you can't inspect the weights, so integrity depends on the **Model Card** and **ML-BOM**.

---

## 📝 Quick Glossary

| Term | Meaning |
| :--- | :--- |
| **Common Crawl** | The primary text corpus behind modern LLMs |
| **Data Provenance** | Chronological record of data's origin and movement |
| **ML-BOM** | Inventory of dataset sources, licenses, PII, filters |
| **Data Minimization** | Collecting only necessary data (core GDPR principle) |
| **Synthetic Data** | Data generated by one AI to train another |
| **Epoch** | One full pass of the algorithm through the dataset |
| **Overfitting** | Memorizing training data instead of learning patterns |
| **Validation Set** | Held-back data used to detect overfitting |
| **Pruning** | Removing low-value parameters to shrink a model |
| **Quantization** | Reducing weight precision (can degrade safety) |
| **Federated Learning** | Decentralized training; vulnerable to update poisoning |
| **Pre-trained Model** | The un-audited base layer of a system |
| **Fine-Tuning** | Task-specific training that can erode safety alignment |
| **Weights** | The floating-point numbers encoding the model's logic |
| **Model Card** | Primary transparency/audit document for a model |
| **Sampling** | Prompt testing — an incomplete security proof |
| **Supply-Chain Risk** | Using open models with no documented audit trail |

---

<div align="center">

📚 <b>Learning source:</b> TryHackMe — <i>AI Security</i> path (Module 2: AI Models & Data) · Notes compiled by <a href="https://github.com/nithiya-rajesh">Nithiya Rajendran</a>

<i>"Trust the pipeline, not just the output."</i>

</div>
