# Understanding AI Supply Chains

---

## What This Room Is About

Every time we use an AI product — Claude, ChatGPT, GitHub Copilot — we are trusting a model trained somewhere, on some data, by someone you have never verified. This room introduces the concept of AI supply chains: what they are, where they differ from traditional software supply chains, and where attackers target them.

**Framework Alignment:**
- OWASP LLM Top 10: **LLM03** (Supply Chain Vulnerabilities)
- MITRE ATLAS: **AML.T0010** (ML Supply Chain Compromise)

---

## Task 1 — Introduction

### The Core Threat

In 2024, security researchers found over 100 models on Hugging Face (the largest public platform for sharing AI models, often called the "GitHub of AI") that were:

- Fully functional — they did exactly what they claimed
- Professional-looking — thorough documentation, credible names, thousands of downloads
- Malicious — the moment you called `model.load()`, they opened a **reverse shell** to an attacker's server

This is the defining characteristic of supply chain attacks: **they exploit trust, not weakness**. You did nothing wrong. You trusted something that looked trustworthy.

---

## Task 2 — Traditional Software Supply Chains

### The House Analogy

Building software is like building a house. You don't manufacture every component yourself — you source bricks from one supplier, timber from another, wiring from a third. The finished house is only as secure as its weakest component.

Software works the same way. A modern Python app might depend on hundreds of third-party packages. When you run `pip install`, you are not just trusting the package you listed — you are trusting every **transitive dependency** (packages pulled in automatically by your dependencies, that you never chose yourself).

### Why Supply Chain Attacks Work

They scale efficiently. Compromise one widely-used component and you gain access to every system that depends on it.

**Real-world examples:**

| Incident | Year | What Happened | Impact |
|---|---|---|---|
| SolarWinds | 2020 | Malicious code injected into a software update — victims installed it willingly | ~18,000 organisations compromised including US government agencies |
| Log4Shell | 2021 | Critical bug in the Log4j logging library used by millions of Java apps | Affected millions of applications worldwide |

In both cases, the victims did everything right. The compromise happened upstream.

---

## Task 3 — How AI Supply Chains Differ

### From Code to Models

Traditional software supply chains deal with **code**. AI supply chains add a fundamentally different type of artefact: **trained models**.

A trained model is not just code. It is the product of four things:

1. **Architecture** — the neural network structure
2. **Training data** — potentially millions of examples
3. **Training process** — settings and hardware configuration
4. **Serialised weights** — the learned parameters saved to a file

### The Game Save File Analogy

Think of model weights like a saved game state. When you save your progress in a video game, the file captures everything — your position, inventory, completed quests. Model weights work similarly; they capture everything the model "learned" during training.

Now imagine downloading that save file from a stranger. It loads correctly, your character has the right stats, and the game plays as expected. But the stranger had access to every byte — they could have **embedded something alongside the legitimate data that stays invisible until it matters**.

### Why Pickle Is Dangerous

Most model files (`.pkl`, `.pt`, `.bin`) use Python's **pickle** format. Pickle doesn't just save data — it saves *instructions for reconstructing objects*. When you call `model.load()`, Python executes those reconstruction instructions. An attacker can embed malicious code in those instructions.

The model loads fine and works correctly — but arbitrary code ran silently alongside it.

### Transfer Learning Makes This Worse

Most teams don't train from scratch — they download a base model and fine-tune it (called **transfer learning**). This is faster and cheaper, but it creates a critical security problem:

> **Backdoors inserted during pre-training can survive fine-tuning.**

An attacker who poisons a popular base model compromises every downstream application that fine-tunes from it.

**LoRA adapters** add another trust dependency: your supply chain now includes the base model *and* the adapter. A clean base model paired with a malicious adapter is still a compromised model.

### The Four Components of an AI Supply Chain

| Component | What It Is | Where It Comes From | Example |
|---|---|---|---|
| **Models** | Pre-trained weights and architectures | Hugging Face Hub, TensorFlow Hub, PyTorch Hub | `bert-base-uncased`, `gpt2` |
| **Datasets** | Training and evaluation data | Hugging Face Datasets, Kaggle, academic repos | ImageNet, Common Crawl |
| **Frameworks** | ML libraries that train and run models | PyPI, conda, GitHub | PyTorch, TensorFlow, scikit-learn |
| **Dependencies** | Supporting packages frameworks rely on | PyPI, npm, conda-forge | NumPy, SciPy, Pillow, tokenizers |

Every arrow in the dependency graph is a trust relationship. Break any one link and everything above it is compromised.

---

## Task 4 — Two Ways to Consume AI

### Paradigm 1: Downloading Model Files

You download pre-trained weights from a repository and run them on your own infrastructure. Everything you load crosses your **trust boundary** and runs on your systems.

**File format determines your risk:**

| Format | Risk Level | Notes |
|---|---|---|
| `.pkl` `.pt` `.bin` | ☠️ High | Pickle-based — code executes on `model.load()` |
| `.safetensors` | ✅ Low | Raw weights only, no executable code |
| `.h5` | ⚠️ Medium | Not pickle, but can contain executable architecture-level code |
| `.gguf` | ⚠️ Medium | No serialisation exploit, but weight-level backdoors still apply |

**Quantisation adds another trust layer:** GGUF models are almost always quantised (compressed from 32-bit to 4/8-bit) by a third party. That compression step is itself an unverified point in the supply chain.

### Paradigm 2: Calling Models via API

You send a prompt to a provider (OpenAI, Anthropic, Google), they run inference, you receive a response. You never touch the model file.

This might seem safer — but the supply chain is still there, just hidden:

| Supply Chain Element | Download Paradigm | API Paradigm |
|---|---|---|
| Model weights | You inspect/scan them | Provider controls them; you cannot inspect |
| Training data | Often documented in model card | Often undisclosed or vaguely described |
| Fine-tuning | You control it | Provider may fine-tune without notice |
| Versioning | You pin a specific file hash | Provider may update behind the same endpoint |
| Hosting | Your infrastructure | Provider's infrastructure, shared, multi-tenant |

**Key insight:** When you call a model through an API, you are trusting the provider's entire pipeline — their training data, fine-tuning decisions, hosting security, and update practices. You cannot verify any of these independently.

---

## Task 5 — The Four Attack Layers

Every stop on the supply chain route is a potential interception point. A sophisticated attacker doesn't need to own the entire chain — just **one checkpoint**.

### Layer 1: The Model Layer

The most distinctive part of the AI attack surface. Three distinct attack levels:

| Level | How It Works | Does SafeTensors Fix It? |
|---|---|---|
| **Serialisation-level** | Code hidden inside the file format, executes when the file is loaded | ✅ Yes |
| **Architecture-level** | Malicious logic embedded in the model's layers, executes on every prediction | ❌ No |
| **Weights-level** | Learned values subtly altered to misbehave on specific inputs | ❌ No |

> **Critical insight:** Converting to SafeTensors eliminates serialisation-level attacks but leaves architecture-level and weights-level attacks completely untouched. SafeTensors is a single control, not a complete solution.

### Layer 2: The Dependency Layer

Covers the packages and libraries your ML project depends on. Attack methods include:
- **Dependency confusion** — uploading a public package with the same name as an internal one
- **Typosquatting** — registering `pytorch` instead of `torch`, hoping for typos
- **Malicious packages** — professional-looking packages that hide malware

### Layer 3: The Data Layer

Training data is the foundation of every ML model, and poisoning it is hard to detect.

> Researchers have shown that replacing as little as **0.1%** of a training dataset with crafted samples can introduce a reliable backdoor — without measurably affecting accuracy on clean data.

For denial-of-service objectives, the effective threshold drops as low as **0.001%**.

### Layer 4: The Infrastructure Layer

Covers the systems that host, distribute, and manage AI artefacts:
- Gaining access to model or package repositories
- Injecting malicious steps into CI/CD build pipelines
- Stealing maintainer credentials to push malicious updates under trusted identities

### The Compounding Effect

These layers don't exist in isolation. A single download can trigger compromises at every layer simultaneously:

1. Compromise a Hugging Face account **(infrastructure layer)**
2. Upload a backdoored model with malicious pickle code **(model layer)**
3. Include a `requirements.txt` referencing typosquatted packages **(dependency layer)**
4. Distribute a poisoned dataset alongside the model **(data layer)**

---

## Task 6 — Real-World Incidents

Five significant incidents from 2022 to 2025, each exploiting a different part of the supply chain:

### Incident 1: PyTorch Dependency Confusion (December 2022)

**Layer:** Dependency

An attacker published a malicious package named `torchtriton` on PyPI, exploiting the fact that PyTorch's nightly builds depended on an internal package with the same name. `pip`'s version resolution installed the attacker's public version instead.

The package:
- Stole SSH keys, Git configuration, `/etc/passwd`, and up to 1,000 home directory files
- Exfiltrated everything via encrypted DNS queries
- Persisted for **five days** before discovery

### Incident 2: Hugging Face Hub Vulnerabilities (2023–2024)

**Layer:** Infrastructure + Model

- **November 2023:** Lasso Security found over 1,500 exposed API tokens, 655 with write permissions — allowing attackers to push updates to legitimate repositories
- **2024:** JFrog identified ~100 malicious pickle-based models with professional descriptions and real ML functionality that silently executed attacker code at load time

### Incident 3: Ultralytics/YOLO Build Pipeline Compromise (December 2024)

**Layer:** Infrastructure

Attackers injected malicious code into the **GitHub Actions build workflow** for Ultralytics (makers of the popular YOLO object detection library). A cryptominer was embedded into published PyPI packages — developers who ran a standard `pip install` received the malicious version.

The source code was clean. The **build process** was not.

### Incident 4: @solana/web3.js Compromise (December 2024)

**Layer:** Infrastructure

Attackers stole a maintainer's npm credentials and published two backdoored versions of a widely-used package. The package exfiltrated cryptocurrency private keys from wallet holders.

- Lasted only **hours** before detection
- Estimated losses: **$130,000–$184,000**
- Victims did nothing wrong — they installed from a trusted, legitimate source

### Incident 5: NullifAI Scanner Evasion on Hugging Face (February 2025)

**Layer:** Model

ReversingLabs researchers discovered that deliberately **corrupted pickle files** could bypass Hugging Face's automated security scanners entirely while still executing malicious code when loaded.

The technique exploited edge cases in how scanners parse malformed pickle structures — meaning a model could **pass all platform-level checks and still be dangerous**.

### Summary Table

| Incident | Year | Layer | Key Lesson |
|---|---|---|---|
| PyTorch `torchtriton` | 2022 | Dependency | Same package name = pip picks the attacker's version |
| Hugging Face tokens | 2023-24 | Infrastructure + Model | Write-access tokens = ability to push to legitimate repos |
| Ultralytics/YOLO | 2024 | Infrastructure | Build pipelines are part of the supply chain |
| Solana web3.js | 2024 | Infrastructure | Stolen credentials = trusted identity, no flags raised |
| NullifAI | 2025 | Model | Automated scanners can be bypassed; malformed files still execute |

---

## Task 7 — Practical: Evaluating a Model (Trojan Model Gift Package)

### The Scenario

Your team lead found a sentiment analysis model (`trustworthy-ai-models/bert-sentiment-classifier`) and wants to deploy it to production. Your job: evaluate whether it is safe.

### Red Flags on the Suspicious Model

| Signal | Suspicious Model | Why It's a Red Flag |
|---|---|---|
| Organisation | `trustworthy-ai-models` — **Not verified**, joined January 2025 | Brand new account, no track record |
| Downloads (last month) | **1** | Near zero adoption — nobody trusts it |
| File format | `pytorch_model.bin` — **Pickle Format** tag shown | Can execute arbitrary code on load |
| Security notice | ⚠️ "This model uses pickle serialisation which can execute arbitrary code" | Platform itself is warning you |

### The Verified Model Comparison (google-bert/bert-base-uncased)

| Signal | Verified Model | Why It's Trustworthy |
|---|---|---|
| Organisation | `google-bert` — **Verified organisation** | Established, joined 2018 |
| Downloads (last month) | **44,802,476** | Millions of users worldwide |
| File format | `model.safetensors` — **SafeTensors** | No executable code, safe to load |
| Security status | ✅ "SafeTensors format, verified organisation with 6+ year track record" | Clean bill of health |

### Verdict

**Do not approve the suspicious model for production.** Every single signal is wrong — unverified brand-new organisation, 1 download, pickle format with an explicit security warning. A legitimate model earning trust looks like google-bert: years of track record, millions of users, safe file format.

---

## Task 8 — Conclusion: Key Takeaways

| Takeaway | What It Means |
|---|---|
| **Model files are code in disguise** | Pickle-based models execute at load time; format choice determines your exposure |
| **The four attack layers are distinct** | Model, dependency, data, and infrastructure attacks require different defences |
| **Transitive dependencies expand your attack surface silently** | You are responsible for every package `pip` installs, not just the ones you listed |
| **Automated scanners are not a complete defence** | The NullifAI case shows malformed files can pass platform checks while remaining executable |
| **Infrastructure compromise beats content filtering** | Stolen credentials push malicious code with full legitimacy; no tooling flags it at download time |
| **SafeTensors is one control, not a complete solution** | It eliminates serialisation-level attacks only; architecture-level and weights-level attacks remain |

---

## Quick Reference: Attack Layers vs Defences

| Layer | Attack | Defence |
|---|---|---|
| Model | Pickle code execution, backdoored weights | Use SafeTensors, inspect architecture and weights |
| Dependency | Typosquatting, dependency confusion | Pin versions, use lock files, audit transitive deps |
| Data | Training data poisoning (0.1% threshold) | Audit data sources, use curated datasets |
| Infrastructure | Stolen credentials, CI/CD injection | MFA on all accounts, signed artifacts, audit build pipelines |

---

## Answers Summary

| Question | Answer |
|---|---|
| What do model files contain that allows them to run code when loaded? | Serialised objects |
| What are the four key components of an AI supply chain (alphabetically)? | datasets, dependencies, frameworks, models |
| What is the dominant file format for running local LLMs such as LLaMA, Mistral, and Qwen? | gguf |
| At which layer do pickle-based attacks occur? | Model Layer |
| Which level of model attack is eliminated by converting to SafeTensors? | Serialisation-level |
| 0.1% of training data replaced with crafted samples — which attack layer? | Data Layer |
| What is the name of the unverified organisation that uploaded the suspicious model? | trustworthy-ai-models |
| How many downloads does the suspicious model have (last month)? | 1 |
| What file format does google-bert/bert-base-uncased use for its weights? | safetensors |
