***

# 🤖 AI Security & LLM Red Teaming
### Mastering AI/ML Vulnerabilities, Prompt Security, and Defensive Systems

This repository contains my research, notes, and practical lab write-ups from the **AI Security Learning Path**. The curriculum focuses on the intersection of Machine Learning and Cybersecurity, covering everything from prompt injection to advanced data poisoning in RAG systems.

---

## 🗺️ Learning Roadmap

### 📦 Module 1: AI Fundamentals & Threats
*   **Concepts:** Evolution of AI/ML/DL, Transformer Architectures, and the MITRE ATLAS Framework.
*   **Key Lab:** [AI/ML Security Threats](./Module_01_Threats.md)

### 🛡️ Module 2: Secure AI Systems
*Focus on the architecture and threat modeling of AI-integrated environments.*
*   **Securing AI Systems:** Implementing RBAC and MFA for model access.
*   **LLM Security:** Mapping the OWASP Top 10 for LLMs.
*   **AI Threat Modelling:** Applying frameworks (STRIDE/ATLAS) to AI assets.
*   **System Reconnaissance:** Identifying AI endpoints and model versions.

### ⌨️ Module 3: Prompt Security
*The "Front-End" of AI security—protecting the model from malicious user input.*
*   **Prompt Injection:** Overriding system instructions to leak data or bypass filters.
*   **Jailbreaking:** Techniques used to "break" the safety alignment of LLMs.
*   **Prompt Defence:** Implementing "Guardrails" and input sanitization.
*   **Labs:** White Rabbit, LLMborghini (Practical bypass scenarios).

### ⛓️ Module 4: AI Supply Chain Security
*Securing the lifecycle of the model, from training data to deployment.*
*   **Supply Chain Understanding:** Risks in pre-trained models (HuggingFace/GitHub).
*   **Attack Vectors:** Model backdooring and dependency confusion in AI libraries.
*   **Securing the Chain:** Signed models, checksums, and trusted environments.

### 🧪 Module 5: Data Poisoning & RAG Security
*Advanced attacks targeting the knowledge base and retrieval mechanisms.*
*   **RAG Security Fundamentals:** Securing Retrieval-Augmented Generation (LLMs + External Data).
*   **Data Poisoning in RAG:** Injecting malicious entries into vector databases to manipulate AI output.
*   **Sensitive Information Disclosure:** Techniques to trick RAG systems into leaking internal documents.

---

## 📝 AI Security Cheatsheet

| Attack Type | Domain | Description |
| :--- | :--- | :--- |
| **Prompt Injection** | Input | Using clever phrasing to ignore original system instructions. |
| **Jailbreaking** | Logic | Using roleplay or "DAN" style prompts to force the AI to provide restricted info. |
| **Data Poisoning** | Training/RAG | Contaminating the source data so the model learns incorrect/malicious logic. |
| **Model Theft** | API | Querying the API repeatedly to "clone" the internal weights of a model. |
| **Privacy Leakage** | Output | The model revealing patient names or PII from its training corpus. |

---

## 🛠️ Security Toolkit
*   **Monitoring:** SHAP, LIME (Model Explainability).
*   **Frameworks:** MITRE ATLAS, OWASP LLM Top 10.
*   **Standards:** ISO/IEC 27090.
*   **Environment:** Python, Docker, Jupyter Notebooks, LangChain.

---

## 👩‍💻 About the Author
**Nithiya Rajendran**  
*Security Researcher | Penetration Tester | AI in Offensive Security Enthusiast*

- **LinkedIn:** [Nithiya Rajendran](https://www.linkedin.com/in/nithya-rajendran/)
- **Write-ups:** [Medium Portfolio](https://medium.com/@nithiya.rajesh1998)
- **Certifications:** Security+, CCNA, CEH (In-Progress)

---
*Disclaimer: All research and code here are for educational and ethical testing purposes only. Knowledge is the ultimate defense.* 🚀

***

