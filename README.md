# RAG-Powered Customer Support Triage and SLA Routing Engine

An end-to-end, self-hosted **AI Support Operations & Ticket Triage Workflow** built with **n8n**, **Docker**, and **Ollama** LLM. This system automates incoming customer ticket classification, grounds response drafts strictly in internal SLA documentation via **Retrieval-Augmented Generation (RAG)**, enforces dynamic SLA routing rules, and logs structured audit trails—running 100% locally with zero external API costs.

## 🎯 Business Problem & Impact

Manual ticket triage in SaaS and CRM operations leads to missed Service Level Agreements (SLAs), inconsistent customer replies, and routing bottlenecks. 

This workflow solves three core operational gaps:
1. **Zero-Touch Intake & Classification:** Eliminates manual tagging by deterministically extracting `priority`, `category`, and `sentiment` from unstructured customer submissions.
2. **Hallucination-Free Policy Grounding:** Uses local vector search over company SOPs (`support_policy.txt`) so agents receive accurate, policy-backed draft replies rather than generic LLM guesses.
3. **Automated SLA Enforcement:** Automatically routes critical issues to Tier-2 escalation queues with a 2-hour SLA while directing standard inquiries to the General Support Desk (24-hour SLA), logging every decision to a persistent CSV audit file.

## 🏗️ System Architecture

The n8n canvas is structured into two decoupled workflows: 
- **Ingestion Pipeline** (knowledge base indexing) and a
- **Live Production Pipeline** (real-time intake, classification, RAG retrieval, routing, and logging).

```text
[TOP FLOW: KNOWLEDGE BASE INGESTION (Run once per container restart)]
Manual Trigger ──► Read Policy File (support_policy.txt)
                          │
                          ▼
               Default Data Loader + Recursive Splitter (500 chars / 50 overlap)
                          │
                          ▼
               Embeddings Ollama (all-minilm) ──► Simple Vector Store [Insert: cloudpulse_policy as memory key]
```
<img width="903" height="373" alt="image" src="https://github.com/user-attachments/assets/2a3bbd14-68ed-45da-974b-a625eb212869" />

```
[BOTTOM FLOW: LIVE PRODUCTION TRIAGE & ROUTING]
n8n Form Trigger (Customer Name, Issue/Feedback)
       │
       ├──► Basic LLM Chain (qwen2.5:1.5b, Temp=0) + Structured Output Parser
       │         └─► Extracts JSON: { priority, category, sentiment }
       │
       ├──► Question & Answer Chain (qwen2.5:1.5b, Temp=0)
       │         └─► Vector Store Retriever (Top-4 chunks from cloudpulse_policy)
       │         └─► Generates policy-grounded Suggested Reply
       │
       ▼
  [IF Node: priority == "high"]
   ├── TRUE  ──► Edit Fields: Tier-2 Escalation Squad (2-Hour SLA Alert)
   └── FALSE ──► Edit Fields1: General Support Desk (24-Hour SLA Target)
       │
       ▼
  Convert to File (CSV) ──► Append/Write to Disk (/home/node/.n8n-files/routed_tickets.csv)
       │
       ▼
  n8n Form Ending (Displays Ticket Summary, Assigned Team, SLA Target & Suggested Reply)
```
<img width="1386" height="547" alt="image" src="https://github.com/user-attachments/assets/a33c2c1b-2877-4e91-94fb-41d730e5b0d1" />


## 🛠️ Tech Stack & Infrastructure

* **Workflow Orchestration:** n8n (`v2.47.7`) running in Docker with WebSocket push backend (`N8N_PUSH_BACKEND=websocket`) and persistent storage volumes.
* **Local LLM Inference:** Ollama running natively on Windows (`OLLAMA_HOST=0.0.0.0`), bridged into Docker via `http://host.docker.internal`.
* **Models Used (Optimized for 8GB RAM / 2GB WSL2 Allocation):**
  * **Chat & Reasoning:** `qwen2.5:1.5b` (`Temperature = 0` for deterministic JSON parsing and strict policy adherence).
  * **Vector Embeddings:** `all-minilm` (fast, lightweight semantic embeddings).
* **Vector Database:** n8n In-Memory Vector Store (`cloudpulse_policy` namespace).
* **Audit & Persistence:** Docker bind-mounted volume (`/home/node/.n8n-files`) generating structured CSV records (`routed_tickets.csv`).


## ✨ Key Engineering Highlights

* **Resource-Constrained Optimization:** Engineered to run smoothly on an 8 GB RAM host machine (with WSL2 capped at 2 GB RAM / 2 vCPUs via `.wslconfig`) by pairing quantized `qwen2.5:1.5b` and `all-minilm` models with lightweight chunking (`500` size / `50` overlap).
* **Deterministic Structured Outputs:** Uses keyword-anchored few-shot prompting paired with n8n's `Structured Output Parser` to guarantee clean JSON schema extraction (`priority`, `category`, `sentiment`) even on a 0.5B parameter model.
* **Strict RAG Guardrails:** System prompt enforces `{context}` grounding against *CloudPulse CRM Support & SLA Policy Manual*, preventing fabricated refund rules or unsupported technical promises.
* **Closed-Loop User Experience:** Configured `Respond When: Workflow Finishes` on the Form Trigger so submitters receive an immediate, formatted resolution summary screen upon processing.

## 🚀 Quickstart & Local Setup

### 1. Prerequisites
* Docker Desktop & Ollama installed locally.
* Pull the required lightweight models in your terminal:
  ```bash
  ollama pull qwen2.5:1.5b
  ollama pull all-minilm
  ```
* Ensure Ollama accepts container connections by setting the environment variable `OLLAMA_HOST=0.0.0.0`.

### 2. Launch n8n in Docker
```command prompt
docker run -d \
  --name n8n \
  --restart always \
  -p 5678:5678 \
  -e N8N_PUSH_BACKEND=websocket \
  -v n8n_data:/home/node/.n8n \
  -v n8n_files:/home/node/.n8n-files \
  n8nio/n8n:2.47.7
```
- Grant `/home/node/.n8n-files` folder permissions to `node:node` and copy `data/support_policy.txt` into this volume).*
<img width="1907" height="720" alt="image" src="https://github.com/user-attachments/assets/14bf168d-5745-4238-86e0-726a934d4bf4" />
<img width="1245" height="771" alt="image" src="https://github.com/user-attachments/assets/31f05ff0-8621-466a-8c3c-441fa5c0b38f" />

### 3. Import & Run
1. Open `http://localhost:5678` and import `workflow/rag_ticket_triage_workflow.json`.
2. Configure the **Ollama Credential** pointing to `http://host.docker.internal`.
3. Click **Execute Workflow** on the top **Manual Trigger** node once to embed `support_policy.txt` into the `cloudpulse_policy` vector store.
4. Activate the workflow, open the **n8n Form Trigger** production URL, submit a test ticket (e.g., *"Production API sync is failing with 500 errors and blocking our sales team"*), and view the automated SLA triage output.

Output:
<img width="1665" height="798" alt="image" src="https://github.com/user-attachments/assets/b147c7f3-be86-40dd-a117-78a8d9a821b2" />
1. Customer Support Ticket form, when submitted - with workflow executing in background

<img width="1715" height="757" alt="image" src="https://github.com/user-attachments/assets/7b2add34-56d0-44ac-84e8-c5597dc1ba70" />
2. When Support ticket is logged and routed to assigned team, with suggested reply (fixes) to customer generated based on support policy provided to model
