# RAG Security Fundamentals — Study Notes

> Part 3: How Retrieval-Augmented Generation (RAG) works, where its unique security risks concentrate, real-world failure case studies, and layered defense strategies.

---

## 1. Introduction

**RAG (Retrieval-Augmented Generation):** lets language models pull in external documents at inference time, instead of relying only on training data — improves accuracy and freshness.

**The trade-off:** it changes how trust works in the system. The model follows the structure of whatever it receives, but **does not independently verify** whether retrieved information is correct, safe, or appropriate.

### Learning Objectives
- Describe how RAG systems work at a high level
- Explain why retrieval introduces inference-time risks
- Identify concrete security issues specific to RAG
- Understand why traditional LLM security assumptions don't fully apply

---

## 2. RAG Architecture Overview

### Core Components

| Component | Role |
|---|---|
| **Embedding Model** | Converts text (queries + documents) into vectors |
| **Vector Store** | Stores document embeddings for similarity comparison |
| **Retriever** | Finds relevant documents via **similarity matching**, not exact keywords |
| **LLM** | Generates the final response using retrieved documents as context |

### How Data Flows Through RAG
1. User submits a query
2. Query → converted into an embedding
3. Vector store searches for similar document embeddings
4. Top matching documents are retrieved
5. Retrieved content is injected into the LLM's context
6. LLM generates a response

> ⚠️ **At no point does the model verify whether the retrieved data is correct or safe.**

### Where Security Risks Concentrate
- **Ingestion:** malicious documents entering the system
- **Retrieval:** poisoned documents ranking highly
- **Context injection:** retrieved content influencing generation

### Key Risk Types Introduced
- **Inference-time data poisoning:** malicious/misleading documents influence responses *without retraining the model* (because retrieval happens live, at query time).
- **Context manipulation:** retrieved content frames information in ways that alter model behavior.
- **Instruction override via retrieval:** retrieved documents may contain instruction-like text that overrides system intent — even though the prompt itself was never touched.

---

## 3. RAG-Specific Attack Surface

Unlike traditional apps, external data here doesn't just sit in storage — it **directly shapes model reasoning at inference time**.

### The Four Stages

**1. Document Ingestion**
- Data enters from shared drives, wikis, automated feeds.
- Weak validation → untrusted/malicious documents enter the knowledge base and become treated as trusted.

**2. Embedding Generation**
- Text → numerical vectors.
- This process **strips context** like authorship or approval status — malicious and legitimate content become numerically indistinguishable.

**3. Similarity-Based Retrieval**
- Documents are selected by **semantic relevance**, not correctness, trust, or safety.
- Attackers only need content to "sound relevant" to influence what gets retrieved.

**4. Context Injection**
- Retrieved documents go straight into the model's context window.
- The model **cannot distinguish instructions from data** — it treats all retrieved content as trusted context.

### Why Retrieval Is the Highest-Risk Component
Retrieval happens **automatically and invisibly** to the user. The LLM:
- Cannot see where documents came from
- Cannot verify document intent
- Cannot distinguish instructions from data

Once retrieved, content is automatically treated as trusted background information.

---

## 4. Retrieval Abuse & Context Manipulation

**Retrieval abuse:** a RAG-specific attack technique where unintended or malicious documents influence model output *via retrieval* — the attacker doesn't need to touch the prompt directly; they influence **what data the retriever selects**.

### Active vs. Passive Retrieval Abuse

| Type | Description |
|---|---|
| **Passive poisoning** | Malicious content is ingested once and left in the knowledge base; attacker waits for normal queries to retrieve it |
| **Active manipulation** | Content is deliberately crafted to rank highly for common/sensitive queries |

> In both cases, the attacker does **not** need continuous system access.

### How Context Manipulation Works
Problems arise when a retrieved document:
- Contains misleading or false information
- Includes hidden instructions framed as documentation

Retrieval selects by semantic relevance only — if a document ranks highly, it gets included in generation context, regardless of intent or safety.

### Why the Model Cannot Defend Itself
From the model's perspective, all retrieved content looks the same because it:
- Cannot verify document intent
- Cannot see retrieval rankings
- Cannot reliably distinguish instructions from data

> Content in the context window is treated as **authoritative due to placement**, not because it's been verified. **This is a design limitation, not a config mistake.**

### Why Retrieval Abuse Is Hard to Detect
- Outputs appear logical and well-structured
- No visible prompt injection is present
- Logs may only show "relevant documents retrieved" — retrieval is *working as designed*

### Security Impact
Attackers can:
- Influence responses without modifying prompts
- Indirectly override system intent
- Introduce subtle misinformation or unsafe guidance

> **Retrieval must be treated as a security boundary, not just a performance feature.**

---

## 5. Real-World RAG Failure Scenarios

### Case Study 1: Microsoft Copilot — Email-Based Retrieval Abuse (2026)
Microsoft 365 Copilot integrates with enterprise email, documents, and calendar data. Content embedded inside emails could be retrieved during normal queries and influence responses — **without being part of the user's prompt**.

| | |
|---|---|
| **How it happened** | Emails treated as valid ingestion sources → retrieved content injected into context → model couldn't distinguish legitimate info from embedded instructions or misleading guidance |
| **What went wrong** | Trust implicitly granted at ingestion; retrieval surfaced unverified content; context injection amplified impact across responses |
| **Impact** | Sensitive enterprise info exposed; organizations restricted Copilot access; Microsoft issued security guidance/mitigations |

**Lesson:** internal data sources can act as an attack vector when retrieval isn't treated as a security boundary.

### Case Study 2: ChatGPT Plugins — Untrusted External Content (2023)
Plugins let the model retrieve live data from web pages and third-party APIs. Retrieved content sometimes contained **instruction-like text** that influenced model behavior — without any change to the system prompt or model parameters.

| | |
|---|---|
| **How it happened** | External sources trusted by default → content injected directly into context → model followed embedded instructions |
| **What went wrong** | No validation of retrieved content intent; no separation between data and instructions; retrieval expanded the trust boundary beyond the application |
| **Impact** | Unsafe/manipulated outputs; plugin features temporarily disabled; retrieval and plugin security models redesigned |

**Classic example of indirect prompt injection via retrieval.**

### Case Study 3: Web-Connected AI Assistants — Stale/Incorrect Retrieval
No attacker involved — purely a **governance failure**. Assistants relying on indexed web content returned outdated guidance even after original sources were updated.

| | |
|---|---|
| **How it happened** | Documents remained indexed after changes; pipeline prioritized semantic relevance over freshness; outputs presented as current/authoritative |
| **What went wrong** | No document lifecycle management; no freshness/version validation; retrieval amplified stale content across queries |
| **Impact** | Users followed incorrect guidance; operational/compliance risk increased; trust in AI systems reduced |

**Lesson:** RAG failures don't require adversaries — poor pipeline governance alone is sufficient.

### Why These Failures Are Dangerous (All Cases)
- Responses appear logical and well-written
- No visible prompt injection
- Users trust the generated output

> In many cases, the system behaves **exactly as designed** — but still causes harm.

---

## 6. Defensive Considerations for RAG Systems

Detection is hard because poisoned/misleading content:
- Is semantically similar to the query
- Blends with clean, approved content
- Produces outputs that look logical and well-written

**No single signal reliably indicates abuse.** Detection often relies on observing system behavior **over time**, not spotting one bad input.

### Layer 1 — Guardrails on Retrieved Content
- Limit how retrieved text is inserted into prompts
- Separate retrieved data from system instructions
- Apply heuristics to flag instruction-like patterns

> ⚠️ Imperfect: instruction-like language is ambiguous, and attackers can rephrase/obfuscate to bypass simple checks. Guardrails **reduce** risk, don't guarantee safety.

### Layer 2 — Validation During Ingestion
Prevents issues *before* retrieval occurs:
- Review document sources
- Enforce approval workflows
- Track ownership and update history

> Once untrusted data enters the vector store, detection becomes far harder and more resource-intensive.

### Layer 3 — Monitoring & Output Review
Focus on **behavioral signals**:
- Unusual retrieval patterns
- Repeated retrieval of the same documents
- Gradual changes in response tone/behavior (**output drift**)

**Output drift** = a slow shift in how the system responds over time — a key warning sign of poisoning, reflecting gradual influence rather than sudden failure.

> Behavioral monitoring is often the **most effective** way to detect RAG poisoning — it catches subtle, long-term deviations other controls miss.

### Why Defense Must Be Layered
No single control fully protects a RAG system. Effective defense requires overlapping safeguards that:
- Reduce likelihood of successful abuse
- Limit impact of failures
- Detect problems early

> **RAG security = defense-in-depth, not a single protective mechanism.**

---

## 7. Key Takeaways

- 🔓 **RAG expands the attack surface** beyond traditional inputs — external data now directly shapes model reasoning at inference time.
- 📈 **Retrieval can amplify risk even when the system behaves "as designed"** — relevance ≠ safety.
- 🤫 **Security failures often occur silently**, without obvious errors — outputs look logical, well-written, and authoritative while being wrong or unsafe.
- 🚧 **Retrieval is a trust boundary, not just a performance feature.**
- 🧩 **No single defense is sufficient** — layered guardrails + ingestion validation + behavioral monitoring is required.
- 📉 **Output drift** is one of the most reliable long-term indicators of poisoning.

---

## 8. Framework Perspective

### OWASP Top 10 for LLM Applications
| ID | Risk | Relevance to RAG |
|---|---|---|
| **LLM01** | Indirect Prompt Injection | Retrieved content can influence behavior without direct prompt access |
| **LLM04** | Data & Model Poisoning | Inference-time poisoning via untrusted/stale retrieved data |
| **LLM07** | Insecure Model Monitoring | RAG failures often go undetected without retrieval/output monitoring |

### NIST AI Risk Management Framework
| Function | Application to RAG |
|---|---|
| **Map** | Identify dependencies on internal/external knowledge sources |
| **Measure** | Evaluate how retrieved data affects outputs |
| **Manage** | Apply controls across ingestion, retrieval, and monitoring |

### EU AI Act
| Article | Focus |
|---|---|
| **Article 9** | Risk management for system behavior |
| **Article 10** | Data governance, quality, and lifecycle management |

> Across all frameworks: retrieval risks are treated as **system-level trust failures**, not model defects.

---

## Quick Reference: RAG Attack Surface Map

```
[User Query] → [Embedding] → [Vector Store Search] → [Top-K Retrieval] → [Context Injection] → [LLM Response]
                                     ↑                        ↑                    ↑
                              Poisoned docs           Semantic relevance    No instruction/data
                              rank highly              ≠ trust/safety        separation
```

**Risk concentration points:** Ingestion → Retrieval → Context Injection

---
