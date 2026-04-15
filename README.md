# Linfox Logistics RAG Chatbot

[Overview](#overview) | [Chatbot in Action](#chatbot-in-action) | [Quick Start](#quick-start) | [Key Findings](#key-findings) | [Architecture](#architecture) | [Reports](#reports) | [Bug Fix](#upstream-bug-fix)

---

## Overview

A domain-specific RAG chatbot built on Azure AI Foundry, customised for Melbourne logistics operations using Linfox Australia as the domain. Allows operations staff to query depot locations, delivery zones, freight types, shift times, delay reasons and escalation procedures through natural language.

Forked from [Azure-Samples/get-started-with-ai-chat](https://github.com/Azure-Samples/get-started-with-ai-chat) and extended with custom data ingestion, infrastructure fixes and systematic behaviour analysis.

The primary purpose of this project is not to build a production-grade chatbot. It is to investigate and experiment with RAG behaviour, how chunking boundaries, prompt versions, and retrieval configurations affect what the system actually does in real conditions. Linfox Melbourne logistics operations was chosen as the domain to make the failure modes operationally meaningful rather than theoretical.

---

## Chatbot in Action

<div align="center">
  <img src="docs/images/chatbot_demo.gif" width="700" alt="Chatbot Demo">
</div>

*Five queries demonstrating correct retrieval, chunking failure, conversation history behaviour, information injection and out of scope handling. Full analysis in the [RAG Behaviour Analysis Report](docs/rag-analysis.md).*

---

## Quick Start

**Prerequisites:** Python 3.10+, Node.js 18+, Azure subscription, Azure Developer CLI (`azd`)

```powershell
# 1. Clone the repo
git clone https://github.com/deesk/linfox-rag-chatbot-azure.git
cd linfox-rag-chatbot-azure

# 2. Install dependencies
pip install -r src/api/requirements.txt

# 3. Deploy to Azure
azd up

# 4. Shut down when done
azd down --purge
```

> Deployment takes approximately 25 minutes.

> **Note:** Pre-generated embeddings are included in the repo. Only run `python scripts/build_embeddings.py` if you modify the knowledge base data.

---

## Updating the Knowledge Base

The Microsoft template has a built-in embedding process for its own sample data. To support custom domain data, a reusable pipeline was built: `scripts/build_embeddings.py`

This script reads `.txt` files from `src/static/data/`, generates embeddings using `text-embedding-3-small`, saves `embeddings.csv` to `src/api/data/`, and deletes the existing Azure Search index so it is recreated fresh on next deploy.

To use your own domain data, replace `src/static/data/melbourne-logistics.txt` then run:

```powershell
python scripts/build_embeddings.py
```

Then redeploy:

```powershell
azd up
```

---

## Architecture
<div align="center">
  <img src="docs/images/rag_analysis/d1_rag_query_flow.png" width="700" alt="RAG Query Flow">
</div>

A standard RAG pipeline — user query is embedded, matched against the logistics knowledge base in Azure AI Search, top 5 chunks injected into GPT-4o-mini alongside conversation history to generate a response.

Full architecture and data pipeline details in the [RAG Behaviour Analysis Report](docs/rag-analysis.md).

---

## Key Findings

RAG quality depends primarily on data structure, chunking strategy and prompt engineering, not the AI model itself.

**What worked:**
- Depot locations, shift times and escalation procedures returned correctly when content landed within a single retrieval chunk
- Out of scope queries handled gracefully without explicit instruction
- Conversation history supplemented incomplete RAG retrieval in real user sessions

**What failed:**
- System returned 3 of 4 freight types with full confidence. Dangerous goods missing due to a chunk boundary. No error signal to the user
- Delivery and loading zones contaminated each other because they shared Melbourne suburb names in content
- User injected false delay reasons. System accepted them, used real knowledge base data to make them sound credible, and carried them forward even when challenged

**Key insight:**
The most dangerous failure is not a sorry response. It is a confident wrong answer. A system that sounds certain while missing critical information gives the user no signal to question it.

Full investigation: [RAG Behaviour Analysis Report](docs/rag-analysis.md)

---

## Architecture
<div align="center">
  <img src="docs/images/rag_analysis/d1_rag_query_flow.png" width="700" alt="RAG Query Flow">
</div>

A standard RAG pipeline — user query is embedded, matched against the logistics knowledge base in Azure AI Search, top 5 chunks injected into GPT-4o-mini alongside conversation history to generate a response.

Full architecture and data pipeline details in the [RAG Behaviour Analysis Report](docs/rag-analysis.md).

---

## Tech Stack

| Component | Technology |
|---|---|
| AI Model | GPT-4o-mini (Azure AI Foundry) |
| Embeddings | text-embedding-3-small (100 dimensions) |
| Vector Search | Azure AI Search |
| Backend | Python, Gunicorn, Azure Container Apps |
| Frontend | React (Microsoft template) |
| Infrastructure | Azure Bicep, Azure Developer CLI (azd) |
| Data pipeline | Custom Python (`scripts/build_embeddings.py`) |

---

## Reports

**[RAG Behaviour Analysis Report](docs/rag-analysis.md)**
Systematic investigation across 6 parts and 8 findings covering chunking failures, vector contamination, conversation history behaviour, prompt injection and system prompt security. Each finding documented with evidence, methodology and root cause analysis.

**[Cost Analysis](docs/cost-analysis.md)**
Real Azure spend breakdown across 16 active development days (AU$2.59 total). Covers provisioned vs consumption billing, idle cost observations and production considerations.

---

## Upstream Bug Fix

A blocking bug was identified and fixed in this fork from the upstream Microsoft template.

**Internal Server Error on first load** ([Issue #129](https://github.com/Azure-Samples/get-started-with-ai-chat/issues/129))

After a successful `azd up` deployment, the app returned an Internal Server Error on every page load immediately. The root cause was a `TemplateResponse` syntax in `src/api/routes.py` incompatible with the current Starlette version on Python 3.13.

Fixed by updating the argument format:

```python
# Before (broken)
return templates.TemplateResponse(
    "index.html",
    {"request": request}
)

# After (fixed)
return templates.TemplateResponse(
    request=request,
    name="index.html"
)
```
---

*Portfolio proof of concept. Not affiliated with or endorsed by Linfox Australia.*