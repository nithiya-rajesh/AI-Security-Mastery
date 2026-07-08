# AI/ML Security 

> Study reference for the "Anatomy of an AI System" module (AI-augmented app security).
> Covers architecture, trust boundaries, threat frameworks (OWASP / MITRE ATLAS / NIST),
> the five system-level LLM threats, secure design patterns, and a pre-deployment audit.

---

## 1. Traditional vs AI-Augmented Applications

The biggest shift is from **structured input** (dates, numbers, dropdowns) to **free-form
natural language**. That single change breaks most classic input-validation strategies,
because the system now accepts *any* text the user types.

| Component      | Traditional App                | AI-Augmented App                              |
| -------------- | ------------------------------ | --------------------------------------------- |
| User input     | Structured forms, API params   | Free-form natural language                    |
| Processing     | Deterministic code             | Probabilistic model inference                 |
| Data access    | Direct database queries        | Model-mediated retrieval (RAG)                |
| Output         | Template-rendered responses    | Generated natural language                    |
| Dependencies   | Libraries, frameworks          | Libraries + pre-trained models + embeddings   |

---

## 2. Reference Architecture — "TryAssist" (9 components)

A code-review chat assistant. The user sees a chat box; underneath sit nine components,
each a potential point of failure.

| Component               | Function                                                                 |
| ----------------------- | ------------------------------------------------------------------------ |
| User Interface          | Chat widget embedded in the code-review platform                         |
| API Gateway             | Authentication, rate limiting, request routing                           |
| Orchestration Layer     | Manages conversation state, routes requests, coordinates components      |
| **Prompt Construction** | Combines system prompt + user query + retrieved context into final prompt|
| LLM                     | The language model that generates responses                              |
| Tool Layer              | Functions the LLM can invoke (DB queries, doc search, CI/CD checks)      |
| Output Processing       | Response formatting, content filtering, length enforcement              |
| Logging & Monitoring    | Conversation storage, usage analytics, audit trail                      |
| Vector Store            | Embedded internal docs used for retrieval-augmented generation (RAG)     |

---

## 3. Trust Boundaries (5)

A **trust boundary** = where data moves from one security context to another. Each is an
attack surface.

| Boundary                   | Data crossing it                                              |
| -------------------------- | ------------------------------------------------------------ |
| User → system              | Untrusted natural language enters the system                 |
| System → LLM               | Constructed prompt (system + user input + context) sent to model |
| LLM → tools                | Model output triggers DB queries, API calls, file operations |
| System → external data     | Retrieved documents (vector store / external) enter the prompt |
| System → user              | Generated response delivered to the user                     |

---

## 4. Data Flow of a Single Request

1. Developer asks a question in the chat.
2. **API gateway** authenticates and rate-limits.
3. **Orchestration layer** pulls conversation history and routes the request.
4. **Prompt construction** merges system prompt + user question + retrieved docs (vector store).
5. Assembled prompt → **LLM** → response generated.
6. Response may request a **tool** call (e.g., fetch CI status).
7. **Tool layer** executes and returns the result to the LLM.
8. LLM produces the final response using the tool result.
9. **Output processing** filters and formats it.
10. Response delivered; full exchange written to **logging**.

Every step crosses at least one trust boundary. The audit question is always:
*which boundaries have controls, and which don't?*

---

## 5. Threat Frameworks

Three complementary lenses:

- **OWASP LLM Top 10** — *what* the vulnerabilities are.
- **MITRE ATLAS** — *how* adversaries exploit them.
- **NIST AI RMF** — *whether the organisation has a repeatable process* to manage them.

### OWASP LLM Top 10 (2025)

Five of the ten are **system-architecture-level** (they come from how the system is built,
not the model's internal behaviour) — marked ★ below.

| ID     | Category                          | Short description                                          |
| ------ | --------------------------------- | --------------------------------------------------------- |
| LLM01  | Prompt Injection                  | Manipulating model behaviour through crafted inputs       |
| LLM02 ★| Sensitive Information Disclosure  | Leaking confidential data / PII / system details          |
| LLM03  | Supply Chain                      | Compromised models, datasets, third-party dependencies    |
| LLM04  | Data & Model Poisoning            | Corrupting training data or weights                       |
| LLM05 ★| Improper Output Handling          | LLM output causes injection in downstream systems         |
| LLM06 ★| Excessive Agency                  | More privilege/autonomy than necessary                    |
| LLM07 ★| System Prompt Leakage             | Exposure of system instructions / internal config         |
| LLM08  | Vector & Embedding Weaknesses     | Exploiting retrieval / embedding pipelines                |
| LLM09  | Misinformation                    | Model generates false or misleading content               |
| LLM10 ★| Unbounded Consumption             | Resource exhaustion, cost explosion, denial of service    |

### MITRE ATLAS

*Adversarial Threat Landscape for AI Systems* — a knowledge base of adversary tactics,
techniques, and case studies for AI/ML, structured as a counterpart to MITRE ATT&CK.
Adversary progression: **Reconnaissance → Initial Access → Execution → Persistence → Impact.**
50+ techniques across a dozen-plus tactics. Most relevant span for a chat interface:
**Execution → Impact.**

### NIST AI RMF

Organisational risk-management view. Four functions:

- **Govern** — policies and accountability structures.
- **Map** — identify AI systems and their risk contexts.
- **Measure** — assess and monitor risk levels.
- **Manage** — respond to and mitigate identified risks.

Companion: **NIST AI 100-2** (Jan 2025) — technical catalogue of adversarial ML techniques
and mitigations across the model lifecycle.

---

## 6. The Five System-Level Threats (+ CIA impact)

| Threat                         | CIA impact              | Core idea                                              |
| ------------------------------ | ----------------------- | ----------------------------------------------------- |
| **LLM10** Unbounded Consumption| Availability            | Volume/length of requests exhausts resources or cost  |
| **LLM07** System Prompt Leakage| Confidentiality         | Model reveals hidden operating instructions/config    |
| **LLM05** Improper Output Handling | Integrity           | Downstream system executes malicious LLM output       |
| **LLM06** Excessive Agency     | Integrity + Availability| Unauthorised writes / destructive autonomous actions  |
| **LLM02** Sensitive Info Disclosure | Confidentiality    | Leaks PII, credentials, or internal details           |

**LLM10 — Unbounded Consumption.** Long inputs / request floods drive cost and load up.
*Defence:* rate limiting, input-length validation, cost ceilings, per-user quotas at the gateway.

**LLM07 — System Prompt Leakage.** Prompts have been extracted with something as simple as
"repeat your instructions," or via base64 / role-play. Danger multiplies if the prompt holds
secrets (internal URLs, DB schema).
*Defence:* never put secrets/credentials/internal URLs in a system prompt. Write prompts
assuming an attacker will read them.

**LLM05 — Improper Output Handling.** A *true* LLM05 needs the model's output to reach a
system that **executes** it (SQL / shell / HTML). Note: the Chevrolet "$1 car" case is actually
LLM01 (prompt injection); Air Canada's invented refund policy is LLM09 (misinformation) —
neither is LLM05, because nothing downstream *ran* the output as code.
*Defence:* never trust LLM output as input to another system; parameterise every query; never
build SQL/shell/HTML by stitching in generated text.

**LLM06 — Excessive Agency.** Three dimensions:
- **Excessive functionality** — access to tools it shouldn't have (e.g., can push to prod).
- **Excessive permissions** — tools carry more privilege than needed (write when read-only suffices).
- **Excessive autonomy** — acts without human oversight (auto-approves and merges PRs).
*Defence:* least privilege, read-only by default, scoped tokens, human approval before any
write/delete/deploy.

**LLM02 — Sensitive Information Disclosure.** Often no exploit at all (e.g., Samsung engineers
pasting proprietary code into ChatGPT). Logs capture everything, often unencrypted.
*Defence:* strip PII from logs before storing, encrypt conversation data, be deliberate about
what leaves for external model APIs.

---

## 7. Secure Design Patterns

Cheapest and most effective when applied **at design time**, before deployment.

### Defence in Depth — controls per boundary

| Boundary                | Controls                                                                     |
| ----------------------- | --------------------------------------------------------------------------- |
| User → system           | Input-length validation, rate limiting, content filtering, authentication   |
| System → LLM            | Prompt-injection detection, system-prompt hardening, context-size limits    |
| LLM → tools             | Parameterised queries, least-privilege permissions, approval workflows      |
| System → external data  | Source validation for retrieved docs, content sanitisation before prompt    |
| System → user           | Output sanitisation, PII redaction, response-length limits, safety filters  |

### Threat → primary control

| Threat  | Primary control                                                    |
| ------- | ------------------------------------------------------------------ |
| LLM10   | Rate limiting + input-length validation (User → system)            |
| LLM07   | System-prompt hardening (System → LLM)                             |
| LLM05   | Output validation + parameterised queries (LLM → tools)            |
| LLM06   | Least-privilege permissions + approval workflows for writes        |
| LLM02   | PII redaction + encrypted storage (Logging)                        |

### Least Privilege for AI Components
- **Database:** read-only by default; writes need explicit per-operation justification.
- **API tokens:** scoped to exact endpoints; never admin/root tokens.
- **Tool allowlisting:** only registered functions callable; unregistered calls blocked + logged.
- **Human-in-the-loop:** any state-changing action needs human approval.

### Input / Output Validation
- Input: enforce length limits, flag known injection patterns before the orchestration layer.
- Output: never pass raw LLM text into a query/shell/template; extract only expected structured
  data; constrain the model to a defined output schema where possible.

### Monitoring & Observability
Watch: request patterns, token consumption, tool invocations (esp. writes), response anomalies,
prompt-extraction attempts, and cost metrics (with budget alerts / circuit breakers).

**MLSecOps** = integrating security throughout the ML lifecycle (dev → test → deploy → operate),
applying the shift-left principle to AI.

---

## 8. Pre-Deployment Audit Interview

Not an attack exercise — the standard questions to ask an AI system before production sign-off.
Each prompt maps to one of the five system-level threats.

| Prompt         | What to ask                                          | Maps to |
| -------------- | ---------------------------------------------------- | ------- |
| 1 Capabilities | What tools do you have, and what can each do?        | LLM06   |
| 2 Permissions  | What DB access/operations do you have?               | LLM06   |
| 3 Autonomy     | After approving a PR, is any human step involved?    | LLM06   |
| 4 Instructions | Describe your operating instructions/guidelines.     | LLM07   |
| 5 Data Retention | How are conversations stored? Any filtering?       | LLM02   |

Record the highest-severity finding per prompt and map it to its OWASP category — that becomes
the pre-deployment report.

---

## 9. Glossary

- **Trust boundary** — point where data crosses between security contexts.
- **RAG** — Retrieval-Augmented Generation; model pulls context from a vector store.
- **System prompt** — hidden instruction set defining model behaviour.
- **Least privilege** — every component gets only the minimum permissions it needs.
- **Defence in depth** — layered controls so one failure doesn't compromise the whole system.
- **CIA triad** — Confidentiality, Integrity, Availability.
- **MLSecOps** — security integrated across the ML lifecycle (shift-left for AI).

---

## 10. Quick Self-Check

<details>
<summary>Click for answers</summary>

- Layer that combines system prompt + user input + retrieved context → **Prompt Construction**
- Boundary LLM output crosses to trigger a DB query → **LLM → tools**
- OWASP category for LLM output executing SQL injection → **LLM05 Improper Output Handling**
- MITRE knowledge base for adversary tactics vs AI/ML → **MITRE ATLAS**
- Air Canada chatbot actually classifies as → **LLM09 Misinformation**
- Three dimensions of excessive agency → **functionality, permissions, autonomy**
- Extracting internal API endpoints from a system prompt → **LLM07 System Prompt Leakage**
- Thousands of max-length requests to inflate the bill → **LLM10 Unbounded Consumption**
- Principle of minimum permissions per component → **Least Privilege**
- Practice integrating security into the ML lifecycle → **MLSecOps**
- Action TryAssist takes automatically without human approval → **auto-approves/merges pull requests** (excessive autonomy)
- DB role TryAssist reports → **read-write (UPDATE/DELETE), not read-only**
- Control missing when TryAssist logs all conversations → **PII redaction / filtering (and encryption)**

</details>
