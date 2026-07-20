# AI Supply Chain Security — Study Notes

> Notes summarizing key concepts on AI/ML supply chain risks, attack surfaces, and real-world incidents.

---

## 1. What Is a Supply Chain?

**Definition:** A network of dependencies where each component (or supplier) relies on other components, forming a chain of trust. The finished product is only as secure as its **weakest link**.

**Analogy:** Building a house — you don't make your own bricks, timber, wiring, or plumbing. You trust suppliers, who trust *their* suppliers.

### Software Supply Chains
- A modern app depends on many third-party packages.
- Installing a package means trusting not just it, but every **transitive dependency** (a dependency pulled in automatically, not chosen directly).
- Supply chain attacks **exploit trust** rather than bypassing defenses — attackers compromise something the victim already trusts.

### Classic Examples

| Incident | Year | What Happened | Impact |
|---|---|---|---|
| **SolarWinds** | 2020 | Malicious code injected into the Orion build process; shipped via a trusted update | ~18,000 orgs compromised, incl. US govt agencies |
| **Log4Shell** | 2021 | Critical RCE vulnerability in the widely-used Log4j library | Millions of Java apps affected |

**Key insight:** Supply chain attacks scale efficiently — compromising one widely-used component grants access to every system depending on it.

---

## 2. The AI Supply Chain

AI introduces a new artifact type beyond code: **trained models**.

A trained model = architecture + training data + training process + **serialized weights**.

> **Analogy:** Model weights are like a video game save file — it loads and plays correctly, but a stranger's save file could have something malicious embedded alongside the legitimate data, invisible until it matters.

- Pickle-based model files (`.pkl`, `.pt`, `.bin`) contain **serialized objects** that can execute code the moment `model.load()` is called.
- Most of what you're trusting (data, training process, architecture) **cannot be inspected or verified**.

### Transfer Learning & Fine-Tuning Risk
- **Transfer learning:** adapting someone else's pre-trained model to your task.
- **Fine-tuning:** adjusting weights on a smaller, task-specific dataset — fast, but builds on an unverified foundation.
- **Backdoors inserted during pre-training can survive fine-tuning** — one poisoned base model compromises every downstream app built on it.
- **LoRA adapters** add a second trust dependency: a clean base model + malicious adapter = still compromised.

### The Four Components of an AI Supply Chain

| Component | What It Is | Where It Comes From | Example |
|---|---|---|---|
| **Models** | Pre-trained weights & architectures | Hugging Face Hub, TensorFlow Hub, PyTorch Hub | `bert-base-uncased`, `gpt2` |
| **Datasets** | Training/evaluation data | HF Datasets, Kaggle, academic repos | ImageNet, Common Crawl |
| **Frameworks** | ML libraries | PyPI, conda, GitHub | PyTorch, TensorFlow, scikit-learn |
| **Dependencies** | Supporting packages | PyPI, npm, conda-forge | NumPy, SciPy, Pillow, tokenizers |

**Key insight:** Every arrow in the architecture = a trust relationship (app → framework → registry → uploader). Breaking any link compromises the whole system.

---

## 3. Two Ways to Consume AI

### Paradigm 1: Downloading Model Files
You download weights and run them on your own infrastructure — you own hosting, inference, and security.

| Format | Pickle-based? | Risk |
|---|---|---|
| `.pkl`, `.pt`, `.bin` | Yes | Can execute arbitrary code on load |
| `.safetensors` | No | Stores raw weights only — no executable code |
| `.h5` (Keras) | No | Not pickle, but can embed executable code in layers |
| `.gguf` (local LLMs: LLaMA, Mistral, Qwen) | No | No serialization exploit, but weight-level attacks still apply |

- GGUF models are usually **quantized** (compressed from 32-bit to 4/8-bit). If a third party did the quantization, that's another unverified transformation in the chain.

### Paradigm 2: Calling Models via API
You send prompts to a hosted provider (OpenAI, Anthropic, Google, OpenRouter) and never touch the model file.

| Supply Chain Element | Download Paradigm | API Paradigm |
|---|---|---|
| Model weights | You inspect/scan | Provider controls; can't inspect |
| Training data | Often documented | Often undisclosed |
| Fine-tuning | You control | Provider may fine-tune silently |
| System prompts | N/A | Untrusted templates can alter behavior |
| Versioning | You pin a file hash | Provider may swap the model behind the same endpoint |
| Hosting | Your infra | Provider's shared, multi-tenant infra |

**Key insight:** API calls don't eliminate the supply chain — they just hide it behind a JSON response.

---

## 4. The Four Attack Layers (Download Paradigm)

### Layer 1 — Model Layer
Most AI-distinctive attack surface. Three levels:
- **Serialization-level:** code hidden in the file format, executes on load
- **Architecture-level:** malicious logic in model layers, executes on every prediction
- **Weights-level:** learned values subtly altered to misbehave on specific inputs

> ⚠️ Stripping executable code (e.g., using SafeTensors) stops serialization-level attacks only — architecture- and weights-level attacks remain.

### Layer 2 — Dependency Layer
Packages/libraries your ML project relies on. Attack methods: dependency confusion, typosquatting, malicious packages disguised with professional descriptions.

### Layer 3 — Data Layer
Poisoning training data is hard to detect.
- Replacing **~0.1%** of a dataset with crafted samples can implant a reliable backdoor without hurting clean-data accuracy.
- For denial-of-service goals, the threshold drops to **~0.001%**.

### Layer 4 — Infrastructure Layer
Systems that host/distribute/manage AI artifacts. Attacks: compromising repos, injecting malicious CI/CD steps, stealing maintainer credentials.

### The Compounding Effect
Layers can be chained in one attack, e.g.:
1. Compromise a Hugging Face account (infrastructure)
2. Upload a backdoored pickle model (model)
3. Add a `requirements.txt` with typosquatted packages (dependency)
4. Bundle a poisoned dataset (data)

A single download → compromise at every layer simultaneously.

---

## 5. Real-World Incidents (2022–2025)

| # | Incident | Date | Summary |
|---|---|---|---|
| 1 | **PyTorch `torchtriton` Dependency Confusion** | Dec 2022 | Attacker published a malicious package matching the name of an internal PyTorch nightly dependency; pip installed the public (malicious) version. Stole SSH keys, Git config, `/etc/passwd`, up to 1,000 home directory files via DNS exfiltration. Active 5 days before discovery. |
| 2 | **Hugging Face Hub Vulnerabilities** | 2023–2024 | 1,500+ exposed API tokens found (655 with write access); ~100 malicious pickle models found disguised as legitimate, executing code at load time. |
| 3 | **Ultralytics (YOLO) Build Pipeline Compromise** | Dec 2024 | GitHub Actions workflow compromised; cryptominer embedded into published PyPI packages — standard `pip install` delivered malware. |
| 4 | **@solana/web3.js Compromise** | Dec 2024 | Maintainer's npm credentials stolen; two backdoored versions published, exfiltrating crypto private keys. Losses: **$130K–$184K**. Not AI-specific, but same pattern (trusted identity → upstream compromise). |
| 5 | **NullifAI Scanner Evasion (Hugging Face)** | Feb 2025 | Deliberately corrupted pickle files bypassed HF's automated scanners entirely while still executing malicious code on load. |

**Common thread:** None of these required "breaking in." Victims trusted a component that had earned trust — and that trust *was* the vulnerability.

---

## 6. Key Takeaways

- 🧩 **Model files are code in disguise** — pickle-based models execute at load time; file format determines your exposure.
- 🎯 **The four attack layers are distinct** (model, dependency, data, infrastructure) — each needs different defenses and produces different forensic signals.
- 🔗 **Transitive dependencies silently expand your attack surface** — you're responsible for everything pip installs, not just what you listed.
- 🛡️ **Automated scanners ≠ complete defense** — NullifAI proved malformed files can pass platform checks while remaining executable.
- 🔑 **Infrastructure compromise beats content filtering** — stolen credentials on a trusted repo push malicious code with full legitimacy.
- ✅ **SafeTensors eliminates serialization-level attacks only** — not architecture- or weight-level attacks. It's one control, not a complete solution.

---

## Quick Reference: File Format Risk Cheat Sheet

```
.pkl / .pt / .bin   → Pickle-based → Arbitrary code execution risk
.safetensors        → No executable code → Safest for serialization
.h5 (Keras)         → Not pickle, but can embed executable layer code
.gguf                → No serialization exploit → Weight-level attacks still possible
```

---
