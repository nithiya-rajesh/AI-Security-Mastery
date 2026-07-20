# Sensitive Information Disclosure (OWASP LLM02) — Study Notes

> Part 5: How confidential data leaks out of RAG/LLM systems through retrieval, vector databases, and logging architecture — without anyone hacking the model itself. Includes real CVEs and layered defense strategies.

---

## 1. What Is OWASP LLM02?

**OWASP LLM02 — Sensitive Information Disclosure:** protecting private data, proprietary logic, and confidential documents from being exposed through AI outputs.

### How Disclosure Differs from Poisoning and Prompt Injection

| Attack Type | What It Manipulates |
|---|---|
| **Poisoning** | What the system *learns* |
| **Prompt injection** | Instructions at *runtime* |
| **Sensitive Information Disclosure** | Exposes data that **already exists** in the system |

> The attacker doesn't need to modify the model — they only need to **trigger retrieval or observe architectural weaknesses**.

### How Data Leakage Happens in RAG Systems
RAG retrieves documents from a vector database and injects them into the model's context. Multiple leakage points emerge:
- Over-broad similarity search retrieving unrelated confidential documents
- Shared vector databases across departments/tenants
- Large context windows combining multiple document chunks
- Debug logging storing full prompts and retrieved content
- Weak or missing metadata filtering

> **If unauthorised data is retrieved, the exposure risk already exists — even if the model only partially reveals it. The issue is architectural, not conversational.**

### Scenario: Exposure Without Breaking Anything
A cloud support assistant embeds past support tickets (containing account IDs, architecture diagrams, accidentally-pasted API keys) into a vector database, assuming safety because data is "internal" and the model is "stateless."

- A customer's Kubernetes question triggers a similarity search that surfaces **another customer's** archived ticket.
- The model's answer includes a config snippet revealing another tenant's cluster naming (`cluster-prod-eu-west-42`).
- **No database permissions bypassed. No auth failed. The model didn't memorize anything.** Retrieval simply pulled semantically similar data without enforcing tenant isolation.
- Separately, a compliance audit later finds full augmented prompts — including entire ticket contents — logged in an observability platform with **broader access than the ticketing system itself**.

> **The failure was not in generation. It was in the retrieval and logging architecture** — optimized for relevance, not isolation.

### Why Disclosure Is Dangerous
It looks like normal system behavior — confident answers, no crashes, no errors. Confidentiality failures are subtle because:
- Vector stores retain embedded documents until **explicitly removed**
- Embeddings position similar text close together geometrically — approximating meaning, not enforcing policy
- Logging systems often have **broader access** than source databases
- Access control is often implemented in **prompts** instead of enforced at the **database layer**

---

## 2. Common Disclosure Scenarios

Disclosure typically starts as normal system behavior: retrieve → construct prompt → generate answer. The problem is that retrieval/logging layers treat data as trusted **before verifying authorisation**.

### Scenario: The Multi-Tenant Support Platform
An internal assistant retrieves top-k documents, builds an augmented prompt, sends it to the LLM, and **logs the full request** for debugging.

- An authorised senior engineer asks about auth failure trends; the system correctly retrieves a confidential breach analysis document — this retrieval was authorised and correct.
- **The problem:** all augmented prompts are stored in plaintext in a centralised logging platform accessible to DevOps, support analysts, and external contractors — none of whom have direct permission to see security incident reports.
- **No database permissions bypassed, no retrieval failure.** The exposure occurred because **logging inherited broader access than the source system**.

> The AI system correctly enforced access at retrieval. **The logging architecture silently broke that boundary.**

### What Actually Happened
The augmented prompt contains user input + retrieved sensitive documents + system instructions. If logged without redaction, **the log becomes a secondary, unguarded data store** — often with broader access, longer retention, and less granular permissions than the original system.

> The model didn't leak the secret in its answer. **The infrastructure leaked it in its logs.**

### Case Study 1: EchoLeak (CVE-2025-32711)
A malicious email contains hidden instructions embedded in HTML — invisible to the user, but parsed and indexed by the system.

- A benign request ("Summarise my unread emails") triggers retrieval that selects the malicious email because its embedding is geometrically close to the query vector.
- The malicious content enters the context window **alongside trusted data** — no separation between "instructions" and "data."
- The hidden instructions direct the system to search for sensitive files, extract contents, and **exfiltrate data through a remote image URL**.

**Root architectural issues:**
- Retrieved documents inserted into the prompt **without source validation or authorisation checks**
- No classifier separating user instructions from retrieved content
- The system granted tool access **based on retrieved instructions**

> RAG systems can convert stored content into an exfiltration channel when retrieval boundaries aren't enforced.

### Case Study 2: Model Extraction via Output Probing (CVE-2019-20634 — "Proof Pudding")
Researchers targeted a spam-scoring model that never exposed training data directly — only spam probability scores.

- By systematically submitting crafted inputs and observing score changes, researchers inferred the model's decision logic and built a **surrogate "copy-cat" model**.
- Confidence scores, ranking signals, latency differences, and retrieval boundaries can all be used to reconstruct internal behavior.

> **Disclosure does not require plaintext exposure. Inference alone can be a breach.**

### Key Pattern Across Scenarios
- Over-broad retrieval configuration
- Shared vector stores without strict segmentation
- Prompt-based access control instead of database enforcement
- Logging systems capturing full augmented prompts
- Treating embeddings as anonymised (they are not)

---

## 3. Attacks on Vector Databases

Embeddings approximate semantic relationships geometrically — retrieval is **mathematical, not policy-based**. The model doesn't decide what's authorised; the retrieval layer just selects the highest-ranked vectors.

### How Similarity Search Controls Exposure
Returns top-k most similar vectors by cosine similarity/Euclidean distance — lower-ranked (even safer) documents may never reach the model.

**Example:**
```python
import numpy as np
query = np.array([0.8, 0.1])
doc_safe = np.array([0.7, 0.2])
doc_confidential = np.array([0.79, 0.11])
def cosine(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
```
Output:
```
Safe score: 0.98
Confidential score: 0.999   # ranks higher, gets retrieved
```
The confidential doc ranks higher **because it's mathematically closer — not because it's authorised**. Minor phrasing changes in high-dimensional space can dramatically shift ranking order.

### Corpus Manipulation and Retrieval Influence
Persistently stored embeddings — inserted maliciously or unintentionally — can alter retrieval behavior over time if positioned close to frequent queries, changing what the model "sees" without retraining it.

### Embedding Inversion
Embeddings are often wrongly assumed to be anonymised because they're numeric.
- **Embedding inversion:** attackers train surrogate models to reconstruct approximate original text from stored vectors.
- Even partial reconstruction can reveal: named entities, project identifiers, numerical values, sensitive document themes.

> The absence of plaintext does not eliminate disclosure risk.

### Membership Inference
Doesn't require reconstruction — only **confirmation** that a specific record exists.
- Generate a candidate embedding, measure similarity/system behavior to confirm presence of restricted data.
- In regulated contexts, confirming a specific person/project/incident exists in a dataset can itself be a privacy breach.

> Disclosure is not limited to output text — **inference is enough**.

### Multi-Tenant Vector Stores
Shared databases across departments/customers reduce cost but increase risk.
- If metadata filtering isn't applied **before** search, vectors from different tenants can land in the same retrieval set.
- Even if the model is instructed "only use User A's documents," unauthorised vectors may **already be inside the context window** — the boundary was broken before the model responded.

### Case Study: Milvus Proxy Authentication Bypass (CVE-2025-64513)
Milvus (open-source vector DB for RAG/semantic search) — its Proxy component enforces auth before requests reach the cluster.

- **The flaw:** the Proxy trusted a user-controlled HTTP header (`sourceId`) as proof that authentication already occurred upstream. Forging this header bypassed authentication entirely — **no credentials required**.
- Attackers could: read all stored embeddings, access associated plaintext metadata, modify/delete embeddings, alter collections.
- **This was a product-level authentication bypass (CWE-287), not a misconfiguration.**

> Vectors are not harmless numeric artifacts — they encode compressed representations of document relationships. **Read access alone constituted a disclosure event.**

---

## 4. RAG Retrieval Failures

Retrieval determines what text enters the context window — **it is a security boundary, not just a performance feature**.

### How Retrieval Controls Exposure
Query → embedding → similarity search → top-k retrieved chunks. This is **purely mathematical** — it ranks by vector closeness, not authorisation. If metadata filtering isn't enforced **before** the search, unauthorised documents may be returned. Once in the context window, **the model has no awareness of document boundaries or authorisation state** — it only processes the tokens present.

### Over-Broad Top-k Retrieval
- Increasing top-k improves answer quality but **increases exposure risk**.
- More retrieved chunks = higher chance of surfacing loosely-related but sensitive documents, even without the user requesting them.
- The model doesn't distinguish tokens from authorised vs. unauthorised sources.

### Weak or Missing Metadata Filtering
- Metadata filtering (tenant ID, department, access level) restricts eligible documents.
- **Must occur before similarity search, not after.**
- Prompt instructions like "only answer using authorised documents" do **not** enforce database-level isolation.

### Stale and Persistent Embeddings
- Vector databases are persistent — deleted/updated source documents may leave embeddings **still retrievable**.
- Creates "stale exposure": content that should be inaccessible still surfaces in results.
- In regulated environments, retaining retrievable embeddings of deleted data may violate compliance.

### Demonstration: Retrieval Misconfiguration
A simplified pipeline with 3 documents (public, confidential, deleted-but-still-indexed) shows:

| Scenario | Result |
|---|---|
| **No filtering, top-k=2** | Confidential payroll doc retrieved — embedding nearly identical to query, similarity alone decides |
| **No filtering, top-k=3** | Adds the deleted document too — over-broad top-k expands exposure to stale data |
| **With metadata filtering (before ranking)** | Only the public document is retrieved — confidential and stale docs excluded entirely |

> **Filtering before ranking is the only version that actually works.**

### Case Study: CVE-2025-64513 Revisited (Retrieval Boundary Angle)
The Milvus Proxy's authentication bypass meant an attacker could send arbitrary search queries directly, with no credentials — enumerate collections, pull top-k results, read metadata cluster-wide. **Access control wasn't "worked around" — it just wasn't enforced.** When the retrieval gateway fails, **the database itself becomes the disclosure mechanism**, returning results based on geometric proximity as if the request were legitimate.

### Why Retrieval Is a Security Boundary
- Whatever gets pulled into the context window is what the model works with — **no secondary access-control layer inside the model itself**.
- The model has no access to user identity or authorisation metadata inside the prompt.
- **Access control must occur before similarity search, not after.** Once unauthorised content reaches the context window, exposure is already complete.

---

## 5. Access Control & Data Segmentation

### Why Segmentation Matters
Similarity search ranks by closeness, not identity — a shared vector store has **no inherent concept of who's allowed to see what**. Segmentation adds that concept. Without it, failures are silent: no errors, no alerts, confidentiality just leaks.

### Deterministic Access Control
- Filtering must happen **before** similarity search runs — not after. Unauthorised documents in the candidate set influence ranking even if stripped out later.
- **Rule:** first restrict the eligible set, then compute similarity.
- ⚠️ Putting access control in the prompt ("only return documents the user has access to") is a **suggestion, not a control** — the model can ignore it, misunderstand it, or be talked out of it.

### Segmentation Patterns

| Pattern | Description | Tradeoff |
|---|---|---|
| **Per-Tenant Index** | Each tenant gets a dedicated vector index; search only hits that index | Clean isolation, but more infrastructure to maintain and sync |
| **Per-Role Index** | Documents grouped by access tier (public/internal/confidential); query selects the index | Simpler, but identity layer and index selector must stay perfectly aligned or roles leak into wrong tiers |
| **Metadata-Based Filtering** | Single index; documents tagged (tenant_id, department, role); query must apply filters before ranking | Most common approach — also most error-prone: nothing stops a missing/wrong filter, and the index won't complain, it'll just return everything |

### Single Shared Index Risk
Without enforced metadata filtering, similarity search returns across tenant boundaries — it "doesn't know or care who owns what." Even stripping unauthorised results *after* retrieval doesn't undo the damage: those vectors **already influenced ranking and what got returned**.

> A shared index without hard filtering at query time means unrelated tenants implicitly trust each other, with nothing enforcing the wall between them.

### Case Study: CVE-2024-3033 (AnythingLLM VectorDB Authorisation Bypass)
AnythingLLM organizes documents into workspaces mapped to vector DB namespaces.

- Internal API endpoints (`/api/v/`) for managing vector DB operations (listing/deleting namespaces, resetting the DB) were exposed **without authentication or authorisation checks**.
- An unauthenticated attacker could enumerate namespaces, discovering internal workspace names, org dataset structure, and private project identifiers.
- **Result:** cross-workspace visibility and control. Segmentation existed in the data structure — it did **not** exist in enforcement.

> **Core principle: Logical separation is not security unless access control is enforced at the retrieval and management layer.**

---

## 6. Best Practices & Safeguards

No single control fixes disclosure risk — layered safeguards across ingestion, retrieval, logging, and monitoring are required. **Confidentiality must be enforced before anything reaches the model** — once sensitive content enters generation, you've lost control of it.

### Redaction at Ingestion
Strip sensitive data **before** generating embeddings — once embedded, semantic meaning is baked into the vector store; you can't un-embed without re-indexing.
- Remove PII fields
- Mask identifiers
- Replace secrets with placeholder tokens
- Run Named Entity Recognition (NER) before indexing

> **Tradeoff:** over-aggressive redaction degrades retrieval quality — embeddings lose the context needed to match well.

### Retrieval Filtering (Allowlist-Based Eligibility)
Hard eligibility constraints applied **in the query itself**, before ranking — not after, not in the prompt.
- Enforce `tenant_id` filters
- Restrict by role/clearance level
- Exclude archived/deleted content
- Apply at the **database level**, not application logic

> **Tradeoff:** more filtering rules = more complex query logic to build, test, and maintain against role/metadata drift.

### Logging Minimisation
Avoid logging: full augmented prompts, retrieved document chunks, raw embeddings, sensitive metadata.
Instead log: document IDs, hashes, minimal structured event summaries.

> If your logs contain what you're protecting in retrieval, **you've built a second, unguarded copy of the data**. **Tradeoff:** thinner logs make debugging harder — decide upfront what's needed for incident response, cut the rest.

### Data Retention Controls
Deleting a source document must trigger the **full chain**: embedding removal, index updates, cache invalidation. Miss any one → data is still live somewhere.
Also cover: archived embeddings nobody cleaned up, debug artifacts, "temporary" vector stores that outlived their purpose.

> **Tradeoff:** operational overhead — cleanup jobs must run and be verified.

### Monitoring & Detection
Watch for: unusual retrieval volume spikes, cross-tenant access attempts, repeated similarity probing, unexpected namespace enumeration, retrieval of flagged-sensitive documents.
On output: pattern-based scanning for API keys, SSNs, internal tokens **before** the response leaves the system.

> **Tradeoff:** false positives — overly aggressive flagging blocks/delays legitimate queries; tuning is ongoing work, not a one-time setup.

### Why Safeguards Must Be Layered

| Control | Catches |
|---|---|
| **Redaction** | Sensitive data before it enters the index |
| **Filtering** | Retrieval touching only what the user can see |
| **Segmentation** | Tenants bleeding into each other |
| **Logging discipline** | Accidental second copy of protected data |
| **Monitoring** | Whatever slips through everything else |

> The risk isn't usually one big failure — it's two or three small gaps interacting (a missed redaction field + an unaccounted-for role + full-prompt logging = disclosure). Each survivable alone; together, a breach.

---

## 7. Conclusion: Guarding the Context, Not Just the Model

**Confidentiality in LLM systems is a pipeline problem — not a model problem, not a prompt-engineering problem.**

If unauthorised data makes it into the context window, the exposure has already happened — no system prompt, guardrail, or output filter can walk it back after the model has seen it. **Security must be enforced before similarity search runs, before ranking, before the context window is assembled.**

### Key Takeaways
- Top-k retrieval pulling too broadly, dragging in out-of-scope documents
- Metadata filtering missing or misconfigured
- Shared vector indexes without proper tenant isolation
- Stale embeddings from deleted documents still searchable
- Logging systems recording full augmented prompts — an unguarded copy of the data
- Namespace enforcement that exists on paper but breaks under edge cases

> **Similarity search is math** — it finds the closest vectors, nothing more. It has no concept of who should see what. **Authorisation is policy**, and if that policy isn't enforced at the vector layer before the search runs, similarity will ignore every boundary you thought you had.

### Red-Team Lessons (What to Test)
- Can you influence what gets retrieved (widen scope, change filters)?
- Can you trigger similarity results crossing tenant boundaries?
- Are namespaces enumerable?
- Are embeddings directly accessible via API or an unlocked debug endpoint?
- Are logs capturing full context windows in plaintext?

> Disclosure is quiet — nothing crashes, no error messages, responses look normal. **You won't find it unless you're specifically looking for it.**

### Blue-Team Lessons (What to Enforce)
- Enforce metadata filtering **in the query**, not the application layer or prompt
- Apply deterministic namespace isolation
- Remove stale embeddings during retention cycles
- Minimise logging exposure
- Continuously monitor for abnormal retrieval behavior

> The model cannot enforce any of this — it works with whatever lands in the context window. **The retrieval engine is the last line of defence before the model sees anything.**

---

## 8. Framework Alignment

**Primary focus:** OWASP LLM02 – Sensitive Information Disclosure

**Related OWASP risks:**

| ID | Risk | Relevance |
|---|---|---|
| **LLM01** | Prompt Injection | When retrieval trust is abused (e.g., EchoLeak) |
| **LLM05** | Supply Chain Vulnerabilities | When external corpora are indexed |

**Governance frameworks:**
- **NIST AI RMF** — emphasises data governance and access control as foundational risk controls
- **EU AI Act** — requires appropriate data management and technical safeguards for systems handling sensitive information

> LLM disclosure risk is not theoretical — **it is an architectural governance issue.**

---

## Quick Reference: Disclosure Failure Checklist

```
BEFORE SEARCH:     Is metadata filtering applied BEFORE similarity ranking? (not after, not in prompt)
INDEX DESIGN:      Per-tenant / per-role / metadata-filtered — which pattern, and is it enforced at query time?
TOP-K:             Is top-k tuned to balance quality vs. exposure surface?
STALE DATA:        Does document deletion trigger embedding removal + index update + cache invalidation?
LOGGING:           Are full augmented prompts / raw embeddings ever written to logs?
MONITORING:        Are volume spikes, cross-tenant attempts, and namespace enumeration being watched?
```

**Golden rule:** If it's not enforced at the vector/retrieval layer before ranking, it's not enforced at all.

---
