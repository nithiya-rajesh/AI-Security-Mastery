<div align="center">

# 🔍 AI Forensics — Cheatsheet

### Machine learning in DFIR — from anomaly detection to courtroom admissibility

<img src="https://img.shields.io/badge/Domain-AI_Security-6366f1?style=flat-square&labelColor=0d1117" />
<img src="https://img.shields.io/badge/Focus-DFIR-f43f5e?style=flat-square&labelColor=0d1117" />
<img src="https://img.shields.io/badge/Standard-Daubert-38bdf8?style=flat-square&labelColor=0d1117" />
<img src="https://img.shields.io/badge/Source-TryHackMe-2ecc71?style=flat-square&labelColor=0d1117" />

<i>Where AI accelerates forensic investigation — and where non-determinism, bias, and the black box make human validation non-negotiable.</i>

</div>

---

## 📑 Table of Contents

1. [Core DFIR Challenges Solved by AI](#1--core-dfir-challenges-solved-by-ai)
2. [Practical AI Implementations](#2--practical-ai-implementations-in-the-wild)
3. [Structural Limitations & Metrics](#3--structural-limitations--metrics)
4. [Applied AI Forensics](#4--applied-ai-forensics)
5. [Legal & Ethical Implications](#5-%EF%B8%8F-legal--ethical-implications)
6. [Case Study: The RobbCo Breach](#6--case-study-the-robbco-breach)
7. [Case Study: contAInment (IR Writeup)](#7--case-study-containment-ir-writeup)
8. [Quick Glossary](#-quick-glossary)

---

## 1. ⚡ Core DFIR Challenges Solved by AI

Traditional forensics drowns in data volume. AI/ML helps across three pillars:

- **High-Speed Data Processing** — parallelized deep learning and Transformers parse, classify, and index massive log/text volumes in milliseconds.
- **Anomaly Detection** — baseline models of "normal" system/network/user behavior isolate subtle deviations that signal sophisticated attacks.
- **Elastic Scalability** — ingests telemetry across cloud, hybrid, and remote endpoints without a linear rise in analyst workload.

---

## 2. 🛰️ Practical AI Implementations in the Wild

| DFIR Task | AI/ML Mechanism | Platforms | Forensic Outcome |
| :--- | :--- | :--- | :--- |
| **Anomaly Detection / UEBA** | Unsupervised models (Isolation Forests, Autoencoders) baseline behavior | Splunk UEBA, Elastic ML, Exabeam | Flags outliers without static rules |
| **Phishing Detection** | Transformer LMs (BERT, RoBERTa) analyze tone/structure/intent | Defender for O365, Splunk NLP | Detects impersonation & credential-harvesting links |
| **Malware Analysis** | Classifies static/dynamic features & sandbox output | Defender (STAMINA), Cylance, VirusTotal ML | Flags zero-days via feature-vector matching |
| **Alert Triage** | Scores events from historical triage & threat feeds | Cortex XSOAR/XSIAM, QRadar, Charlotte AI | Suppresses noise, surfaces high-severity incidents |
| **Timeline Correlation** | Clusters disparate logs chronologically, maps causality | Timesketch, Velociraptor, Jupyter | Reconstructs full attack path across hosts |

---

## 3. 🧮 Structural Limitations & Metrics

### A. The Nondeterministic Hurdle
- **Deterministic (traditional):** same input → identical output every time (e.g. a hash).
- **Probabilistic (AI):** outcomes predicted via statistics.
- 🛡️ **Forensic impact:** courts need **consistent, repeatable, admissible** results. Non-determinism can make an AI engine reconstruct slightly different timelines from *identical* log data across runs.

### B. Accuracy, Precision & Recall
Relying on accuracy alone causes catastrophic validation failures.

```
Accuracy  = Correct Predictions / Total Predictions
Precision = True Positives / (True Positives + False Positives)   ← minimizes false alarms
Recall    = True Positives / (True Positives + False Negatives)   ← minimizes missed threats
```

- **Accuracy** — ⚠️ misleading on **imbalanced datasets** (the forensic norm). Among 10,000 files with 1 malware, a model that always says "benign" scores **99.99% accuracy** while failing completely.
- **Precision** — of everything flagged malicious, how much *actually* was? High precision saves analyst time.
- **Recall / Sensitivity** — of all real threats present, how many were found? High recall ensures nothing malicious is left behind.

### C. Garbage In, Garbage Out (GIGO)
A model is only as reliable as its training corpus. Biased, incomplete, or corrupted forensic data produces **mathematically confident but false** assertions.

---

## 4. 🧠 Applied AI Forensics

### Image & Video Forensics
- **CNNs** map spatial/sequential patterns in visual data via learned filters.
- **Tamper detection (ELA + CNN):** Error Level Analysis highlights compression inconsistencies; paired with a CNN classifier it flags manipulated regions at ~**94% accuracy**.
- **Deepfake detection:** CNNs analyze micro-expressions, heartbeat-induced skin-color shifts, and rendering artifacts.
- **GANs:** Generator (creates fakes) vs. Discriminator (detects fakes). 🛡️ Defenders use this competition to pre-train detection engines on synthetic media before attackers deploy it.

### Communication Analysis (NLP)
- **Advanced phishing detection:** Transformer models (BERT, RoBERTa) read tone, grammar, and psychological urgency — studies show ~**99% accuracy** vs. benign mail.
- **Chat/social triage:** sentiment analysis flags threatening language, collusion, or espionage signals across large log volumes.

### Timeline Reconstruction & UBA
- **Automated timelines:** ingests heterogeneous, time-sequenced data (filesystem timestamps, connections, web/firewall logs) into one master timeline — effective even when local logs were cleared.
- **Behavioral anomaly detection:** flags "impossible travel" (login from India and USA within 30 min) or unusual command-line activity per account.

### Malware Detection & Analysis
- **STAMINA (static):** Microsoft + Intel project translates binaries into **pixel maps**; CNNs classify by visual similarity to known families.
- **Dynamic behavioral analysis:** converts runtime JNI/API call sequences into a processable feature sequence (often a 2D image) to detect malicious intent during execution.

---

## 5. ⚖️ Legal & Ethical Implications

### Explainability & Courtroom Admissibility
Forensics demands transparent, reproducible methods, but deep models are **black boxes**.
- **The clash:** if you can't explain *how* a model flagged evidence, opposing counsel can get it excluded.
- **The Daubert Standard (U.S.):** admissibility test for expert/scientific testimony — AI evidence must prove reliability, error rate, peer-reviewed methodology, and **explainability**.
- **Rule of thumb:** AI is a *guiding light* for triage; final conclusions must be validated and testified to by a human.

### Bias, Fairness & Algorithmic Injustice
- Skewed training data → compromised predictions (e.g. overlooking non-English indicators).
- ⚠️ **Case study:** faulty facial recognition has misidentified minority groups at disproportionate rates, causing wrongful arrests.
- **Mandate:** demonstrable bias can render evidence inadmissible; analysts have a duty to validate tools against balanced datasets.

### Integrity & Chain of Custody
- **Pipeline risk:** feeding evidence into cloud AI (e.g. public ChatGPT APIs) can **break chain of custody** if intermediate outputs aren't logged or data sits on third-party servers.
- **Mitigation:** document, audit, and run forensic AI on **secure, on-prem, or isolated** systems.

### Privacy & Compliance
- Uploading evidence to public cloud APIs can violate **GDPR/HIPAA** and compromise court orders.
- 🛡️ **Solutions:** isolated/air-gapped deployments of offline open-weights models (LLaMA, Mistral); **federated learning** to train without moving raw data.

---

## 6. 🕵️ Case Study: The RobbCo Breach

**Target:** RobbCo (firmware & OS developer). **Trigger:** anomalous off-hours login; proprietary source (RETROS BIOS, MF Boot Agent, UOS) targeted.
**Setup:** local **Scikit-Learn** classifiers (`classify_logs.py`, `file_anomalies.py`) isolate anomalous artifacts.

### Attack Lifecycle
```
[Phishing Lure]        [Foothold]           [Sudo Abuse]              [Evasion & Exfil]
akeane@poseidon ─► j.morgan (.ods) ─► /tmp/.x reverse shell ─► authorized_keys ─► /dev/shm archive
```

- **Phase I — Initial Access:** phishing from `akeane@poseidonenergy.net`; malicious `invoice_Q1_2075.ods` macro harvested `.bash_history`, SSH configs, `/etc/passwd`; staged to `/tmp/invoice_dump.txt`, exfil to `192.168.0.100`; SSH foothold as `j.morgan` at 03:01:02.
- **Phase II — C2:** dropper `/tmp/.syncd` (fetches from `http://10.0.0.66/payload.sh`); reverse shell `/tmp/.x` → `10.0.0.66:4444`.
- **Phase III — Privilege Escalation:** no kernel exploit — abused `sudo` to append their SSH key to root-equivalent `r.house` via `authorized_keys`.
- **Phase IV — Persistence:** backdoor `/usr/local/bin/sysmon` → `10.0.0.66:5555` (disguised as monitoring); fake log `/opt/robbco/sys/boot_monitor.log` to justify CPU usage.
- **Phase V — Data Theft:** targeted `mfboot_main.c`, `core.asm`; staged base64 archive in Linux **shared memory** `/dev/shm/.core_dump_2025.tgz.enc` to evade filesystem monitoring.

> 🧑‍⚖️ **Human-in-the-loop:** the ML script flagged the core source files as "suspicious" — human review confirmed **False Positive** (legitimate source), but confirmed the `/dev/shm` archive as the real staging payload. Illustrates why AI triage needs human validation.

### Recovery
```bash
# Decode the base64 payload from shared memory
base64 -d /dev/shm/.core_dump_2025.tgz.enc > /tmp/stolen.tar.gz

# Extract to a secure forensic folder
mkdir /tmp/stolen_source
tar -xzvf /tmp/stolen.tar.gz -C /tmp/stolen_source
```

**IoCs:** `akeane@poseidonenergy.net` · `invoice_Q1_2075.ods` · `/tmp/invoice_dump.txt` · `/dev/shm/.core_dump_2025.tgz.enc` · SSH `authorized_keys` abuse via `sudo`.

---

## 7. 🧬 Case Study: contAInment (IR Writeup)

**Org:** West Tech (defense R&D). **Asset:** workstation of `o.deer`. **Threat:** ransom note; proprietary files exfiltrated, encrypted, and locked. **Goal:** find the exfil vector, recover the key, isolate the real flag among 500 decoys.

### Phase I — Discovery
```bash
# Find packet captures across the filesystem
find / -name "*.pcap*" 2>/dev/null
# → /home/o.deer/Documents/pcap_dumps/2025-06-17/session_XXXX_dump.pcap
```

### Phase II — Protocol Analysis
```bash
# Extract readable ASCII from the captures
strings /home/o.deer/Documents/pcap_dumps/2025-06-17/*
```
The PCAP revealed a successful **AI prompt-injection session** against a local LLM (bypassing guardrails to leak config). An obfuscated string was found:
```
w#e@%s~t^t-e$c*h_v^i%ct_im_1   →  westtechvictim1   (candidate password)
```

### Phase III — Decryption
```bash
unzip -P "westtechvictim1" westtech_projects_encrypted.zip
# → thm_flags.txt (500 base64 candidates) + thm_flags_guide.txt (criteria)
```

### Phase IV — Flag Verification (Python)
Real flag decodes to `thm{n1,n2,n3,n4,n5}` and contains **exactly 3 primes** between 10 and 99.

```python
#!/usr/bin/env python3
"""TryHackMe - contAInment : Automated Cryptographic Flag Validator"""
import base64

def is_prime(n):
    if n < 2:
        return False
    for i in range(2, int(n ** 0.5) + 1):
        if n % i == 0:
            return False
    return True

def main():
    target_path = "westtech_projects/thm_flags.txt"
    try:
        with open(target_path, "r") as file:
            for line_num, line in enumerate(file, 1):
                line = line.strip()
                try:
                    decoded = base64.b64decode(line).decode("utf-8")
                except Exception:
                    continue
                if not decoded.startswith("thm{") or not decoded.endswith("}"):
                    continue
                try:
                    nums = list(map(int, decoded[4:-1].split(",")))
                except ValueError:
                    continue
                if len(nums) != 5:
                    continue
                prime_count = sum(1 for n in nums if 10 <= n <= 99 and is_prime(n))
                if prime_count == 3:
                    print(f"[*] Match Found at Line {line_num}!")
                    print(f"[REAL FLAG] --> {decoded}")
                    break
    except FileNotFoundError:
        print(f"[-] Error: Could not locate target file at {target_path}")

if __name__ == "__main__":
    main()
```

### Lessons Learned
1. **Strict prompt security** — the breach began with **indirect prompt injection**; enforce input validation, hard system-prompt rules, and API monitoring.
2. **Network monitoring integrity** — continuous packet capture was what made the AI-compromise path auditable at all.
3. **Local secrets hardening** — obfuscated single-file passwords are a prime harvesting vector; use secrets management and restrict directory access.

---

## 📝 Quick Glossary

| Term | Meaning |
| :--- | :--- |
| **Anomaly Detection** | Finding structural deviations from a baseline |
| **UEBA** | User & Entity Behavior Analytics — behavioral baselining |
| **Precision** | TP / (TP + FP) — of flagged items, how many were real |
| **Recall / Sensitivity** | TP / (TP + FN) — of real threats, how many were found |
| **Non-determinism** | Probabilistic outputs; inputs don't map to fixed results |
| **CNN** | Neural net for spatial patterns (images, video, API streams) |
| **GAN** | Generator vs. Discriminator dual-network architecture |
| **Sentiment Analysis** | Extracting emotional/behavioral cues from text |
| **STAMINA** | Binary-to-pixel-map malware classification (Microsoft/Intel) |
| **Dynamic Analysis** | Analyzing behavior during active execution |
| **Daubert Test** | U.S. standard for admissibility of expert/scientific evidence |
| **Black Box** | A model whose internal weights defy human interpretation |
| **Chain of Custody** | Documented trail of evidence seizure, control, and analysis |
| **Federated Learning** | Decentralized training that keeps raw data local |

---

<div align="center">

📚 <b>Learning source:</b> TryHackMe — <i>AI Security</i> path (AI Forensics) · Notes compiled by <a href="https://github.com/nithiya-rajesh">Nithiya Rajendran</a>

<i>"AI is the guiding light for triage — the human signs the report."</i>

</div>
