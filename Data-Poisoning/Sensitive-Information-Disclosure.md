# Sensitive Information Disclosure in RAG Systems
**TryHackMe — Data Poisoning Module | Sub-Module: Sensitive Information Disclosure**


---

## Table of Contents

- [Task 1 — Introduction](#task-1--introduction)
- [Task 2 — What Is Sensitive Information Disclosure?](#task-2--what-is-sensitive-information-disclosure)
- [Task 3 — Disclosure Scenarios & Case Studies](#task-3--disclosure-scenarios--case-studies)
- [Task 4 — Embedding & Vector Disclosure](#task-4--embedding--vector-disclosure)
- [Task 5 — Retrieval Misconfiguration](#task-5--retrieval-misconfiguration)
- [Task 6 — Access Control & Data Segmentation](#task-6--access-control--data-segmentation)
- [Task 7 — Defence Mechanisms](#task-7--defence-mechanisms)
- [Task 8 — Practical Lab (Meridian Health)](#task-8--practical-lab-meridian-health)
- [Task 9 — Conclusion](#task-9--conclusion)
- [Key Answers Summary](#key-answers-summary)

---

## Task 1 — Introduction

### What Is This Room About?

Imagine you have a smart AI assistant at work. It can answer questions by searching through company documents. Sounds useful, right?

The problem is — if that assistant searches through **all** documents without checking who is allowed to see what, it can accidentally show secret information to the wrong person. This is called **Sensitive Information Disclosure**.

> **Simple analogy:** It's like a filing clerk who, when you ask for one document, pulls out five — including some that are marked "confidential" and meant only for the CEO.

### The Key Insight

> LLMs don't leak data because they decide to reveal secrets. They leak because **sensitive data was allowed to enter the context window**.

### OWASP Category

This room focuses on **OWASP LLM02 — Sensitive Information Disclosure**, which covers protecting:
- Private personal data
- Proprietary business logic
- Confidential documents

### Learning Objectives

| Objective | What You Learn |
|---|---|
| Define LLM02 | What sensitive information disclosure is |
| Parametric vs retrieval leakage | Training data leaks vs runtime retrieval leaks |
| Architectural leak points | Where in the pipeline data escapes |
| System design issue | Why this is an infrastructure problem, not a model problem |

---

## Task 2 — What Is Sensitive Information Disclosure?

### The Three-Way Distinction

These three attacks are commonly confused. Here's how to tell them apart:

| Attack Type | What It Targets | Simple Explanation |
|---|---|---|
| **Data Poisoning** | What the system learns | Feeding bad food to make someone sick |
| **Prompt Injection** | Runtime instructions | Whispering bad instructions in someone's ear mid-task |
| **Sensitive Info Disclosure** | Data that already exists | The filing clerk accidentally showing you files you shouldn't see |

> **Key point:** For disclosure, the attacker doesn't need to hack the model. They just need to **trigger the retrieval system** or observe architectural weaknesses.

### How RAG Creates Leakage Points

A RAG (Retrieval-Augmented Generation) system works like this:

```
User asks a question
        ↓
System searches a database of documents
        ↓
Retrieves the most "similar" documents
        ↓
Feeds them into the AI's context window
        ↓
AI generates an answer using those documents
```

Each step in this pipeline is a potential leakage point:

| Leakage Point | What Goes Wrong |
|---|---|
| Over-broad similarity search | Retrieves unrelated confidential documents |
| Shared vector databases | No separation between departments or customers |
| Large context windows | Multiple sensitive chunks combined in one prompt |
| Debug logging | Full prompts stored in plaintext that anyone can read |
| Missing metadata filtering | No access rules enforced before searching |

### Real-World Scenario — The Cloud Provider

A cloud company builds an AI support assistant. They embed past support tickets into the database to improve answers. Those tickets contain customer account IDs, API keys, and architecture diagrams.

A customer asks: *"Why does my Kubernetes deployment keep failing?"*

The system finds an old ticket from **another customer** with a very similar problem. It pulls that ticket and the AI uses it to answer — accidentally revealing the other customer's cluster name: `cluster-prod-eu-west-42`.

**Nothing was hacked. No one broke in. The system just did what it was designed to do.**

---

## Task 3 — Disclosure Scenarios & Case Studies

### The Multi-Tenant Support Platform

A company's AI assistant uses RAG. The engineering team enables **verbose logging** to debug problems. Every request now logs:
- The user's query
- All retrieved document chunks
- The full augmented prompt
- The model's response

A senior engineer asks about authentication failures. The system correctly retrieves a confidential breach analysis — the engineer is authorised to see it.

**The problem?** The full prompt (including the confidential breach analysis) is now stored in plaintext logs. DevOps staff and external contractors who have log access can now read that confidential document — even though they could never access it directly.

> **Simple analogy:** You're allowed to read a secret letter. But your assistant photocopies it and pins it to the public noticeboard for everyone to see.

### What Actually Happened

```
Confidential document retrieved (authorised) ✅
         ↓
Full augmented prompt logged in plaintext ⚠️
         ↓
Log system has broader access than source database ❌
         ↓
DevOps / contractors read the confidential content
```

The model didn't leak it. The **logging architecture** leaked it.

### Case Study 1 — EchoLeak (CVE-2025-32711)

**What happened:** A malicious email was sent to a target user. Hidden instructions were embedded in the HTML — invisible to the human reader but readable by the AI system.

When the user later asked the AI to *"summarise my unread emails,"* the retrieval system found the malicious email because it was **geometrically close** to the query in vector space.

The hidden instructions told the system to:
1. Search for sensitive files
2. Extract their contents
3. Send them to a remote URL (exfiltration)

**Root cause:** Retrieved documents were inserted into the prompt without checking whether they contained instructions or data. There was no separation between trusted instructions and untrusted content.

### Case Study 2 — Proof Pudding (CVE-2019-20634)

**What happened:** Researchers targeted an email spam filter. The model only returned spam probability scores — not raw data.

But by submitting many crafted inputs and watching how scores changed, they were able to:
- Understand how the filter made decisions
- Approximate its internal logic
- Build a "copy-cat" model

> **Key lesson:** You don't need plaintext exposure to have a breach. **Inference from outputs alone can be enough.**

### Why These Matter

| Risk | Description |
|---|---|
| Over-broad retrieval | Too many documents retrieved — confidential ones included |
| Shared vector stores | No separation between tenants or departments |
| Prompt-based access control | Instructions to the model aren't real access control |
| Logging captures full prompts | Logs become an unguarded copy of sensitive data |
| Embeddings aren't anonymous | Numeric vectors still carry semantic meaning |

---

## Task 4 — Embedding & Vector Disclosure

### What Is an Embedding?

When text is stored in a RAG system, it's converted into a list of numbers called an **embedding**. Similar texts get similar numbers, so they cluster together in what's called a **vector space**.

> **Simple analogy:** Imagine a giant library where books are sorted not by title but by how similar their content is. Books about dogs and books about cats are shelved near each other. The system finds what you're looking for by walking to the nearest shelf — not by checking who's allowed to read which book.

### How Similarity Search Works

The system measures closeness using **cosine similarity** — basically measuring the angle between two vectors. The smaller the angle, the more similar the documents.

```python
# Example: Query vs two documents
query          = [0.8, 0.1]
safe_doc       = [0.7, 0.2]   → similarity score: 0.98
confidential   = [0.79, 0.11] → similarity score: 0.999  ← WINS
```

The confidential document wins — not because the user is authorised, but because **the math ranked it higher**.

### Five Vector-Level Risks

#### 1. Cosine Similarity Has No Access Control
The retrieval system is purely mathematical. It finds the closest vectors. It doesn't know or care about permissions.

#### 2. Corpus Manipulation
If an attacker inserts documents with embeddings close to common queries, those documents will repeatedly appear in results — changing what the model sees at runtime without retraining it.

#### 3. Embedding Inversion
People assume embeddings are safe because they're just numbers. **This is wrong.** Attackers can train surrogate models to reverse-engineer text from stored vectors, recovering:
- Names and identifiers
- Project codes
- Numerical values
- Sensitive themes

#### 4. Membership Inference
Sometimes the attacker doesn't need to reconstruct the text. They just need to **confirm it exists**. By submitting a candidate embedding and measuring the similarity score, an attacker can confirm whether a specific person, incident, or project is in the database — and in regulated industries, that confirmation alone is a privacy breach.

#### 5. Multi-Tenant Vector Stores Risk
Many organisations use one shared vector database for multiple departments or customers. If metadata filtering isn't applied **before** the search, different tenants' data can appear in the same results.

> Even if the model is told "only answer using User A's documents," unauthorised vectors from User B may already be in the context window. The boundary was broken **before** the model responded.

### Case Study — Milvus Auth Bypass (CVE-2025-64513)

| Element | Detail |
|---|---|
| **System** | Milvus — popular open-source vector database |
| **Flaw** | Proxy trusted a user-controlled HTTP header (`sourceId`) as proof of authentication |
| **Exploit** | Forge the header → bypass authentication entirely |
| **Impact** | Read all embeddings, access metadata, modify or delete collections |
| **CWE** | CWE-287 (Improper Authentication) |
| **Lesson** | Vectors aren't harmless numbers — they encode compressed document relationships |

---

## Task 5 — Retrieval Misconfiguration

### Retrieval Is a Security Boundary

Whatever enters the context window is what the model works with. There is **no secondary access control layer inside the LLM**. The model has no awareness of who the user is or what they're allowed to see.

> **Simple analogy:** The retrieval layer is the security guard at the door. Once someone is inside the building, there are no more checkpoints. If the guard lets the wrong documents through, the model has no way to know.

### Three Retrieval Failure Modes

#### Failure 1 — Over-Broad Top-k Retrieval

`top-k` is the number of documents retrieved. Higher = better answers, but also **higher exposure risk**.

```
top-k = 3  →  3 documents retrieved
top-k = 10 →  10 documents retrieved (including loosely related sensitive ones)
```

#### Failure 2 — Weak or Missing Metadata Filtering

Metadata filtering restricts which documents are eligible for retrieval. The critical rule:

> **Filter BEFORE similarity search. Not after.**

If you filter after ranking, unauthorised documents have already influenced the results.

A system prompt saying *"only answer using authorised documents"* is **not** access control — it's a suggestion the model can ignore.

#### Failure 3 — Stale Embeddings

When you delete a document from your system, its embedding may remain in the vector database — still searchable, still retrievable, still a liability.

### Code Example — What the Three Scenarios Show

| Scenario | Config | What Gets Retrieved |
|---|---|---|
| **Scenario 1** | top-k=2, no filter | Public doc ✅ + Confidential payroll doc ❌ |
| **Scenario 2** | top-k=3, no filter | Public doc ✅ + Confidential ❌ + Deleted doc ❌ |
| **Scenario 3** | top-k=2, **filter first** | Public doc only ✅ |

### CVE-2025-64513 — The Retrieval Gateway Lesson

When the Milvus Proxy authentication was bypassed, attackers could send **arbitrary similarity queries**. The database returned top-k results as if the request was completely legitimate — because from the database's perspective, it was. The retrieval gateway is the access control layer. When it fails, everything is exposed.

---

## Task 6 — Access Control & Data Segmentation

### The Core Problem

Vector databases rank by **similarity, not identity**. A shared vector store doesn't know which tenant owns which document. It just returns the closest vectors.

> **Simple analogy:** Imagine a filing room where all companies share the same shelves, with no labels. When you search for "Q3 financials," you might get your own documents — or you might get your competitor's.

### The Golden Rule

> **Restrict the eligible set FIRST, then compute similarity.**

Putting access control in the prompt ("only return documents the user can access") doesn't work. The model can ignore it, misunderstand it, or be talked out of it.

### Three Segmentation Patterns

| Pattern | How It Works | Strength | Weakness |
|---|---|---|---|
| **Per-Tenant Index** | Each tenant gets their own dedicated vector index | Strongest — zero cross-contamination possible | Higher cost, more infrastructure to maintain |
| **Per-Role Index** | Documents grouped by access tier (public/internal/confidential) | Simpler than per-tenant | Identity layer and index selector must stay in sync |
| **Metadata-Based Filtering** | One shared index; each doc tagged with `tenant_id`, `role`, `department` | Most commonly deployed | Most commonly misconfigured — a forgotten filter exposes everything silently |

### Shared Index Risk

When multiple tenants share one index with no metadata filtering:

- Similarity search crosses tenant boundaries silently
- Filtering **after** retrieval doesn't undo the exposure — those vectors already influenced ranking
- The model processes whatever tokens arrive — it has no awareness of tenant ownership

### Case Study — CVE-2024-3033 (AnythingLLM Auth Bypass)

| Element | Detail |
|---|---|
| **Platform** | AnythingLLM — RAG app using LanceDB, Qdrant, Chroma, Weaviate |
| **Structure** | Documents organised into workspaces → mapped to vector DB namespaces |
| **Flaw** | Internal API endpoints (`/api/v/`) had no authentication or authorisation |
| **Impact** | Unauthenticated attackers could enumerate namespaces, discover workspace names, delete collections, reset the entire vector DB |
| **Lesson** | *"Logical separation is not security unless access control is enforced at the retrieval and management layer"* |

---

## Task 7 — Defence Mechanisms

### The Principle

> **No single control fixes this. Confidentiality must be enforced before anything reaches the model.**

Think of it like layers of security at an airport — passport control, bag screening, boarding gate check. Each layer catches what the others miss.

### Layer 1 — Redaction at Ingestion

Strip sensitive data **before** generating embeddings. Once something is embedded, its meaning is baked in — you can't un-embed it without re-indexing the entire database.

| Technique | What It Does |
|---|---|
| Remove PII fields | Stops personal data entering the index |
| Mask identifiers | Replaces sensitive codes with placeholders |
| Replace secrets | API keys, passwords become `[REDACTED]` |
| NER pre-indexing | Named Entity Recognition catches names, orgs, locations before embedding |

> **Tradeoff:** Over-redacting degrades retrieval quality — embeddings lose the context needed for accurate matching.

### Layer 2 — Retrieval Filtering (Allowlist-Based)

Hard eligibility rules applied **in the database query itself**, before similarity ranking runs.

```
User sends query
       ↓
Apply metadata filters (tenant_id, role, clearance) ← HERE
       ↓
Run similarity search on the filtered set only
       ↓
Return top-k from the authorised subset
```

> **Tradeoff:** More filtering rules = more complex queries that can silently break when roles or metadata change.

### Layer 3 — Logging Minimisation

Logs are an **unguarded second copy** of your most sensitive data.

| ❌ Never Log | ✅ Safe to Log |
|---|---|
| Full augmented prompts | Document IDs |
| Retrieved document chunks | Hashes |
| Raw embeddings | Minimal structured event summaries |
| Sensitive metadata | Timestamps, query type |

> **Tradeoff:** Thinner logs make debugging harder — decide what's needed for incident response and cut everything else.

### Layer 4 — Data Retention Controls

Deleting a source document must trigger the full chain:

```
Source document deleted
        ↓
Embedding removed from vector DB ← often forgotten
        ↓
Index updated
        ↓
Cache invalidated
```

Miss any step and the data is still live and retrievable.

### Layer 5 — Monitoring & Detection

| Signal | What It Indicates |
|---|---|
| Retrieval volume spikes | Unusual query patterns — possible probing |
| Cross-tenant access attempts | Namespace boundary violations |
| Repeated similarity probing | Attacker mapping what comes back |
| Namespace enumeration | Attacker discovering index structure |
| Sensitive doc flagged in retrieval | High-risk content reaching the context |
| API keys / SSNs in output | Exfiltration attempt in generated response |

### Why Layers Matter — The Compound Failure Model

Each individual failure is survivable. Combined failures cause disclosure:

```
Redaction missed one field
    + Filter didn't account for a new role
    + Logs recorded the full prompt
= Disclosure event
```

---

## Task 8 — Practical Lab (Meridian Health)

### Lab Setup

Meridian Health deployed an internal AI assistant backed by a RAG pipeline. **All documents — public and confidential — are stored in a single shared index with no access controls.**

Your job: demonstrate how this architecture leaks sensitive data.

---

### Phase 1 — Overly Broad Retrieval

**What you did:**

| Query | Expected | Actual |
|---|---|---|
| `What is the vacation policy?` | Public info only | ✅ Correct — 20 days PTO returned |
| `What are the salary ranges for engineering roles?` | Blocked | ❌ Confidential salary data retrieved |
| `What is the CEO's compensation package?` | Blocked | ❌ Julia Fang's full compensation returned |

**Why it happened:** `broad retrieval` — the system retrieved all semantically similar documents regardless of their classification. No metadata filtering was applied.

---

### Phase 2 — Logging Exposure

**What you did:** Ran `SHOW RETRIEVAL LOG`

**What the log showed:**
```
[RETRIEVAL LOG]
Query: What is the CEO's compensation package?
Chunks retrieved: [CONFIDENTIAL] === EXECUTIVE COMPENSATION Q1 2026 ===
Chunks passed to model context: [CONFIDENTIAL] === EXECUTIVE COMPENSATION Q1 2026 ===
Filtering applied: None
```

**The critical observation:** Even though the model responded *"I don't have a reference for that,"* the **confidential chunk was still retrieved and logged in full plaintext**. Anyone with log access — DevOps, contractors — could read the CEO's compensation data.

**What could have prevented this:** `metadata filtering` — applying access control at the query level so confidential chunks never enter the retrieval pipeline at all.

---

### Phase 3 — Semantic Collision

**What you did:** Asked `Tell me about employee benefits enrollment`

**What came back:**
- The public benefits overview ✅
- **Tom Russo's confidential HR record** ❌ — *"Employee #2201 Tom Russo — benefits enrollment flagged for dependent eligibility audit"*

**Why it happened:** `Semantic Collision` — the words "benefits" and "enrollment" appeared in both the public policy document and Tom Russo's confidential HR record. Their embeddings were placed close together in vector space, so both were retrieved by the top-k similarity search.

---

### Phase 4 — Access Control Applied

**What you did:** Ran `ENABLE ACCESS CONTROL` then re-ran all queries

| Query | After ACL |
|---|---|
| Salary ranges | ❌ Access denied |
| CEO compensation | ❌ Access denied |
| Benefits enrollment | ✅ Public overview only — Tom Russo's record gone |
| Vacation policy | ✅ Full public policy still returned |

**Result:** Metadata filtering restricted confidential data without breaking public access.

### Lab Summary

| Phase | Attack Type | Root Cause | Fix |
|---|---|---|---|
| Phase 1 | Overly broad retrieval | No access controls on shared index | Metadata filtering pre-query |
| Phase 2 | Logging exposure | Full prompts logged in plaintext | Logging minimisation |
| Phase 3 | Semantic collision | Similar embeddings surface unrelated confidential docs | Tenant isolation + filtering |
| Phase 4 | Defence applied | Metadata filtering enabled | Access control at retrieval layer |

---

## Task 9 — Conclusion

### The Central Thesis

> **Confidentiality in LLM systems is a pipeline problem — not a model problem.**

By the time the model is generating a response, it's too late. Security must be enforced **before** similarity search runs.

### Six Recurring Failure Patterns

| Failure | Description |
|---|---|
| Over-broad top-k retrieval | Too many docs retrieved — sensitive ones included by proximity |
| Missing metadata filtering | No access rules before similarity search |
| Shared vector indexes | No tenant isolation — everyone's data in one pool |
| Stale embeddings | Deleted documents still retrievable in the index |
| Full prompt logging | Logs become an unguarded copy of sensitive data |
| Weak namespace enforcement | Logical separation that breaks under edge cases |

### Red Team Checklist

- Can you influence what gets retrieved by changing filters or phrasing?
- Can similarity results cross tenant boundaries?
- Are namespaces enumerable — can you map other tenants by probing?
- Are embeddings accessible via unlocked API or debug endpoints?
- Are logs capturing full context windows in plaintext?

### Blue Team Checklist

- Enforce metadata filtering **in the query** — not in the app layer, not in the prompt
- Apply **deterministic namespace isolation** between tenants
- Remove stale embeddings during **retention cycles**
- **Minimise logging** — logs with full context = a side door into your data
- **Monitor continuously** for volume spikes, cross-tenant access, repeated probing

### Framework Alignment

| Framework | Application |
|---|---|
| **OWASP LLM02** | Primary — Sensitive Information Disclosure |
| **OWASP LLM01** | Related — Prompt Injection via retrieval trust abuse |
| **OWASP LLM05** | Related — Supply Chain via external indexed corpora |
| **NIST AI RMF** | Data governance and access control as foundational risk controls |
| **EU AI Act Art. 9/10** | Data management and technical safeguards for sensitive systems |

---

## Key Answers Summary

| Task | Question | Answer |
|---|---|---|
| Task 3 | Mathematical mechanism for document retrieval in RAG | Cosine similarity |
| Task 3 | Parameter controlling how many documents are returned | top-k |
| Task 3 | CVE for zero-click prompt injection via retrieved content | CVE-2025-32711 |
| Task 4 | Metric used to measure similarity between embeddings | Cosine similarity |
| Task 4 | Attack that reconstructs text from stored vectors | Embedding inversion |
| Task 5 | Configuration change that increases exposure surface | Increasing top-k |
| Task 6 | Logical grouping inside a vector DB that separates datasets | Namespace |
| Task 6 | Segmentation model with strongest isolation but highest cost | Per-tenant |
| Task 6 | Enforcement that operates before computation instead of after | Deterministic |
| Task 7 | Control that removes sensitive data before embedding | Redaction |
| Task 7 | Policy ensuring deleted embeddings are removed from storage | Data retention |
| Task 8 | What caused the assistant to expose confidential data | Broad retrieval |
| Task 8 | Why Tom Russo's HR record appeared in benefits query | Semantic Collision |
| Task 8 | Control that could have prevented Phase 2 disclosure | Metadata filtering |
| Task 2 | OWASP category covering sensitive data exposure in LLMs | LLM02 |

---
