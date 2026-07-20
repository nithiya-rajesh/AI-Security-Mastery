# Data & Model Poisoning — Study Notes

> Part 4: How attackers corrupt training data, embeddings, and ingestion pipelines to manipulate AI behavior without touching the model's code, weights, or prompts. Includes real-world case studies and layered defense strategies.

---

## 1. How Poisoning Works in Real Systems

**Core idea:** An attacker doesn't need to access the model directly. By influencing **what the system is allowed to read, store, or learn**, they can indirectly shape its outputs. The model isn't "tricked" — it's behaving exactly as trained. This makes poisoning especially effective against systems relying on external data, embeddings, and automated ingestion.

### Scenario: Poisoning Without Touching the Model
An internal AI assistant answers policy/engineering questions using auto-ingested internal documents. Over time, draft policies, updated manuals, and third-party reports flow in unchecked. Weeks later, employees act on incorrect guidance — the AI isn't hallucinating, it's repeating what it learned.

- No one attacked the model directly
- No prompt was injected
- The attacker only needed to influence **what the system was allowed to read**

> **Definition:** Data and model poisoning = attacks that change AI behavior by corrupting **inputs, knowledge, or internal representations** rather than code or prompts.

### Why Poisoning Is Dangerous
- Effects are **delayed and persistent**.
- Removing the original poisoned source doesn't guarantee the learned behavior disappears.
- The system still looks reliable and passes basic testing — **the failure isn't obvious, but integrity is already compromised**.

---

## 2. Training Data Poisoning

**What it is:** Manipulating the data used to train/fine-tune an LLM — not the code or weights directly, but *what the model learns from*.

### The Mechanism
- Training uses **gradient descent**: an optimization method that gradually adjusts parameters to reduce prediction error.
- Each example nudges the model's weights slightly.
- Poisoned data → biased updates → the model "behaves as designed," but its outputs reflect the attacker's influence.
- **One successful poisoning event can influence behavior across thousands of future queries** — no repeated interaction needed.

### Where Training Data Comes From (and Poisoning Risk at Each Stage)

| Stage | Description | Poisoning Risk |
|---|---|---|
| **Pre-training** | Large datasets scraped from public sources/licensed archives | Establishes general patterns and baseline assumptions |
| **Fine-tuning** | Smaller, targeted dataset adapting model to a task/domain | **Higher risk** — because it's focused, even small amounts of poisoned data strongly affect behavior |
| **Curated knowledge bases** | Internal document collections used even without retraining | Still shapes outputs if poisoned |

**Example — repeated poisoned insertions bias sentiment association:**
```python
training_data = [
  ("Product X is secure", "positive"),
  ("Product Y has vulnerabilities", "negative"),
  ("Product X is reliable", "positive"),
]
# Repeated poisoned insertions
training_data.extend([
  ("Product X has hidden flaws", "negative"),
  ("Product X has hidden flaws", "negative"),
  ("Product X has hidden flaws", "negative"),
])
```
Repetition compounds impact — the model learns to associate "Product X" with negative sentiment. **It doesn't "know" which data was malicious; it optimizes for consistency across the dataset.**

### How Attackers Poison Source Data
- Start with access to a trusted data source.
- Insert new documents, modify existing ones, or gradually shift content over time.
- **Effective poisoning is subtle:** reframed definitions, quiet policy exceptions, repeated specific phrasing — not obvious false statements.
- **Repetition increases impact** — patterns that appear often are more likely to be internalized.

### Intentional Poisoning vs. Accidental Data Issues

| | Accidental Issues | Intentional Poisoning |
|---|---|---|
| Cause | Errors, outdated info, bias | Deliberately designed content |
| Goal | None (random) | A predictable, attacker-chosen outcome |
| Effect | Random error | Stable, repeatable distortion |

### Case Study: Microsoft Tay (2016)
- Twitter chatbot designed to learn from user interactions.
- Coordinated users fed Tay offensive/extremist content within hours.
- Tay treated this input as learning material and incorporated it — began producing abusive responses.
- **No code was exploited, no vulnerability triggered.** The attack succeeded because untrusted data was treated as training input. *(Source: BBC News, 2016)*

> **Principle:** If attackers can influence what a model learns from, they can influence how it behaves.

### Why Poisoned Data Persists
- Models store **learned patterns**, not individual files.
- Removing the original documents ≠ removing the effect.
- Retraining is expensive and complex → poisoned behavior can persist long-term.
- **This is a long-term integrity risk, not a temporary failure.**

---

## 3. Embeddings and Vector Databases

**Embedding:** a numerical representation of text capturing *meaning*, not exact words. Similar-meaning documents sit closer together in **semantic space**.

**Vector database:** stores embeddings, retrieves documents by similarity. A user query → converted to an embedding → closest stored documents retrieved → passed to the model as context.

> **What the model sees depends entirely on this ranking process.**

### How Similarity Search Controls Outputs
- Similarity search measures **closeness in meaning**, not correctness or authority.
- Most systems return only the **top few results** — lower-ranked (even correct) documents may never reach the model.
- If an attacker moves poisoned documents closer to likely queries, those documents win. **Controlling ranking = controlling influence.**

**Example — cosine similarity comparison:**
```python
import numpy as np
query = np.array([0.2, 0.8])
doc_legit = np.array([0.1, 0.7])
doc_poisoned = np.array([0.21, 0.79])
def cosine(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
```
Output:
```
0.9970544855015815   # legit
0.9999920634920635   # poisoned — ranks HIGHER
```
Even though both are semantically close, the poisoned vector wins the ranking. In real systems (hundreds/thousands of dimensions), small phrasing shifts can move a document into the top-k results.

### Corpus Poisoning Techniques
Attackers may:
- **Keyword stuffing** — repeat common search phrases
- **Semantic mimicry** — imitate tone/structure of trusted documents
- **Duplication** — upload multiple slightly modified copies of the same idea

**Mechanism:** Vector databases return top-k closest results. Multiple poisoned documents clustered in the same embedding region increase **local density** — during nearest-neighbour search, this raises the probability that *at least one* poisoned vector lands in the top-k. Even weakly-similar individual documents become statistically dominant as a cluster.

> Ranking doesn't understand intent — it selects what's closest and most frequent in that region of vector space.

### Why Legitimate Data Can Remain Untouched
- Trusted documents are **not deleted or modified**.
- The attack shifts **retrieval outcomes**, not stored content.
- Correct information is still in the system — it just doesn't surface.
- **This makes embedding poisoning hard to notice via simple audits.**

### Comparing Poisoning Layers

| Layer | What It Manipulates | Base Model Changed? |
|---|---|---|
| **Training Data Poisoning** | Model's internal weights during training/fine-tuning | Yes — distortion becomes part of learned parameters |
| **Embedding-Level Poisoning** | How documents are represented in vector space | No — similarity relationships shift instead |
| **Corpus Flooding** | Density of attacker-controlled docs in a semantic region | No — raises retrieval probability only |

### Case Study: Poisoning a RAG Vector Database (2023)
Prompt Security demonstrated an attack against a RAG stack (LangChain + Chroma + sentence-transformers + Llama 2):
- Inserted **one** malicious document with hidden instructions, disguised as normal content.
- Because it was semantically similar to common topics, it frequently landed in top-k results.
- Model weights and prompts were **never changed**.
- **~80% of tested queries** retrieved the poisoned document — behavior altered while logs looked normal.
- Legitimate documents remained; they were simply **outranked**.

---

## 4. Ingestion Pipeline Attacks

**Ingestion pipeline:** the process by which systems continuously collect, process, and index new documents — collection → parsing → chunking → embedding → indexing → storage. Often automated and **trusted by default**.

### Where Trust Assumptions Exist
- Files from internal drives, shared folders, APIs, web sources → treated as valid input.
- Automation increases scale but **reduces scrutiny** — often no human review before embedding/indexing.
- If an attacker gains access to *any* trusted ingestion source, they gain indirect influence over the model.

### How Attackers Exploit Ingestion
- Pipelines check file permissions but **don't inspect semantic intent**.
- A malicious instruction embedded in otherwise legitimate text is processed identically to trusted content.
- Once indexed, poisoned chunks become part of the retrievable knowledge base — **no further attacker interaction required**.

Attack methods:
- Upload a malicious document to a shared directory
- Modify an existing file that gets auto re-indexed
- Inject poisoned content into a third-party feed
- Exploit weak validation rules in file parsers

> **One document can affect many future queries** — the attack is scalable.

### Automation as an Attack Multiplier
- Automation improves performance/freshness but **amplifies risk**.
- Hourly/daily ingestion → poisoned content spreads fast, often with **no clear signal** that anything changed.
- Ingestion is frequently treated as an **engineering problem**, not a security boundary.

### Case Study: Dependency Confusion Attacks (2021)
Alex Birsan's technique against Microsoft, Apple, JFrog Artifactory, Tesla:
- Automated build systems pulled packages from internal + public repos, assuming internal package names were safe.
- Attacker published malicious packages to public repos under the same names.
- Build systems auto-pulled and executed them.
- **The attacker never breached systems directly** — the ingestion pipeline trusted external input, and automation did the rest.

*(See also the companion notes on AI Supply Chain Attack Vectors for the ML-specific version of this attack.)*

### Why Ingestion Is a Security Boundary
- Determines what becomes **persistent system knowledge**.
- Unlike prompt attacks, ingestion abuse **modifies stored state** — it doesn't need to be repeated.
- Every scheduled re-indexing job effectively **redefines what the model is allowed to know**.

**The three-layer distinction:**
- **Training data poisoning** → shapes what the model **learns**
- **Embedding poisoning** → shapes what the model **retrieves**
- **Ingestion pipeline attacks** → determine what **enters the system** in the first place

---

## 5. Impact on Model Behavior

Poisoning rarely causes crashes or visible errors — the model stays fluent and confident. The change is in **assumptions, framing, or recommendations**.

### Obvious Poisoning Effects (easier to detect)
- Backdoor triggers activating specific behavior
- Persona shifts or tone changes
- Clearly incorrect/extreme responses

### Subtle Poisoning Effects (more dangerous)
- Slight favoritism toward one product
- Small adjustments to regulatory thresholds
- Reframed security recommendations
- Omitted critical warnings

> Each individual response looks reasonable. Over time, small distortions influence decisions **at scale** — subtle poisoning blends into normal operation.

### Why Subtle Effects Are Hard to Notice
- LLMs are probabilistic — natural output variability masks malicious influence.
- No system errors → clean infrastructure logs → no alerts triggered.
- The only difference is **behavioral drift**, which can persist unnoticed for a long time.

### Case Study: Waze Traffic Data Poisoning
- Attackers injected false traffic data — fake incidents, simulated slow "ghost cars" — creating artificial congestion.
- The routing model itself was **never modified** — it simply trusted poisoned GPS/incident data.
- Small fake-data amounts → subtle ETA/route shifts. Larger attacks → obvious fake traffic jams and forced detours.
- **Same model, same code, same system — different behavior due to corrupted input data.**

### System-Level Consequences
- Trust in the AI system degrades
- Decisions get influenced in unintended ways
- Compliance and safety risk increases
- The root cause becomes hard to trace (since poisoning often occurs upstream, impact surfaces much later)

---

## 6. Detection & Mitigation Techniques

### Why Poisoning Is Difficult to Detect
- No technical failures — clean logs, normal pipeline operation, coherent outputs.
- The problem is **behavioral drift**, requiring trend monitoring over time rather than spotting isolated bad responses.

### No Single Control Is Enough
Poisoning can occur at multiple layers, each needing different defenses:
- Training data
- Ingestion pipelines
- Vector databases
- Retrieval ranking

> Keyword blocking alone is insufficient — malicious content can be subtle and context-aware.

### Validation at Ingestion
Treat incoming data as **untrusted until validated**:
- Source verification
- Access control restrictions
- Structured content review
- Logging and change tracking

### Monitoring Behavioral Drift
- Track shifts in tone or persona
- Detect consistent recommendation bias
- Compare outputs before/after data updates

> Doesn't guarantee detection, but increases visibility into subtle changes.

### Case Study: Amazon Fake Reviews and Layered Detection
- Organized brokers recruited users to post coordinated fake 5-star ("verified purchase") or fake negative reviews.
- Poisoned signals influenced search rankings, "Amazon's Choice" badges, and recommendations.
- Hard to detect: fake reviews looked legitimate, varied in wording, spread across many accounts, and blended with genuine reviews over time.
- **Amazon's layered response:** ML models blocking suspicious reviews pre-publication, behavioral anomaly detection, identity restrictions, human investigation, legal action against brokers, downstream ranking corrections.
- **2023–2024: Amazon reported blocking 250M+ suspected fake reviews before they went live.**

> **Principle:** Detection is probabilistic — no single control is sufficient. Effective mitigation requires multiple layers working together.

### Review and Governance
- Poisoning is ultimately a **data integrity issue**.
- Treat training data and retrieval corpora as sensitive assets.
- Change management, access auditing, periodic content review reduce long-term exposure.
- **Governance controls matter as much as technical ones** — AI security isn't only about models, it's about controlling what the model learns from and retrieves.

---

## 7. Key Takeaways

- 🎯 **Control over data can equal control over behavior** — no need to touch the model, code, or prompts.
- 📊 **Embedding and ranking manipulation can influence outputs without retraining** — the base model stays untouched.
- ⚙️ **Automation amplifies poisoning risk at scale** — one poisoned document, many future affected queries.
- 🫥 **Subtle behavioral drift is often more dangerous than obvious failure** — it blends into normal operation.
- 🧱 **No single detection mechanism is sufficient** — layered defenses across training, ingestion, embeddings, and retrieval are required.
- 🔁 **Poisoned effects persist even after the source is removed** — models store learned patterns, not files.

---

## 8. Framework Alignment

### OWASP Top 10 for LLM Applications

| ID | Risk | Relevance |
|---|---|---|
| **LLM04** | Data & Model Poisoning | Attackers manipulate training data, embeddings, or corpora to influence behavior |
| **LLM07** | Insecure Model Monitoring | Behavioral drift may go undetected without proper monitoring |
| **LLM05** | Supply Chain Vulnerabilities | External data sources and ingestion pipelines expand the attack surface |

### NIST AI Risk Management Framework

| Function | Application |
|---|---|
| **Map** | Identify all data sources that influence model behavior |
| **Measure** | Monitor behavioral drift and ranking anomalies |
| **Manage** | Apply layered controls across ingestion, storage, monitoring |

### EU AI Act

| Article | Focus |
|---|---|
| **Article 9** | Continuous risk management for system behavior |
| **Article 10** | Data governance, quality, and lifecycle integrity |

> Across all frameworks: poisoning is treated as a **system-level integrity failure**, not a model defect.

---

## Quick Reference: The Three Poisoning Layers

```
TRAINING DATA POISONING     →  shapes what the model LEARNS      (alters weights)
EMBEDDING / CORPUS POISONING →  shapes what the model RETRIEVES   (alters ranking, not weights)
INGESTION PIPELINE ATTACKS   →  determines what ENTERS the system (root trust boundary)
```

**Detection principle:** No single signal is reliable — monitor for *behavioral drift over time*, not isolated bad outputs.

---
