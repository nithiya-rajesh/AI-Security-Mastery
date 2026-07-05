<div align="center">

# 🤖 AI / ML Security Threats — Cheatsheet

### Offensive & defensive security for AI systems, from foundations to threat modeling

<img src="https://img.shields.io/badge/Domain-AI_Security-6366f1?style=flat-square&labelColor=0d1117" />
<img src="https://img.shields.io/badge/Framework-MITRE_ATLAS-f43f5e?style=flat-square&labelColor=0d1117" />
<img src="https://img.shields.io/badge/Source-TryHackMe-38bdf8?style=flat-square&labelColor=0d1117" />

<i>Study notes on how AI works, how it's attacked, and how it's defended — condensed for fast reference.</i>

</div>

---

## 📑 Table of Contents

1. [AI & ML Foundations](#1--ai--ml-foundations)
2. [Large Language Models (LLMs)](#2--large-language-models-llms)
3. [AI Security Threats](#3--ai-security-threats)
4. [Defensive AI & Securing the Model](#4--defensive-ai--securing-the-model)
5. [Practical: AI as a Cyber Assistant](#5--practical-ai-as-a-cyber-assistant)
6. [Summary & Key Takeaways](#6--summary--key-takeaways)
7. [Quick Glossary](#-quick-glossary)

---

## 1. 🧠 AI & ML Foundations

### AI vs. ML

- **Artificial Intelligence (AI):** Systems that simulate human intelligence to perform tasks such as reasoning, problem-solving, and creativity.
- **Machine Learning (ML):** A subfield of AI — the ability of a computer to learn from data without explicit instructions, improving over time.

### The Machine Learning Lifecycle

ML development is an iterative loop, not a one-time process:

1. **Define Problem** — identify the goal (e.g. "Is this email spam?").
2. **Data Collection** — gather raw information.
3. **Data Cleaning & Feature Engineering** — prepare data and extract meaningful patterns.
   - ⚠️ **Overfitting:** the model learns the training data *too* well and fails to generalize to new, unseen data.
4. **Model Training** — run data through the selected algorithm.
5. **Evaluation & Tuning** — optimize performance based on feedback.
6. **Deployment** — put the model into production.
7. **Monitoring** — ongoing accuracy checks; triggers retraining if performance drops.

### ML Algorithms — How It Works

> An **ML algorithm** is the math used to learn; an **ML model** is the final trained output.

The 3 core components of an algorithm:

- **Decision Process** — logic that makes predictions from input.
- **Error Function** — evaluates how "wrong" a prediction was.
- **Model Optimization** — fine-tunes the math to minimize error in the next round.

### The 4 Categories of ML

| Category | Data Type | How It Works | Example |
| :--- | :--- | :--- | :--- |
| **Supervised** | Labeled | Uses "answer keys" to learn mapping | Spam detection, price prediction |
| **Unsupervised** | Unlabeled | Finds hidden patterns/clusters on its own | Customer segmentation |
| **Semi-Supervised** | Mixed | Uses a small labeled set to guide a large unlabeled one | Document classification |
| **Reinforcement** | Trial & Error | Uses a reward/penalty system | Robotics, game AI (AlphaGo) |

### Neural Networks — The Digital Brain

Neural networks replicate the brain's behavior using:

- **Neurons (Nodes):** individual processing units.
- **Synapses (Connections):** links between neurons.
- **Weights:** the importance of a given input (e.g. an email's body text may carry more weight than its subject line).

**Layers:** Input (receives raw data) → Hidden (detects edges, curves, features) → Output (final prediction/classification).

### Deep Learning (DL)

A specialized subfield of ML using neural networks with **more than three layers**.

| Feature | Machine Learning (ML) | Deep Learning (DL) |
| :--- | :--- | :--- |
| **Human Interaction** | Needs humans to label features | Self-learning; finds features itself |
| **Data Type** | Structured / labeled | Unstructured / raw |
| **Scalability** | Harder to scale | Thrives on massive datasets ("Scalable ML") |
| **Structure** | Simpler algorithms | Multi-layered neural networks |

---

## 2. 🗨️ Large Language Models (LLMs)

### What Is an LLM?

Advanced deep-learning models built to process and generate human-like text.

- **Core mechanism:** predict the **next word** in a sequence.
- **Scale:** trained on massive datasets — GPT-3's training data would take a human ~2,600 years to read.
- **Parameters:** numerical "puzzle pieces" the model auto-adjusts to capture language patterns.

### The Training Process

**A. Pre-training (Learning):** processes trillions of words from the internet, books, and code.
- **Backpropagation:** when the model predicts wrong, this algorithm calculates the error and adjusts parameters to make the correct word more likely next time.
- **Unsupervised:** learns patterns directly from raw data, no human labels.

**B. RLHF — Reinforcement Learning from Human Feedback (Alignment):** humans review responses and flag unhelpful, biased, or incorrect answers; the model is tuned to be safe and useful.

### The Transformer Revolution (2017)

Modern LLMs (ChatGPT, LLaMA, DeepSeek) exist because of **Transformer neural networks**, introduced in Google's paper *"Attention Is All You Need."*

- **Parallel processing:** transformers read whole sentences at once instead of word-by-word — much faster.
- **Attention mechanism:** assigns importance ("attention scores") to words to understand context.
  - *Example:* In *"The bank approved the loan because **it** was stable,"* attention helps the model know **"it"** = the bank, not the loan.

### Hardware — Why GPUs

- **GPUs** are built for **parallel processing** (unlike CPUs), running trillions of operations simultaneously — making billion-parameter training feasible.

### The AI Hierarchy

**AI** (mimic human intelligence) → **ML** (learn patterns from data) → **DL** (multi-layered neural networks) → **LLMs** (DL models using Transformers to generate text).

---

## 3. ⚔️ AI Security Threats

### The MITRE ATLAS Framework

Where **ATT&CK** maps traditional cyber attacks, **MITRE ATLAS** (Adversarial Threat Landscape for AI Systems) maps the steps an attacker takes to compromise **AI systems** — a specialized guide for AI-specific threats.

### Category 1 — Vulnerabilities in AI Models

New threats introduced by the technology itself:

| Vulnerability | Definition | Example |
| :--- | :--- | :--- |
| **Prompt Injection** | Overriding a model's original instructions via crafted input | Forcing a chatbot to reveal training data or system details |
| **Data Poisoning** | Manipulating training data to create a biased or backdoored model | Corrupting a spam filter's dataset so it lets certain spam through |
| **Model Theft** | Cloning a model via API queries or unauthorized access | Using API outputs to train a duplicate that mimics the original |
| **Privacy Leakage** | Model inadvertently reveals sensitive training data | An AI trained on medical records leaking patient names |
| **Model Drift** | Performance decay over time as data/environment change | Fraud detection failing after attackers change tactics |

### Category 2 — Enhanced Attacks

Existing attacks made faster and harder to detect with generative AI:

- **🦠 Malware Generation** — GenAI writes malicious code instantly, lowering the skill bar for less-skilled attackers.
- **🎭 DeepFakes** — replicating a person's voice/likeness for social engineering (e.g. impersonating a CEO on a call to authorize a fraudulent transfer).
- **🎣 Advanced Phishing** — LLMs produce fluent, context-aware emails, erasing the old "red flags" (broken grammar). Attackers use **prompt engineering** to bypass safety filters.

---

## 4. 🛡️ Defensive AI & Securing the Model

### The Business Case (IBM Cost of a Data Breach)

- **~$2.2M** average savings per breach for organizations using AI in security.
- **108 days** faster to identify and contain a breach.

### Four Pillars of AI-Enhanced Defense

| Capability | Defensive Application | Example |
| :--- | :--- | :--- |
| **Analysis** | Spot anomalies in massive datasets at speed | Microsoft Defender, Splunk |
| **Prediction** | Automate workflows; catch complex phishing | Blocking AI-generated phishing pre-inbox |
| **Summarization** | Condense long logs/reports into "cliff notes" | Summarizing 1,000+ log lines to find root cause |
| **Investigation** | Natural-language triage & attack-vector ideation | Threat hunting for avenues a human might miss |

### The "Secure AI" Mandate

> ⚠️ Only **~24%** of generative-AI initiatives are currently secured. Adding AI without securing it creates new attack surface.

How to secure AI systems:

1. **Access Control** — RBAC + MFA for anyone interacting with the model.
2. **Privacy & Data Protection** — treat training data as sensitive; encrypt at rest and in transit.
3. **Industry Standards** — follow frameworks like **ISO/IEC 27090** for AI-security guidance.
4. **Monitoring & Explainability** — watch for model drift, bias, and manipulation; use **SHAP** and **LIME** to understand *why* a model made a decision.

---

## 5. 🛠️ Practical: AI as a Cyber Assistant

AI acts as a force multiplier for analysts across four areas.

### Automated Log Analysis
Translates raw logs into plain English during high-pressure incidents.

```
Apr 22 11:45:09 ubuntu sshd[1245]: Failed password for invalid user admin from 203.0.113.55 port 56231 ssh2
```
→ Failed login for a non-existent user `admin` from `203.0.113.55` — likely brute-force or an automated scanner.

### Phishing Detection
AI flags look-alike domains (`m1crosoft365-security.com`), artificial urgency ("within 12 hours"), generic greetings ("Dear User"), and mismatched malicious links.

### Threat Hunting Brainstorming
- **Unusual outbound traffic** → possible C2 / exfiltration.
- **Abnormal process execution** → atypical PowerShell / living-off-the-land binaries (persistence).
- **Privilege escalation** → unauthorized AD user creation or admin-group changes.

### Content Generation (Regex & Logic)
Generate detection patterns on demand — drop straight into `grep`, a SIEM rule, or `fail2ban`:

```regex
Failed password for (invalid user\s+)?\S+ from \d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3} port \d+ ssh2
```

### Technical Retrieval

| Query | Value |
| :--- | :--- |
| DNS over HTTPS (DoH) port | 443 |
| SYN flood timeout (Linux default) | 60 seconds |
| Windows ephemeral port range size | 16,384 (49152–65535) |

---

## 6. 🎯 Summary & Key Takeaways

**The AI Paradox — Weapon vs. Shield:**

- ⚔️ **Offensive:** AI makes existing attacks (phishing, malware) faster and more fluent, *and* introduces new ones (prompt injection, data poisoning, model theft).
- 🛡️ **Defensive:** To fight AI-armed attackers, defenders must use AI too — for faster log analysis, creative threat hunting, and automated triage.
- **The golden rule:** adopt AI **securely from day one** (encryption, RBAC, MFA), or it introduces more risk than it solves.

> The goal for a security professional isn't to fear AI, but to **understand, harness, and secure it.**

---

## 📝 Quick Glossary

| Term | Meaning |
| :--- | :--- |
| **Feature Engineering** | Selecting the right variables for a model to study |
| **Overfitting** | "Memorizing" data instead of "learning" it |
| **Backpropagation** | The error-correction algorithm used in training |
| **Transformer** | The neural-network architecture powering modern LLMs |
| **Attention Score** | How much focus a model puts on a word for context |
| **Token** | A unit of text (word or word-part) an LLM processes |
| **Generative AI** | AI that creates original content vs. just classifying |
| **MITRE ATLAS** | The framework mapping adversarial threats to AI systems |
| **Model Inversion** | Reconstructing training data from model outputs (privacy leak) |
| **Model Drift** | Performance decay as real-world data diverges from training |
| **SHAP / LIME** | Explainability tools that interpret ML decisions |
| **ISO/IEC 27090** | International standard for AI security |
| **Threat Hunting** | Proactively searching for lurking attackers/malware |
| **Incident Triage** | Prioritizing which alerts need immediate action |

---

<div align="center">

📚 <b>Learning source:</b> TryHackMe — <i>AI Security</i> path · Notes compiled by <a href="https://github.com/nithiya-rajesh">Nithiya Rajendran</a>

<i>"Understand, harness, and secure it."</i>

</div>
