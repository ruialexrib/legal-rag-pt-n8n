# Legal RAG PT — n8n

An n8n workflow for asking natural-language questions about the **Porto Municipal Regulatory Code (CRMP)** using a fully local Retrieval-Augmented Generation (RAG) pipeline.

The workflow embeds each question with `bge-m3`, retrieves the five most relevant legal chunks from Qdrant, sends the grounded context to a Portuguese legal language model through Ollama, and returns an answer with article and page references.

> This repository contains the orchestration layer. The document processing, embedding generation, Qdrant indexing, and retrieval evaluation pipeline is maintained in the companion `legal-rag-pt` repository.

## Workflow

```text
User question
    ↓
n8n Chat Trigger
    ↓
Query embedding — bge-m3 via Ollama
    ↓
Top-5 vector search — Qdrant
    ↓
Context and source construction
    ↓
Grounded answer — AMALIA via Ollama
    ↓
Answer with article and page references
```

## Components

- [n8n](https://n8n.io/) for workflow orchestration and the chat interface
- [Ollama](https://ollama.com/) for local embedding and answer-generation models
- [Qdrant](https://qdrant.tech/) for semantic vector search
- `bge-m3` as the embedding model
- `hf.co/ruialexrib/AMALIA-9B-0626-SFT-GGUF:Q3_K_M` as the answer-generation model
- Docker Compose for running n8n

## Repository structure

```text
legal-rag-pt-n8n/
├── workflows/
│   └── legal-rag-pt-n8n.json   # Exported n8n workflow
├── docker-compose.yml           # Local n8n service
└── README.md
```

## Prerequisites

- Docker with Docker Compose
- Ollama running on the host machine
- Qdrant running on the host machine
- A populated Qdrant collection named `crmp_bge_m3`
- The following Ollama models:
  - `bge-m3`
  - `hf.co/ruialexrib/AMALIA-9B-0626-SFT-GGUF:Q3_K_M`

The `crmp_bge_m3` collection can be created and populated by running notebooks `01` through `06` from the companion `legal-rag-pt` project.

## Setup

### 1. Prepare Ollama

Download the embedding and answer-generation models:

```bash
ollama pull bge-m3
ollama pull hf.co/ruialexrib/AMALIA-9B-0626-SFT-GGUF:Q3_K_M
```

Confirm that Ollama is available on port `11434`:

```bash
ollama list
```

### 2. Prepare Qdrant

Start Qdrant and ensure that the `crmp_bge_m3` collection has been populated with the CRMP vectors and payloads.

The workflow expects Qdrant at:

```text
http://host.docker.internal:6333
```

The indexed payloads must include these fields:

- `article`
- `article_title`
- `page_start`
- `page_end`
- `text`

### 3. Start n8n

From the repository root, run:

```bash
docker compose up -d
```

Open n8n at [http://localhost:5678](http://localhost:5678).

The Docker volume `n8n_data` persists workflows, credentials, and instance settings across container restarts.

### 4. Import the workflow

1. Open the n8n interface.
2. Create or select a project.
3. Choose **Import from File**.
4. Select `workflows/legal-rag-pt-n8n.json`.
5. Review the node settings and save the workflow.
6. Activate the workflow if it is not already active.

The exported workflow does not require external API credentials because Ollama and Qdrant are accessed as local HTTP services.

## Usage

Open the workflow and use its chat interface to submit a question in Portuguese, for example:

```text
Que documentos são necessários para apresentar um requerimento?
```

The response contains:

- An answer in European Portuguese
- Relevant CRMP article references
- Article titles when available
- Source page numbers
- A list of the five retrieved sources

## Node reference

| Node | Purpose |
|---|---|
| `Chat Trigger` | Receives the user's question through the n8n chat interface. |
| `Generate Embedding` | Sends the question to Ollama and generates a `bge-m3` vector. |
| `Qdrant Search` | Retrieves the five closest vectors from `crmp_bge_m3`, including their payloads. |
| `Build Context` | Formats retrieved chunks and preserves source metadata for the final response. |
| `Generate Answer` | Prompts AMALIA to answer exclusively from the retrieved context. |
| `Format Response` | Appends article, title, and page references to the generated answer. |

## Current configuration

| Setting | Value |
|---|---|
| n8n URL | `http://localhost:5678` |
| Ollama URL from n8n | `http://host.docker.internal:11434` |
| Qdrant URL from n8n | `http://host.docker.internal:6333` |
| Embedding model | `bge-m3` |
| Embedding dimensions | `1024` |
| Qdrant collection | `crmp_bge_m3` |
| Retrieval limit | `5` chunks |
| Answer model | `hf.co/ruialexrib/AMALIA-9B-0626-SFT-GGUF:Q3_K_M` |
| Answer language | European Portuguese |
| Time zone | `Europe/Lisbon` |

## Grounding behavior

The system prompt instructs the answer model to:

- Use only the retrieved CRMP context
- Avoid filling gaps with external knowledge
- State clearly when the retrieved context is insufficient
- Answer in European Portuguese
- Identify relevant legal articles when available

These instructions reduce unsupported answers but do not guarantee factual accuracy. Outputs should always be checked against the official regulatory text.

## Networking notes

The n8n container accesses Ollama and Qdrant through `host.docker.internal`. The Docker Compose configuration maps this hostname to the host gateway, including on supported Linux installations.

If Ollama or Qdrant runs on another machine or inside a different Docker network, update the URLs in these workflow nodes:

- `Generate Embedding`
- `Qdrant Search`
- `Generate Answer`

## Security

The imported `Chat Trigger` is configured as public. Before exposing n8n outside a trusted local environment:

- Add appropriate authentication and access controls
- Review n8n's deployment and encryption settings
- Avoid exposing Ollama or Qdrant directly to the public internet
- Configure HTTPS through a trusted reverse proxy
- Review workflow execution logs for potentially sensitive questions or retrieved text

## Troubleshooting

### n8n cannot connect to Ollama

- Confirm that Ollama is running on the host.
- Verify that port `11434` is reachable from the container.
- Confirm that both required models appear in `ollama list`.

### Qdrant returns a collection-not-found error

- Confirm that Qdrant is running on port `6333`.
- Run the indexing pipeline from the companion `legal-rag-pt` repository.
- Verify that the collection is named exactly `crmp_bge_m3`.

### Qdrant reports a vector-size mismatch

The query and document embeddings must use the same model. Rebuild the collection or update the workflow so both use `bge-m3` with 1,024-dimensional vectors.

### The answer lacks useful context

- Inspect the output of `Qdrant Search` and verify the retrieved payloads.
- Confirm that the collection contains the expected legal text and metadata.
- Consider adjusting the retrieval limit or improving the upstream chunking and evaluation pipeline.

## Limitations

- The workflow is tailored to the CRMP collection and payload schema.
- Retrieval is vector-only; hybrid search and reranking are not implemented.
- Service URLs, collection names, and models are embedded directly in the workflow.
- The answer model can still produce incorrect or incomplete output.
- The chat trigger is public by default.
- No automated workflow tests are currently included.

## Roadmap

- Move service URLs and model names into environment-based configuration.
- Add hybrid retrieval and reranking.
- Return only distinct articles when several retrieved chunks belong to the same article.
- Add confidence-aware handling for low-scoring retrieval results.
- Add automated workflow tests and deployment checks.
- Add authentication and production-ready n8n configuration.

## Disclaimer

This project is intended for experimental and educational purposes. Its output does not constitute legal advice and must be verified against the applicable official sources.

## License

This repository does not yet include a license. Before publishing it, choose an appropriate license for the workflow and configuration files and confirm the reuse terms that apply to the source document and derived data.
