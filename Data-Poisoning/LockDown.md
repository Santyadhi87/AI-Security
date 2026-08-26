# Lockdown — TryHackMe Write-Up

**Module:** AI Security > Data Poisoning  
**Room:** Lockdown   


---

## Overview

> *"An AI assistant has three open vulnerabilities. Find them. Fix them. Prove the system is secure."*

You play a security engineer at **Meridian Security Group**. The company deployed an internal AI assistant called **Bastion** to help employees with policy questions and operational issues. A routine audit flagged three security vulnerabilities in Bastion's configuration.

Your job: confirm each vulnerability exists, diagnose the root cause, and prescribe the **exact** security control that fixes it. Vague answers are rejected — Bastion only accepts precise remediation descriptions.

---

## Your Setup

You interact with Bastion directly via a chat window. You are logged in as a **security administrator**.

| Command | Purpose |
|---|---|
| `SHOW LOGS` | Inspect logging behaviour |
| `QUERY AS: [name]` | Test tenant isolation |
| `STATUS` | Check progress on flag fragments |

Each correct remediation unlocks a **flag fragment**. Three vulnerabilities → three fragments → one complete flag.

---

## Task 1 — The Challenge

### Understanding the Environment

Bastion is a **RAG-based AI assistant** connected to an internal document store. It serves data from at minimum two document categories:

- **Client Contracts** (e.g., Northwind Pharma — $4.2M/year agreement, 99.95% SLA, $50,000/hour breach penalty, contact: Rachel Dunn, renews September 2026)
- **Employee Data** (e.g., Employee #1847, Carla Diaz — on a Performance Improvement Plan since January 2026)

This immediately raised a red flag: **both sensitive document types were accessible from a single query**, with no apparent access boundaries enforced between them. A standard employee asking a policy question should not be retrieving HR PIP data or confidential contract terms in the same response.

---

## Vulnerability 1 — Missing Document-Level Access Control (Data Retrieval)

### What I Found

When querying Bastion, it returned data from **multiple document classification levels** in a single response — mixing Client Contract details and Employee PII together. There was no enforcement of who could see what; any query could retrieve any document regardless of sensitivity or the user's clearance level.

**Root Cause:** The RAG retrieval pipeline had no **metadata pre-filtering**. At retrieval time, the vector similarity search returned the top-k most semantically similar chunks regardless of their classification metadata (e.g., `confidential: true`, `department: HR`, `access_level: executive`). Confidential documents were not excluded before results were passed to the LLM.

### Why It Matters

Without document-level access control:
- A low-privilege employee querying "sick leave policy" could accidentally (or intentionally) receive results from confidential HR or legal documents
- An attacker with employee access could systematically extract sensitive business data through targeted queries
- This is a textbook example of **OWASP LLM08 — Excessive Agency** and **LLM06 — Sensitive Information Disclosure** at the retrieval layer

### The Fix I Applied

```
Enforce Document-Level Access Control (DLAC)
```

**What this means technically:** Before the vector search returns results to the LLM, each document chunk in the vector store carries metadata tags (classification level, owning department, permitted roles). The retrieval pipeline is modified to **pre-filter** on those tags — only chunks that match the requesting user's role and clearance are ever considered in the similarity search. Confidential documents are excluded from the candidate pool *before* semantic ranking occurs.

**Flag Fragment 1:** `THM{l0ck_`

---

## Vulnerability 2 — Insufficient Log Redaction (Logging)

### What I Found

Running `SHOW LOGS` revealed that Bastion's logging system recorded full document content, including document IDs and potentially sensitive text from retrieved chunks. Logs were not redacted — they stored the actual document data that had been retrieved and surfaced to users.

**Root Cause:** The logging pipeline captured the raw retrieved document text alongside user queries. Over time, logs become a **secondary exfiltration surface**: even if access controls are tightened on the live system, historical logs may contain sensitive data that is readable by anyone with log access (IT staff, SIEM operators, etc.).

### Why It Matters

- Logs are often shared more broadly than production data (e.g., piped to a centralised SIEM or handed to a third-party monitoring service)
- A compromised logging infrastructure exposes the same sensitive data as a compromised RAG system
- Compliance frameworks (GDPR, HIPAA, NZ Privacy Act 2020) require that PII and sensitive business data not persist in unprotected log stores
- This maps to **OWASP LLM10 — Unbounded Consumption** (resource and data leakage) and general data minimisation principles

### The Fix I Applied

```
Implement log redaction
```

**What this means technically:** The logging middleware is updated to strip or hash sensitive content before writing log entries. Specifically:
- Logs record **document IDs only** (a reference pointer), not document text
- Any PII fields (names, employee numbers, financial figures) are either redacted (`[REDACTED]`) or replaced with a one-way hash
- Query text that contains PII can be sanitised before logging

After applying this fix, Bastion confirmed: *"Logs now record doc IDs only."*

**Flag Fragment 2:** `d0wn_`

---

## Vulnerability 3 — Missing Tenant Isolation (User-Level Access)

### What I Found

Using `QUERY AS: [name]` to impersonate other users showed that Bastion served identical results regardless of which user identity was passed. There was no isolation between user sessions or organisational tenants — user A could retrieve the same data as user B with no boundary enforced.

**Root Cause:** The vector database layer had no **tenant isolation** mechanism. In a multi-user or multi-tenant RAG deployment, each user's queries should be scoped to their permitted namespace within the vector DB. Without this, every query searches the entire document corpus globally.

### Why It Matters

- In a real corporate deployment, different business units (HR, Legal, Finance, Engineering) should only access their own documents
- Without tenant isolation, lateral movement within the AI system is trivial: any authenticated user can probe for data belonging to any other team
- This directly enables **insider threat** scenarios and **privilege escalation through the RAG layer**
- Maps to **OWASP LLM02 — Insecure Output Handling** and **AML.T0024 (Exfiltration via ML Inference API)** in MITRE ATLAS

### The Fix I Applied

```
Enforce tenant isolation
```

**What this means technically:** Tenant isolation is implemented at the **vector DB layer**. Each document chunk is tagged with a `tenant_id` (or `namespace`) at ingestion time. At query time, the retrieval call is scoped to the requesting user's tenant namespace — the vector similarity search only operates within that namespace. Documents from other tenants are never included in the candidate pool, even if they are semantically very similar to the query.

After applying this fix, Bastion confirmed: *"Tenant isolation is enforced at the vector DB layer."*

**Flag Fragment 3:** `s3cur3d}`

---

## Complete Flag

Combining the three fragments in order:

```
THM{l0ck_d0wn_s3cur3d}
```

---

## Attack Surface Summary

| # | Vulnerability | Root Cause | Fix Applied | OWASP LLM Ref |
|---|---|---|---|---|
| 1 | Cross-classification data retrieval | No metadata pre-filtering at retrieval | Document-Level Access Control (DLAC) | LLM06 — Sensitive Information Disclosure |
| 2 | Sensitive data in logs | Log pipeline captured raw document text | Log redaction (doc IDs only) | LLM10 — Unbounded Consumption |
| 3 | No user/tenant boundaries | Vector DB searched global corpus for all users | Tenant isolation at vector DB layer | LLM02 — Insecure Output Handling |

---

## MITRE ATLAS Mapping

| Technique | ID | Relevance |
|---|---|---|
| Exfiltration via ML Inference API | AML.T0024 | Retrieving cross-tenant data through Bastion queries |
| Discover ML Artifacts | AML.T0007 | Probing Bastion's document store by querying as different users |
| ML Supply Chain Compromise | AML.T0010 | Poisoned/misconfigured retrieval pipeline enables data leakage |

---

## Key Lessons Learned

### 1. RAG systems inherit traditional access control problems — but the attack surface is wider
In a SQL database, you write `WHERE user_id = ?`. In a RAG system, there's no equivalent built-in. Access control must be **explicitly designed into the retrieval pipeline** via metadata filtering and namespace scoping. It doesn't happen automatically.

### 2. Logs are a second copy of your sensitive data
Every time Bastion retrieved a confidential contract and logged it, a copy of that data landed in the log store. Tightening RAG access controls is incomplete if logs still contain the raw retrieved content. **Data minimisation applies to logs too.**

### 3. Vector databases need tenant isolation, not just the application layer
It's tempting to enforce access control only at the application/API layer ("is this user allowed to call Bastion?"). But if the vector DB is queried globally, a compromised API call — or a misconfigured prompt — can still reach all tenants' data. **Isolation must be enforced as close to the data as possible.**

### 4. Precision matters in security remediation
Bastion rejected vague answers like "implement metadata filtering" (too generic) in favour of exact control names. This reflects real-world remediation: a JIRA ticket saying "fix the access issue" is not actionable. **Security engineers must name the specific control.**

### 5. Three-layer defence for RAG deployments
This room neatly demonstrates the three layers every RAG deployment should harden:

```
Query Layer        → User Authentication + Role Assignment
Retrieval Layer    → Document-Level Access Control + Metadata Pre-filtering
Storage/Log Layer  → Log Redaction + Tenant Namespace Isolation
```

All three must be addressed. Fixing one layer while leaving the others open just shifts the attack surface.

---

## Tools & Commands Used

| Tool / Command | Purpose |
|---|---|
| `SHOW LOGS` | Reveal logging behaviour and identify data leakage in logs |
| `QUERY AS: [name]` | Test whether tenant isolation is enforced |
| `STATUS` | Track flag fragment collection |
| Direct Bastion queries | Surface cross-classification data retrieval vulnerability |

---

## References

- [OWASP LLM Top 10](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [MITRE ATLAS](https://atlas.mitre.org/)
- [NIST AI Risk Management Framework](https://www.nist.gov/artificial-intelligence/ai-risk-management-framework)
- [TryHackMe — AI Security Path](https://tryhackme.com/paths)
