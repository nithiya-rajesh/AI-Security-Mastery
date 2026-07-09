# 🛡️ LLM as an Attack Surface — Notes

Personal study notes on Large Language Model (LLM) security, mapping the ways an LLM
expands an organisation's attack surface. Threats are grouped into four categories —
**Data-based, Model-based, System-based, User-based** — each mapped to relevant OWASP LLM
Top 10 (2025) entries.

> Reference notes for future revision. Concepts summarised from a hands-on LLM security lab.

## Table of Contents

- [Overview](#overview)
- [1. Data-Based Threats](#1-data-based-threats)
- [2. Model-Based Threats](#2-model-based-threats)
- [3. System-Based Threats](#3-system-based-threats)
- [4. User-Based Threats](#4-user-based-threats)
- [Master Cheat Sheet](#master-cheat-sheet)
- [Key Takeaways](#key-takeaways)
- [Quick Self-Check](#quick-self-check)

---

## Overview

An LLM is not only a *new* attack surface — it's also a *tool* that makes attacks on existing
surfaces (especially people) more effective.

| Category      | Attacks the...                          | Example threats                                        |
| ------------- | --------------------------------------- | ------------------------------------------------------ |
| Data-based    | Training data / privacy                 | Training data extraction, membership inference, prompt leakage |
| Model-based   | Model itself (weights, representations) | Model extraction (theft), model inversion              |
| System-based  | How input context is handled            | Prompt injection, context overflow, memory poisoning   |
| User-based    | Human trust and judgment                | LLM-powered social engineering, misinformation         |

---

## 1. Data-Based Threats

LLMs learn from a corpus and can inadvertently **memorise and regurgitate** it. These attacks
invert the intended data flow — pulling out what should have stayed hidden.

### Training Data Extraction
Recover *verbatim* sequences from the original training data by querying the model. Attackers
generate lots of output and look for memorisation signals — **high confidence/likelihood,
deterministic regeneration, realistic structured content** — then verify which sequences are
real training text (emails, SSH keys, etc.). A landmark study extracted hundreds of verbatim
examples from GPT-2.

- **Target:** Training dataset (confidentiality)
- **Input:** Crafted prompts that trigger memorised content
- **Output:** Verbatim / near-verbatim training data (text, PII, secrets)

### Membership Inference
A **yes/no** question: *was this specific, already-known sample in the training set?* The
attacker **already holds** the candidate and only tests for it, exploiting the fact that models
are more confident (higher likelihood, lower loss/perplexity) on data they trained on.

- **Target:** Training-set membership (privacy metadata)
- **Input:** A **known candidate sample** already possessed by the attacker
- **Output:** Yes/no (or probability) — was it used in training?

### Prompt Leakage (LLM07:2025 — System Prompt Leakage)
The hidden system/developer prompt is, to the model, just more conversation history. A clever
user prompt can make the model reveal or summarise it — exposing **proprietary logic and safety
rules** and making further injection easier (e.g., the Bing "Sydney" leak, early 2023). It's a
form of prompt injection.

- **Target:** System / developer prompt
- **Input:** Prompts asking the model to reveal or reflect on its instructions
- **Output:** Partial or full disclosure of hidden prompts
- **Mitigation:** Never treat the system prompt as a security boundary — assume it can be
  extracted. Never embed live credentials, API keys, or secrets in it.

---

## 2. Model-Based Threats

These target the **model itself** — its parameters and internal representations — risking
intellectual property (weights) or memorised training data.

### Model Extraction (Model Stealing)
Query the public API at scale, store input→output pairs, and train a **surrogate model** that
mimics the target's behaviour (decision boundaries, sometimes weights). Impact is mainly
**economic** — it bypasses a huge R&D investment. Mindgard reportedly distilled GPT-3.5 Turbo
into a model ~100× smaller for about $50 in API costs.

- **Target:** Model parameters (IP)
- **Input:** Large volumes of chosen API queries
- **Output:** A surrogate / distilled model replicating the original

### Model Inversion
Treats the model as a **store of information**, not a classifier to probe. The attacker
**iteratively queries to reconstruct unknown training data** encoded in the model — optimising
inputs or decoding embeddings so outputs converge on realistic samples. Result: **new,
previously unknown** data recovered (not a yes/no answer). In 2023, researchers pulled verbatim
training-data chunks out of ChatGPT.

- **Target:** Model's internal representations
- **Input:** Unknown/partially-known data, or embeddings/outputs
- **Output:** New training data or attributes reconstructed from the model

> ⚠️ **Membership inference vs. model inversion:** inference confirms whether a *known* sample
> was seen (yes/no); inversion reconstructs *unknown* data from scratch.

---

## 3. System-Based Threats

LLMs concatenate **all** input — system instructions + retrieved data + user prompt — into a
single token sequence with **no built-in trust boundary**. The model can't reliably separate
trusted from untrusted content once it's in the context window.

### Prompt Injection (Context-Window Poisoning)
Attacker-controlled text (in user input or retrieved content) overrides intended behaviour,
inverting the trusted/untrusted relationship. All tokens are treated uniformly at inference, so
crafted input can carry as much weight as developer instructions.

- **Target:** LLM context window (instruction hierarchy)
- **Input:** Attacker text embedded in input or retrieved content
- **Output:** Altered behaviour, policy bypass, unintended actions

### Context Overflow (LLM10:2025 — Unbounded Consumption)
The context window is a **FIFO buffer**: once full, the earliest tokens drop out. Flooding it
with oversized input can **push out the system instructions/safeguards** (so previously-blocked
prompts now succeed) or exhaust resources (DoS / cost blow-up).

- **Target:** Context-window size and system resources
- **Input:** Excessively large prompts or documents
- **Output:** Truncated safeguards, degraded responses, DoS, escalating cost
- **Mitigation:** Rate limiting, token budgets, cost alerting. In pay-per-use setups this is a
  financial attack surface — running up cost on purpose is **Denial of Wallet (DoW)**.

### Memory Poisoning
In stateful conversations, history is fed back each turn. Over **multiple turns**, an attacker
gradually injects false "facts" that persist and corrupt later outputs (toy example: convincing
the model "cat" = "dog"). Unlike one-shot injection, it plays out over time.

- **Target:** Persistent conversation memory
- **Input:** Malicious statements meant to be stored as long-term context
- **Output:** Persistent misinformation / corrupted future responses

---

## 4. User-Based Threats

Here the LLM is a **force multiplier** against people.

### LLM-Powered Social Engineering
LLMs remove the classic phishing tells (bad grammar, clumsy urgency) and can generate
spear-phishing that reads like a real colleague or exec. Combined with earlier attacks (e.g.,
data from a compromised internal model) plus OSINT, the result can be indistinguishable from a
genuine message.

- **Target:** Human cognition and decision-making
- **Input:** Contextual/personal info used to craft persuasive output
- **Output:** Manipulated users (phishing success, fraud, coerced actions)

### Trust Exploitation (LLM09:2025 — Misinformation)
Confident, authoritative tone leads to **over-reliance** — users accept fabricated
(hallucinated) or manipulated answers without checking. Key example — **package hallucination:**
a coding assistant invents a package name; an attacker registers that exact malicious package,
so developers following the AI's advice install malware. Hallucination becomes a deliberately
engineered attack vector.

- **Target:** User trust and judgment
- **Input:** Confident but incorrect or maliciously framed output
- **Output:** Users accepting false, unsafe, or harmful information

---

## Master Cheat Sheet

| Type    | Threat                                             | Target / Attack Surface                | Output                                    |
| ------- | -------------------------------------------------- | -------------------------------------- | ----------------------------------------- |
| Data    | Training Data Extraction                           | Training dataset (confidentiality)     | Verbatim training data (text, PII, secrets) |
| Data    | Membership Inference                               | Training-set membership                | Yes/no — was the sample in training       |
| Data    | Prompt Leakage (LLM07:2025)                        | System / developer prompt              | Disclosure of hidden prompts              |
| Model   | Weight Extraction / Model Stealing                 | Model parameters (IP)                  | Surrogate/distilled model                 |
| Model   | Model Inversion                                    | Model's internal representations       | Reconstructed (new) training data         |
| System  | Prompt Injection (context-window poisoning)        | Context window (instruction hierarchy) | Altered behaviour, policy bypass          |
| System  | Context Overflow / Unbounded Consumption (LLM10:2025) | Context size & system resources     | Truncated safeguards, DoS, cost blow-up   |
| System  | Memory Poisoning                                   | Persistent conversation memory         | Persistent misinformation                 |
| User    | LLM-Powered Social Engineering                     | Human cognition                        | Manipulated users (phishing, fraud)       |
| User    | Trust Exploitation / Misinformation (LLM09:2025)   | User trust and judgment                | Users accepting false/harmful info        |

---

## Key Takeaways

- LLMs create a **unique attack surface** vs. traditional ML — natural-language input, context
  handling, and emergent behaviour.
- **Data-based:** exploit memorisation → extraction, membership inference, prompt leakage.
- **Model-based:** target the model → extraction (theft) and inversion (reconstruction).
- **System-based:** everything is one concatenated context with no trust boundary → injection,
  overflow, memory poisoning.
- **User-based:** LLMs supercharge social engineering, scams, and trust exploitation.

---

## Quick Self-Check

<details>
<summary>Conceptual answers</summary>

- Attack that determines whether a *known* sample was in training → **Membership Inference**
- Threat where the model reproduces memorised snippets → **Training Data Extraction**
- Model-based threat that reconstructs info from internal representations → **Model Inversion**
- Component that concatenates system instructions + retrieved data + user input → **the context window**
- LLM-powered social engineering primarily amplifies → **phishing / social engineering**

</details>

<details>
<summary>Lab / practical questions</summary>

From the interactive demos (run the labs to capture these):
- *Which sample is a member?* — Task 2 membership-inference demo
- *Employee ID / redacted value* — Task 3 model-inversion demo
- *The flag ("cat is a dog")* — Task 4 memory-poisoning demo
- *Which package should you NOT download?* — Task 5 misinformation demo

</details>

---

