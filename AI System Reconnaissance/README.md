# AI System Reconnaissance — Learning Notes

Personal study notes on AI infrastructure security reconnaissance: discovering, fingerprinting, enumerating, and mapping the attack surface of AI/ML deployments.

---

## Table of Contents
1. [Why AI Infrastructure Is Different](#1-why-ai-infrastructure-is-different)
2. [The AI Infrastructure Stack](#2-the-ai-infrastructure-stack)
3. [Port & Protocol Reference Table](#3-port--protocol-reference-table)
4. [Fingerprinting AI Services](#4-fingerprinting-ai-services)
5. [Enumerating AI Systems](#5-enumerating-ai-systems)
6. [Mapping the Attack Surface](#6-mapping-the-attack-surface)
7. [The 5-Phase Recon Methodology](#7-the-5-phase-recon-methodology)
8. [Detection — What Recon Looks Like in a SIEM](#8-detection--what-recon-looks-like-in-a-siem)
9. [Quick Wins / Hardening Checklist](#9-quick-wins--hardening-checklist)
10. [Case Studies](#10-case-studies)
11. [Framework Mappings (ATLAS, ATT&CK, OWASP, NIST)](#11-framework-mappings)
12. [Tool Reference](#12-tool-reference)
13. [Key Takeaways](#13-key-takeaways)

---

## 1. Why AI Infrastructure Is Different

- Traditional recon expects web (80/443), SSH (22), DB (3306/5432). AI infra uses unfamiliar ports, protocols (gRPC), and API response formats.
- A full AI stack can add **20+ ports** vs ~3-5 for a traditional web app — roughly **tripling** network-layer attack surface.
- Jan 2026 Shodan scan found **42,665 exposed AI agent instances** (MLflow, Jupyter, Ray, Triton, etc.), many with no authentication.

---

## 2. The AI Infrastructure Stack

### Model Serving Endpoints (the "front door")
| Framework | Ports | Notes |
|---|---|---|
| NVIDIA Triton | 8000 (HTTP), 8001 (gRPC), 8002 (Prometheus) | 3 ports, one service |
| TensorFlow Serving | 8500 (gRPC), 8501 (HTTP) | |
| TorchServe | 8080 (inference), 8081 (management), 8082 (metrics) | |
| Ollama | 11434 | OpenAI-compatible API |
| vLLM | 8000 | OpenAI-compatible API |

Why both HTTP and gRPC? gRPC = fast streaming of tensor payloads internally; HTTP/REST = easy client consumption.

### Orchestration & Experiment Tracking (highest-value targets — they contain *everything*)
- **MLflow Tracking Server** — port 5000. Stores experiments, hyperparameters, metrics, model artifacts = complete ML research history.
- **Kubeflow** — ports 80/443. Kubernetes-native; orchestrates pipelines, notebooks, deployments.
- **Ray** — dashboard on 8265, server on 8000. (Target of the ShadowRay campaign.)

### Vector Databases (power RAG / semantic search)
| DB | Ports | Notes |
|---|---|---|
| Qdrant | 6333 (HTTP), 6334 (gRPC) | |
| Weaviate | 8080 | Has GraphQL endpoint |
| Milvus | 19530 | |
| Chroma | 8000 | |

Schema/collection endpoints leak the embedding model used and hint at data sensitivity (e.g., a collection named `internal-hr-policies`).

### Model Registries
Store serialized model files (`.pkl`, `.pt`, `.onnx`, `.mar`) + version history, stage transitions, artifact URIs, and creator identity. **Single highest-value target** — maps the entire ML product portfolio.

### Supporting Infrastructure
- **Jupyter Notebooks (8888)** — often run with `-ip=0.0.0.0`, no auth → direct terminal access. Frequently contain cleartext credentials.
- **MinIO (9000/9001)** — S3-compatible storage for artifacts; bucket listing often enabled.
- **Prometheus metrics** (Triton 8002, TorchServe 8082) — leak model names, batch sizes, GPU utilization, latency → full deployment topology without touching inference API.

---

## 3. Port & Protocol Reference Table

| Component | Port(s) | Protocol(s) | Recon Endpoints |
|---|---|---|---|
| NVIDIA Triton | 8000, 8001, 8002 | HTTP, gRPC, Prometheus | `/v2/health/ready`, `/v2/models` |
| TensorFlow Serving | 8500, 8501 | gRPC, HTTP | `/v1/models/<name>` |
| TorchServe | 8080, 8081, 8082 | HTTP | `/ping`, `/models` |
| Ollama | 11434 | HTTP | `/api/tags`, `/api/show` |
| vLLM | 8000 | HTTP | `/v1/models` |
| MLflow | 5000 | HTTP | `/api/2.0/mlflow/experiments/search` |
| Kubeflow | 80, 443 | HTTP | `/pipeline/apis/v1beta1/pipelines` |
| Ray | 8265, 8000 | HTTP | `/api/jobs/`, Ray Dashboard |
| Qdrant | 6333, 6334 | HTTP, gRPC | `/collections` |
| Weaviate | 8080 | HTTP, GraphQL | `/v1/schema`, `/v1/meta` |
| Milvus | 19530 | gRPC | Port connection |
| Jupyter | 8888 | HTTP | `/api/kernels`, `/api/contents` |
| MinIO | 9000, 9001 | HTTP (S3-compatible) | Bucket listing |
| Prometheus metrics | 8002, 8082 | HTTP | `/metrics` |

**Useful Shodan dorks:**
```
port:5000 "MLflow"
port:8888 title:"Home Page - Select or create a notebook"
http.title:"Ray Dashboard"
port:8001 "triton"
```

---

## 4. Fingerprinting AI Services

Default `nmap -sV` frequently mislabels AI services (e.g., calls port 8000 "http-alt"). Need deeper techniques:

### HTTP Header Fingerprinting
- **TorchServe** → `Server: TorchServe/0.x.x` (unambiguous)
- **Triton** → `NV-Status` header; sending request header `endpoint-load-metrics-format: text` returns GPU/CPU telemetry directly in headers (unique to Triton)
- **FastAPI-based ML services** → `server: uvicorn`, combined with `/predict` or `/embeddings` routes
- **OpenAI-compatible wrappers** (vLLM, LiteLLM, Ollama) → `x-request-id` header + `"object": "model"` JSON field on `/v1/models`

### API Response Signatures
- TensorFlow Serving: `{"model_version_status": [{"version": "1", "state": "AVAILABLE"}]}`
- Triton: `{"name": "fraud_detector", "versions": ["1"], "platform": "tensorflow_graphdef"}`
- MLflow errors reference `mlflow.server` / `mlflow.tracking` namespaces
- OpenAI-compatible: `{"object": "model", "id": "llama-3.1-8b", "created": 1700000000}`

### Error Message Fingerprinting
AI APIs are rigid about tensor shapes/dtypes — malformed input triggers revealing errors:
- TF Serving → error mentions `tensorinfo_map`
- MLflow → stack trace shows `mlflow.server`, `mlflow.tracking`, `databricks`
- MLflow path traversal (**CVE-2024-1558**) exposes full filesystem paths
- Databricks Mosaic AI → Java exception `io.jsonwebtoken.IncorrectClaimException`

### Endpoint Naming Conventions
- Inference: `/predict`, `/invocations` (SageMaker), `/infer`, `/generate`, `/embeddings`, `/score`
- Model mgmt: `/v1/models`, `/v2/models`
- MLflow internal API prefix: `/api/2.0/mlflow/` (distinctive, no other framework uses it)
- Kubeflow: `/pipeline/apis/v1beta1/`

### gRPC Fingerprinting
- Triton on port 8001, TF Serving on port 8500 — invisible to standard HTTP scanners.
- Tool: `grpcurl`
```bash
grpcurl -plaintext target:8001 list
grpcurl -plaintext target:8001 describe inference.GRPCInferenceService
```
If reflection is enabled, dumps the full protobuf schema (like finding `/openapi.json` on REST).

### TLS Fingerprinting (JA3/JA4)
Internal AI service-to-service traffic is dominated by Python libs (requests, urllib, gRPC) rather than browsers → distinctive JA3/JA4 signatures can distinguish automated ML pipeline traffic from human browsing. (GreyNoise found 99% of attack traffic in one campaign shared the same JA4H signature across 62 IPs / 27 countries.)

---

## 5. Enumerating AI Systems

**Fingerprinting** = "that's an MLflow server." **Enumeration** = "that MLflow server has 4 experiments, 3 production models, artifact URIs to S3, created by named employees." One is identification, the other is intelligence.

### MLflow Enumeration Chain
1. `POST /api/2.0/mlflow/experiments/search` → experiment names/IDs (often reveal project codenames)
2. `GET /api/2.0/mlflow/registered-models/list` → full model inventory
3. `GET /api/2.0/mlflow/model-versions/search` → **artifact URI** (e.g. `s3://internal-ml-models-corp/...`), creator `user_id`, timestamps, stage labels
4. `POST /api/2.0/mlflow/runs/search` → hyperparameters, metrics, tags (codenames, Git commit hashes, env identifiers)
5. `GET /api/2.0/mlflow/artifacts/list` → downloadable model files

Five API calls = entire ML portfolio mapped.

### Inference Server Metadata
- Triton: `GET /v2/models/<name>/config` → input tensor names, shapes, dtypes (FP32/UINT64/INT8), max batch size, backend framework
- TF Serving: `GET /v1/models/<name>/metadata` → same input/output tensor specs

This is the equivalent of dumping a database schema — tells attacker exactly how to craft a valid inference request.

### Vector Database Enumeration
- **Weaviate**: `/v1/meta` (version/modules), `/v1/schema` (class defs + vectorizer module → reveals embedding model), `/v1/graphql` (full introspection if unauthenticated)
- **Qdrant**: `/collections` (list), `/collections/<name>` (dimensions, distance metric, point count)
- **Chroma**: older versions expose `/api/v1/collections` unauthenticated

### Prometheus Metrics as Passive Intelligence
`/metrics` on Triton (8002) / TorchServe (8082) reveals model names/versions, request counts, latency, batch sizes, GPU memory — **without sending a single request to the inference API**.

### Debug Interfaces / Info Leakage
- FastAPI auto-generates `/docs` (Swagger) and `/openapi.json` — full schema, auth requirements, example payloads.
- MLflow's `/graphql` has historically bypassed REST auth — resolvers like `mlflowSearchRuns`/`mlflowGetRun` leak usernames, source paths, project inventory.
- `?debug=true` / `?verbose=1` on AI gateways can trigger stack traces with filesystem paths, library versions, sometimes hardcoded creds.

### Jupyter Enumeration
`GET /api/kernels` → running kernel IDs/timestamps. Real value = notebook cell contents, which routinely contain cleartext `MLFLOW_TRACKING_USERNAME`/`PASSWORD`, cloud keys, HF tokens. **This is the classic bridge from one compromised service to the whole stack.**

---

## 6. Mapping the Attack Surface

- The value of recon isn't the individual findings — it's the **connections** between them (credentials → registry → storage).
- AI services constantly talk to each other (inference ↔ vector DB, orchestrator ↔ registry, Jupyter ↔ everything, Prometheus ↔ all). Binding to `0.0.0.0` on any one exposes the whole internal mesh.
- Security perimeter for AI ≠ external firewall; it extends into internal service-to-service pathways. Missing mTLS + permissive network policy = norm, not exception.

### Documented Platform Misconfigurations
- **MLflow**: no auth by default pre-2.x; `CVE-2026-2635` — hardcoded default creds in `basic_auth.ini`; `CVE-2026-2033` — directory traversal → unauthenticated RCE (both CVSS 9.8).
- **Kubeflow**: dashboards often deployed without OIDC, exposed via LoadBalancer/NodePort → unauthenticated user can spawn notebooks tied to cluster-level service accounts.
- **TorchServe**: management API (8081) allows dynamic model registration from arbitrary URLs → malicious `.mar` file execution = RCE.
- **SageMaker**: notebooks with `DirectInternetAccess: Enabled` accept inbound internet connections — 82% of orgs using SageMaker had ≥1 notebook configured this way (2024 report).

### Model Registries = Highest-Value Target
Store: model names, version history, stage labels, timestamps, run IDs (→ training metadata), artifact URIs (→ storage paths), contributor user IDs. One open registry = entire ML portfolio.

Exploitation pattern (IBM X-Force): Jupyter creds found → MLOKit run against registry → every model artifact exfiltrated.

### Supply Chain Reconnaissance
- HF tokens leak via GitHub dorks (`filename:.env HF_TOKEN`), `.env` files, CI/CD logs, K8s secrets.
- **Dependency confusion**: internal package names (e.g. `company-data-utils`) not registered on PyPI can be squatted; Kubeflow pipelines that build containers at training time pull packages live → code exec in training cluster.
- Model download sources (HF Hub, PyTorch Hub) visible in configs/notebooks/build logs → poisoning risk if attacker controls upstream source or compromised token.

---

## 7. The 5-Phase Recon Methodology

### Phase 1 — Passive Reconnaissance
- Shodan / Censys / FOFA banner searches (same dorks as above).
- GitHub dorks for leaked creds: `filename:.env MLFLOW_TRACKING_URI`, `filename:.env HF_TOKEN`, `filename:config.json model_name site:github.com`.
- arXiv papers / engineering blogs revealing architecture choices.
- DockerHub / GHCR for org-named ML images with hardcoded configs.
- Job postings (e.g. "MLflow Administrator") reveal deployed stack.
- Maps to **ATLAS AML.T0000** — Search for Victim's Publicly Available Research Materials.

### Phase 2 — Active Scanning
```bash
nmap -p 5000,6333,8000,8001,8002,8080,8265,8500,8501,8888,9000,11434,19530 \
     -sV --script=http-title,http-headers <target>
```
- Watch for gRPC on 8001/8500 reported generically by nmap → follow up with `grpcurl`.
- Check every discovered service for `/metrics`.

### Phase 3 — API Fingerprinting
Run `ffuf`/`feroxbuster` with an AI-specific wordlist (standard SecLists won't include these):
```
/v1/models  /v2/models  /v2/health/ready
/api/2.0/mlflow/experiments/list  /api/2.0/mlflow/registered-models/list
/pipeline/apis/v1beta1/pipelines  /api/serve/deployments/
/v1/schema  /v1/meta  /api/kernels  /api/contents
/openapi.json  /docs  /graphql  /metrics
/api/tags  /api/show  /collections  /healthz  /ping
```
For every 200 response, apply Task-3 fingerprinting (headers/JSON/errors).

### Phase 4 — Metadata Extraction
Run the full enumeration chain from Section 5 against every confirmed service (MLflow 5-call chain, Triton/TF config endpoints, vector DB schema, Jupyter kernels/contents).

### Phase 5 — Supply Chain Review
- Identify model download sources in configs/notebooks/build logs.
- Check if artifact buckets (S3/GCS/MinIO) are publicly readable.
- Audit `requirements.txt`/`Pipfile` for squattable internal package names.
- Check container registries for unauthenticated pull access.

---

## 8. Detection — What Recon Looks Like in a SIEM

| Pattern | Signature |
|---|---|
| **Model enumeration** | Burst of sequential GETs to `/v2/models` from one IP (10-50 reqs/seconds, single source) |
| **Scripted MLflow access** | Calls to `/registered-models/list`, `/model-versions/search` with **no** corresponding UI session/cookie — matches MLOKit's pattern |
| **Prometheus scraping from outside monitoring stack** | `/metrics` requests from IPs outside the known Prometheus CIDR |
| **AI-aware port scanning** | Sequential hits on 5000, 8000, 8001, 8080, 8265, 8888 from same source — matches the Phase-2 nmap command |
| **Path traversal against MLflow artifacts** | `../` or `%2e%2e%2f` in requests → probing for CVE-2026-2033 |
| **Jupyter access without session** | `/api/kernels`, `/api/contents` requests without valid session cookie |

---

## 9. Quick Wins / Hardening Checklist

- [ ] Enable MLflow auth (`MLFLOW_TRACKING_USERNAME` / `MLFLOW_TRACKING_PASSWORD`) or put behind authenticating reverse proxy
- [ ] Disable Jupyter `--allow-root` / `--ip=0.0.0.0` in production; require token auth; never bind 0.0.0.0 without VPN/ingress auth
- [ ] Block AI-specific ports at perimeter: 5000, 8000-8002, 8080, 8265, 8500/8501, 8888, 9000 — never internet-facing without explicit intent
- [ ] Disable Triton's model control endpoint (`--model-control-mode none`) in production
- [ ] Restrict Prometheus `/metrics` to internal monitoring CIDR only
- [ ] Rotate/scope Hugging Face tokens — fine-grained, read-only, minimal scope
- [ ] Strip debug headers / verbose errors from ML service responses before reaching untrusted networks
- [ ] Audit MinIO/S3 bucket policies hosting model artifacts — weights should never be publicly readable

---

## 10. Case Studies

### IBM X-Force — ML Training Infrastructure Attack Chain (2025)
Phishing → Azure ML access → Jupyter notebook contained cleartext MLflow creds → MLOKit (X-Force's purpose-built CLI) automated enumeration (`list-models`, `download-model`) → artifact URIs → S3 → full model portfolio exfiltrated via normal, "authenticated" REST calls. Each component trusted the one before it — the failure was trust chaining, not a single software bug. MLOKit later extended with `list-notebooks`, `add-notebook-trigger` (code exec via lifecycle configs), `poison-model` (overwrite artifacts). Open-source on GitHub.

### GreyNoise — LLM Endpoint Reconnaissance (Oct 2025–Jan 2026)
91,000+ attack sessions via Ollama honeypots. One campaign: 80,000+ requests over 11 days probing conversational endpoints for OpenAI/Gemini/Anthropic schema compatibility across 73+ model endpoints. Used simple, innocuous test prompts ("hi", "How many states in the US?", "count the r's in strawberry") — response structure alone reveals which model/proxy is behind the endpoint, no content-moderation triggers. IPs overlapped with infrastructure used for 200+ known CVE exploits (incl. React2Shell CVE-2025-55182) — pure fingerprinting feeding a larger exploitation pipeline.

### ShadowRay Campaign (CVE-2023-48022)
Ray's Job Submission API (port 8265) shipped without auth *by design* (Anyscale disputed the CVE). 230,000+ exposed Ray dashboards found via Shodan. Malicious jobs submitted via `/api/jobs/`: stage 1 read `/etc/passwd`, ran `printenv` to harvest AWS IAM/cloud creds → lateral movement. Primary goal: GPU hijacking for XMRig cryptomining (capped at 60% CPU, disguised as Linux kernel workers to evade detection). **ShadowRay 2.0** (late 2025) added LLM-generated adaptable payloads, persistence via cron/systemd, GitLab→GitHub payload migration, and `sockstress` TCP-exhaustion attacks — a full multi-purpose botnet.

### Hugging Face Spaces Breach (June 2024)
Attackers accessed Spaces platform, harvested developer-stored auth secrets/HF tokens with access to private models/datasets/configs. Directly illustrates Phase 1 (passive recon of leaked tokens) and Phase 5 (supply chain) risk. HF's response = the same "quick wins": revoke compromised tokens, migrate to fine-grained scoped tokens, stop storing tokens in app secrets.

---

## 11. Framework Mappings

### MITRE ATLAS
| Room Content | ATLAS ID | Technique |
|---|---|---|
| Shodan/GitHub dorking for AI infra | AML.T0000 | Active Scanning / Search Public Research Materials |
| Locating registries/artifacts via unsecured APIs | AML.T0048 (also referenced as T0007) | Discover ML Artifacts |
| Exposed HF tokens, dependency confusion | AML.T0040 (also referenced as T0010) | ML Supply Chain Compromise |
| Enumerating LLM config / API schema compatibility | AML.T0069 (also referenced as T0014) | Discover LLM System Information / ML Model Family |
| All recon activities collectively | AML.TA0002 | Reconnaissance (Tactic) |

*(Note: the room content used slightly different technique numbers in different tasks — ATLAS IDs evolve; always confirm against the current ATLAS matrix.)*

### MITRE ATT&CK (Enterprise)
| Room Content | ATT&CK ID | Technique |
|---|---|---|
| Port scanning AI services | T1046 | Network Service Scanning |
| Extracting topology from metrics/metadata | T1592 | Gather Victim Host Information |
| Probing unauthenticated mgmt interfaces | T1595.002 | Vulnerability Scanning |
| Pre-engagement intel gathering | TA0043 | Reconnaissance (Tactic) |

### OWASP Top 10 for LLM Applications (2025)
| Finding | OWASP LLM ID | Risk |
|---|---|---|
| Exposed MLflow/Jupyter/unauth APIs | LLM05 | Improper Output Handling |
| Downloadable model artifacts from unsecured registries | LLM06 | Excessive Agency |
| Leaked HF tokens, dependency confusion, poisoned models | LLM03 | Training Data Poisoning / Supply Chain Vulnerabilities |
| Default creds / missing auth (MLflow, Kubeflow) | LLM10 | Model Theft |

### NIST AI RMF 1.0 (Govern / Map / Measure / Manage)
- **Map 1.1** — AI system components & interactions identified (Tasks 2-4: discovery).
- **Map 1.5** — Potential AI system risks assessed (Task 5: attack surface mapping).
- **Map 3.2** — Third-party AI resource risks identified (supply chain — HF tokens, hub dependencies).
- **Measure 2.6** — Verifying AI systems function as intended (Prometheus/debug endpoint enumeration reveals systems *not* behaving as intended, security-wise).

### NIST Cybersecurity Framework (CSF 2.0)
- **ID.AM** (Asset Management) — discovering/inventorying AI infra components.
- **ID.RA** (Risk Assessment) — mapping AI attack surface, identifying misconfigurations.

---

## 12. Tool Reference

| Tool | Purpose | Phase |
|---|---|---|
| Shodan / Censys / FOFA | Internet-wide AI service banner search | 1 |
| GitHub dorks | Leaked credentials/configs in public repos | 1 |
| Nmap (+NSE scripts) | Port discovery, service version detection | 2 |
| grpcurl | gRPC interaction, protobuf schema dump (if reflection enabled) | 2 |
| ffuf / feroxbuster | Directory brute-forcing w/ AI-specific wordlists | 2, 3 |
| curl | Manual HTTP probing, headers, error triggering | 3, 4 |
| MLOKit (IBM X-Force Red) | Automated MLflow enumeration/model exfil | 4 |
| Nuclei | Template-based scanning for known AI misconfigs | 2, 3 |
| Agrus Scanner | Purpose-built shadow AI detection, 50+ probes across all ports | 2 |

---

## 13. Key Takeaways

1. AI infra triples network attack surface vs. a typical web app — know the non-standard ports (5000, 6333/4, 8000-8002, 8080, 8265, 8500/1, 8888, 9000/1, 11434, 19530).
2. **Fingerprinting ≠ Enumeration.** Fingerprinting identifies the framework; enumeration extracts the actual intelligence (models, credentials, data schemas, storage locations).
3. **MLflow / model registries are the single highest-value target** — 5 API calls can map an entire ML portfolio.
4. **Jupyter notebooks are the classic pivot point** — cleartext creds in cells routinely bridge one compromised service into the entire stack (see IBM X-Force chain).
5. Debug-friendly defaults (verbose errors, `/docs`, `/graphql`, `?debug=true`) rarely get turned off before production — huge free intel source.
6. Passive channels (GitHub dorks, Shodan, job postings, published papers) often reveal as much as active scanning — check them first.
7. Everything you do as an attacker leaves a signature — scripted API calls without UI sessions, sequential port hits, off-CIDR Prometheus scraping are all detectable patterns.
8. Map every finding to a recognized framework (ATLAS/ATT&CK/OWASP LLM/NIST) — this is what turns a raw finding into a communicable, actionable risk for stakeholders.
9. Next step in the learning path: **AI Threat Modelling Assessment** — turning these recon findings into actual risk/impact analysis and mitigation prioritization.
