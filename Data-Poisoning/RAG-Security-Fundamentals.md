# RAG Security Fundamentals

**Module:** Data Poisoning  

---

## Overview

RAG (Retrieval-Augmented Generation) is a system where an AI looks up documents from a knowledge base before generating a response. Unlike a standard LLM that relies purely on training data, a RAG system retrieves external content at the time of answering — making the quality and safety of that content a direct security concern.

---

## Task 2 — Core Components of a RAG System

### What it is
A RAG system has four key components: the **Embedding Model**, **Vector Store**, **Retriever**, and **LLM**. Together they convert a user's question into a vector, search a database for similar documents, and feed those documents to an AI to generate an answer.

### Why it matters
Understanding the components tells you *where* an attacker can intervene. Each component is a potential security boundary — if any one of them is compromised, the final output is compromised too.

### How it works
1. User submits a query
2. Embedding model converts the query into a vector (numerical representation)
3. Vector store searches for documents with similar vectors
4. Retriever selects the top matching documents
5. Documents are injected into the LLM's context window
6. LLM generates a response — **without verifying if the retrieved content is safe or correct**

### Real-world analogy
Think of it like an **open-book exam**. The student (LLM) is allowed to look things up (retrieval). If someone secretly replaces pages in the reference book with wrong answers, the student will confidently write those wrong answers — and you'd never know the book was tampered with.

### Answer Table

| Question | Answer |
|---|---|
| What numerical representation is used to capture the meaning of text in RAG systems? | Vectors |
| Which component selects the documents for the LLM? | Retriever |

---

## Task 3 — RAG Attack Surfaces

### What it is
RAG systems have four main stages where security failures can occur: **document ingestion**, **embedding generation**, **similarity-based retrieval**, and **context injection**. Each stage adds a new layer of exposure.

### Why it matters
Each stage increases the system's attack surface. A failure at ingestion poisons everything downstream — retrieval and generation will both be affected without any further attacker action.

### How it works

| Stage | What happens | Security risk |
|---|---|---|
| Document ingestion | External documents enter the knowledge base | Malicious or untrusted docs treated as valid |
| Embedding generation | Text is converted to vectors | Metadata like authorship and approval status is stripped — all docs look equal |
| Similarity-based retrieval | Documents are ranked by semantic relevance | Attacker only needs content to "sound relevant" — not to be correct or safe |
| Context injection | Retrieved docs are inserted into the LLM prompt | Model cannot distinguish instructions from data |

### Real-world analogy
Imagine a **company wiki** that automatically pulls in content from anyone who has access. A rogue employee uploads a fake HR policy. The search system surfaces it whenever someone asks about leave entitlements — because it *sounds* relevant. The AI presents it as official policy. No one checks who wrote it.

### Answer Table

| Question | Answer |
|---|---|
| Which RAG stage introduces the largest indirect attack surface? | Document ingestion |
| What component is lost during embedding generation that affects security? | Context |

---

## Task 4 — Retrieval Abuse and Context Manipulation

### What it is
Retrieval abuse is when malicious or misleading documents influence a model's output through the retrieval process — **without the attacker ever touching the prompt**. It comes in two forms: passive poisoning and active manipulation.

### Why it matters
This is one of the hardest RAG attacks to detect because the system behaves exactly as designed. Logs show "relevant documents retrieved" — everything looks normal, but the output is wrong or harmful.

### How it works

| Type | Description |
|---|---|
| **Passive poisoning** | Malicious content is uploaded once and left in the knowledge base. Normal user queries eventually retrieve it automatically |
| **Active manipulation** | Content is deliberately crafted to rank highly for specific sensitive queries — like SEO, but for attacks |

Once a poisoned document enters the context window, the model treats it as authoritative. The model cannot see retrieval rankings, verify document intent, or distinguish instructions from data — this is a **design limitation**, not a misconfiguration.

### Real-world analogy
**Passive:** A disgruntled contractor uploads a fake troubleshooting guide to a company's internal knowledge base before leaving. Six months later, support staff using an AI assistant get the wrong fix every time they ask about that product area.

**Active:** An attacker optimises a fake refund policy document to use the exact phrasing customers use when asking about returns. The AI always surfaces it first and tells customers they have no refund rights.

### Answer Table

| Question | Answer |
|---|---|
| What retrieval abuse technique involves crafting malicious content so it ranks highly for sensitive queries? | Active manipulation |
| What does retrieval select documents based on? | Semantic relevance |

---

## Task 5 — Real-World Case Studies

### What it is
Three real-world incidents where RAG-style retrieval failures caused harm — ranging from deliberate abuse to governance failures with no attacker involved at all.

### Why it matters
These cases prove that RAG failures don't require sophisticated attackers. Poor ingestion controls, missing validation, and no document lifecycle management are enough to cause serious operational, compliance, and reputational damage.

### How it works — Case by Case

**Case 1: Microsoft Copilot — Email-Based Retrieval Abuse (2026)**
- Copilot ingested enterprise emails as valid knowledge sources
- Emails containing embedded instructions were retrieved during normal queries
- The model could not distinguish legitimate information from embedded manipulation
- **Impact:** Sensitive enterprise data exposed; organisations restricted Copilot access; Microsoft issued mitigations

**Case 2: ChatGPT Plugins — Untrusted External Content (2023)**
- Plugins allowed the model to retrieve live data from external web pages and APIs
- Retrieved content contained instruction-like text that the model followed
- No validation of retrieved content intent; no separation between data and instructions
- **Impact:** Unsafe outputs generated; plugin features temporarily disabled; security model redesigned
- This is a textbook example of **indirect prompt injection via retrieval**

**Case 3: Web-Connected AI Assistants — Stale Retrieval**
- AI assistants retrieved outdated documents that had already been updated at the source
- The retrieval pipeline prioritised semantic relevance over freshness
- No document lifecycle management — old docs stayed indexed indefinitely
- **Impact:** Users followed incorrect guidance; compliance and trust risks increased

### Real-world analogy
Case 3 is like a **library that never removes old editions**. A customer asks the AI librarian for the current tax rate. The AI finds a 2019 document that mentions tax rates — it's semantically relevant — and confidently tells the customer the wrong figure. No hacker involved. Just bad housekeeping.

### Answer Table

| Question | Answer |
|---|---|
| In the Web-Connected AI Assistants cases, failures were caused by governance gaps in what specific part of the system? | Retrieval pipeline |

---

## Task 6 — Detection, Guardrails, and Defence-in-Depth

### What it is
Detecting RAG poisoning is hard because poisoned outputs look normal — well-written, logically structured, and authoritative. Defence requires **multiple overlapping layers**: guardrails on retrieved content, strong ingestion validation, and ongoing behavioural monitoring.

### Why it matters
No single control fully protects a RAG system. If you rely on only one mechanism (e.g., just guardrails), a sophisticated attacker will find a way around it. Defence-in-depth means that even if one layer fails, others catch the problem.

### How it works

| Defence layer | What it does | Limitation |
|---|---|---|
| **Ingestion validation** | Reviews sources, enforces approval workflows, tracks ownership | Must happen before data enters — reactive controls are too late |
| **Guardrails on retrieval** | Limits how retrieved text is inserted; flags instruction-like patterns | Attackers can rephrase content to bypass heuristics |
| **Behavioural monitoring** | Watches for unusual retrieval patterns, repeated doc retrieval, response tone changes | Requires time to establish a baseline; subtle drift may be missed early |

**Output drift** — a slow, gradual shift in how the system responds over time — is a key warning sign of poisoning. It's the opposite of a sudden failure; it's the system slowly being steered in the wrong direction.

### Real-world analogy
Output drift is like a **slowly leaking tap**. Day to day you don't notice anything wrong. A month later the floor is soaked. By the time you notice, the damage has been building for weeks. That's what poisoning looks like in a RAG system — not a loud alarm, but a quiet, steady drift toward wrong answers.

### Answer Table

| Question | Answer |
|---|---|
| What type of monitoring is a useful way to detect RAG poisoning? | Behavioural monitoring |
| What does output drift reflect instead of a sudden failure? | Gradual influence |

---

## Key Lessons

- A RAG system never verifies whether retrieved content is correct, safe, or trusted — this is by design, not misconfiguration
- The **retriever** is the highest-risk component because retrieval is automatic, invisible to the user, and the model trusts all retrieved content equally
- **Inference-time data poisoning** means attackers don't need to retrain the model — they just need to get a bad document into the knowledge base
- RAG failures don't require attackers — governance gaps alone (stale docs, missing lifecycle management) cause real harm
- **Defence-in-depth** is the only effective strategy: validate at ingestion, apply guardrails at retrieval, monitor behaviour over time
- **Output drift** is the fingerprint of long-term RAG poisoning — watch for gradual changes, not just sudden failures

---

## Techniques & Concepts Summary

| Concept | Description |
|---|---|
| Inference-time data poisoning | Poisoning that occurs at query time via retrieved documents, not during model training |
| Passive poisoning | Malicious doc uploaded once; retrieved automatically by normal queries |
| Active manipulation | Content crafted to rank highly for specific sensitive queries |
| Context injection | Retrieved documents inserted into the LLM's context window before generation |
| Output drift | Gradual shift in system response behaviour caused by long-term exposure to poisoned content |
| Indirect prompt injection | Instructions embedded in retrieved data that the model follows — without the attacker touching the prompt |
| Defence-in-depth | Overlapping security controls across ingestion, retrieval, and monitoring |

---

*Write-up by Santosh | TryHackMe AI Security Path | Module 5 — Data Poisoning*
