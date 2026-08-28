<div align="center">

# Legal RAG PT — n8n

### Fully local conversational RAG workflow for Portuguese legal documents

[![n8n](https://img.shields.io/badge/n8n-Workflow%20Automation-EA4B71?logo=n8n&logoColor=white)](https://n8n.io/)
[![Ollama](https://img.shields.io/badge/Ollama-Local%20Models-black)](https://ollama.com/)
[![Qdrant](https://img.shields.io/badge/Qdrant-Vector%20Database-DC244C)](https://qdrant.tech/)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

**n8n · RAG · bge-m3 · Qdrant · AMALIA-9B · Ollama**

</div>

---

## About

This project provides the conversational orchestration layer for **Legal RAG PT**, a fully local Retrieval-Augmented Generation system for natural-language consultation of the **Porto Municipal Regulatory Code (CRMP)**.

The workflow embeds each question with `bge-m3`, retrieves the five most relevant legal chunks from Qdrant, constructs grounded context, sends it to AMALIA-9B through Ollama, and returns an answer in European Portuguese with article and page references.

The upstream corpus processing and retrieval evaluation pipeline is maintained in [`legal-rag-pt`](https://github.com/ruialexrib/legal-rag-pt), while the complete technical documentation is available in [`legal-rag-pt-doc`](https://github.com/ruialexrib/legal-rag-pt-doc).

---

## Architecture

```text
User Question
     │
     ▼
n8n Chat Trigger
     │
     ▼
bge-m3 Query Embedding ──► Ollama
     │
     ▼
Qdrant Top-5 Vector Search
     │
     ▼
Context + Source Construction
     │
     ▼
AMALIA-9B ──► Ollama
     │
     ▼
Grounded Answer
     │
     ▼
Article + Page References
```

---

## Technology Stack

| Technology | Purpose |
| --- | --- |
| **n8n** | Workflow orchestration and chat interface |
| **bge-m3** | Query embeddings |
| **Qdrant** | Semantic vector retrieval |
| **AMALIA-9B** | Grounded answer generation |
| **Ollama** | Local model execution |
| **Docker Compose** | Local n8n runtime |

---

## Repository Structure

```text
legal-rag-pt-n8n/
├── workflows/
│   └── legal-rag-pt-n8n.json
├── docker-compose.yml
└── README.md
```

---

## Requirements

- Docker with Docker Compose
- Ollama running on the host
- Qdrant running on the host
- Populated `crmp_bge_m3` collection
- `bge-m3`
- `hf.co/ruialexrib/AMALIA-9B-0626-SFT-GGUF:Q3_K_M`

The Qdrant collection can be created by running notebooks `01` through `06` from the companion `legal-rag-pt` project.

---

## Quick Start

Install the required Ollama models:

```bash
ollama pull bge-m3
ollama pull hf.co/ruialexrib/AMALIA-9B-0626-SFT-GGUF:Q3_K_M
```

Start n8n:

```bash
docker compose up -d
```

Open n8n at `http://localhost:5678`, import `workflows/legal-rag-pt-n8n.json`, review the node configuration, and activate the workflow.

No external API credentials are required for the default setup because Ollama and Qdrant are accessed as local HTTP services.

---

## Workflow Nodes

| Node | Purpose |
| --- | --- |
| `Chat Trigger` | Receives the user's question |
| `Generate Embedding` | Generates a `bge-m3` query vector through Ollama |
| `Qdrant Search` | Retrieves the five closest legal chunks |
| `Build Context` | Constructs grounded context and preserves source metadata |
| `Generate Answer` | Generates an answer exclusively from retrieved context |
| `Format Response` | Appends article, title, and page references |

---

## Current Configuration

| Setting | Value |
| --- | --- |
| n8n | `localhost:5678` |
| Ollama from n8n | `host.docker.internal:11434` |
| Qdrant from n8n | `host.docker.internal:6333` |
| Embedding model | `bge-m3` |
| Embedding dimensions | `1024` |
| Collection | `crmp_bge_m3` |
| Retrieval limit | `5` chunks |
| Answer model | `hf.co/ruialexrib/AMALIA-9B-0626-SFT-GGUF:Q3_K_M` |
| Language | European Portuguese |
| Time zone | `Europe/Lisbon` |

---

## Grounding

The answer model is instructed to use only the retrieved CRMP context, avoid filling gaps with external knowledge, state when context is insufficient, answer in European Portuguese, and identify relevant legal articles when available.

These constraints reduce unsupported answers but do not guarantee factual accuracy. Outputs should always be checked against the official regulatory text.

---

## Security & Limitations

The supplied chat trigger is public by default and the workflow is designed primarily for a trusted local environment. Before external deployment, add authentication, HTTPS, access controls, secure n8n configuration, and appropriate logging policies.

The current implementation is tailored to the CRMP payload schema, uses vector-only retrieval, embeds service configuration directly in the workflow, and does not include automated workflow tests.

---

## Future Work

Planned improvements include environment-based configuration, hybrid retrieval, reranking, distinct-article handling, confidence-aware responses, automated workflow tests, and production-ready authentication.

---

## Disclaimer

This project is intended for experimental and educational purposes. Its output does not constitute legal advice and must be verified against the applicable official sources.

---

## License

This project is licensed under the [MIT License](LICENSE).
