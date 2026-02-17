# Multimodal RAG Workflow (n8n)

An automated document ingestion and retrieval-augmented generation (RAG) pipeline built in n8n. It watches a folder for new files, converts them to Markdown via Docling, extracts and serves embedded images, stores vector embeddings in Qdrant, and exposes a chat interface powered by a local Ollama LLM.

---

## Architecture Overview

```
Local File Trigger
       │
       ▼
Read File from Disk
       │
       ▼
Docling API (PDF → Markdown + images)
       │
       ├──────────────────────────────────────────┐
       ▼                                          ▼
Inject Base URL into image paths          Extract image filenames
(Code JS1)                                (Code JS)
       │                                          │
       ▼                                          ▼
Qdrant Vector Store (insert)              Split Out image names
  ├── Ollama Embeddings                           │
  └── Default Data Loader                        ▼
       └── Recursive Text Splitter        Execute Command
                                          (mv images to served dir)

─── Chat Interface ───────────────────────────────────────────────

When chat message received
       │
       ▼
AI Agent (Ollama LLM)
  └── kb_search tool (Qdrant retrieve-as-tool)
        └── Ollama Embeddings
```

---

## Node Reference

| Node | Type | Purpose |
|------|------|---------|
| **Local File Trigger** | `localFileTrigger` | Polls `/data/shared/rag-files/pending` for new files |
| **Read/Write Files from Disk** | `readWriteFile` | Reads the newly added file into binary data |
| **HTTP Request** | `httpRequest` | POSTs the file to the Docling service (`http://docling:5001/v1/convert/file`) for conversion to Markdown with referenced images |
| **Code in JavaScript** | `code` | Parses the Docling response to extract all embedded image filenames from the Markdown |
| **Code in JavaScript1** | `code` | Rewrites relative image paths in the Markdown to full URLs (`http://localhost:8080/`) so they render correctly in chat responses |
| **Split Out** | `splitOut` | Splits the array of image filenames into individual items for per-image processing |
| **Execute Command** | `executeCommand` | Moves each extracted image from the Docling scratch folder to `/data/shared/extracted-images` where they are served over HTTP |
| **Qdrant Vector Store (insert)** | `vectorStoreQdrant` | Inserts document chunks + embeddings into the `multimodal-rag` Qdrant collection |
| **Embeddings Ollama** | `embeddingsOllama` | Generates embeddings using `nomic-embed-text:latest` for ingestion |
| **Default Data Loader** | `documentDefaultDataLoader` | Loads the Markdown text into LangChain document format |
| **Recursive Character Text Splitter** | `textSplitterRecursiveCharacterTextSplitter` | Splits Markdown into 700-token chunks with Markdown-aware splitting |
| **When chat message received** | `chatTrigger` | Webhook-based chat entry point |
| **AI Agent** | `agent` | Orchestrates the chat response using the `kb_search` tool and the Ollama LLM |
| **Ollama Chat Model** | `lmChatOllama` | Chat LLM (`gpt-oss:20b`) used by the agent for response generation |
| **kb_search** | `vectorStoreQdrant` (retrieve-as-tool) | Retrieves the top-12 most relevant chunks from Qdrant; exposed to the agent as a tool |
| **Embeddings Ollama1** | `embeddingsOllama` | Generates query embeddings using `nomic-embed-text:latest` for retrieval |

---

## Prerequisites

| Service | Default Address | Notes |
|---------|-----------------|-------|
| **n8n** | — | Self-hosted instance |
| **Docling** | `http://docling:5001` | Document conversion microservice |
| **Qdrant** | configured via credential | Local vector database |
| **Ollama** | configured via credential | Hosts `nomic-embed-text:latest` and `gpt-oss:20b` |
| **Static file server** | `http://localhost:8080` | Serves extracted images from `/data/shared/extracted-images` |

---

## Folder Structure

```
/data/shared/
├── rag-files/
│   └── pending/          ← Drop new documents here
├── docling-scratch/       ← Docling writes temporary images here
└── extracted-images/      ← Final image location (served over HTTP)
```

---

## Setup

1. **Configure credentials** in n8n:
   - `qdrantApi` → point to your Qdrant instance (used by both `Local dataset` and the `kb_search` tool)
   - `ollamaApi` → Ollama instance hosting `nomic-embed-text:latest`
   - `ollamaApi` (account 2) → Ollama instance hosting `gpt-oss:20b`

2. **Create the Qdrant collection** named `multimodal-rag` before running the workflow. The collection must be compatible with the embedding dimensions produced by `nomic-embed-text` (768 dimensions).

3. **Start a static file server** rooted at `/data/shared/extracted-images` on port `8080`. Example with Python:
   ```bash
   cd /data/shared/extracted-images && python3 -m http.server 8080
   ```
   Or configure Nginx/Caddy to serve that directory.

4. **Ensure Docling is running** and reachable at `http://docling:5001`.

5. **Import and activate** the workflow in n8n.

---

## Usage

### Ingesting a Document

Drop any supported file (PDF, DOCX, etc.) into `/data/shared/rag-files/pending/`. The workflow will automatically:

1. Detect the new file.
2. Convert it to Markdown with extracted images via Docling.
3. Move images to the served directory.
4. Embed and store Markdown chunks in Qdrant.

### Chatting with Your Knowledge Base

Open the chat interface linked to the **When chat message received** webhook. Ask questions about any ingested documents. The AI agent will search the knowledge base and include any relevant images inline in its response.

---

## Configuration Notes

- **Chunk size**: Set to `700` tokens with Markdown-aware splitting. Adjust in the **Recursive Character Text Splitter** node if needed.
- **Retrieval top-K**: The `kb_search` tool retrieves the top `12` chunks per query. Tune this in the `kb_search` node.
- **Image base URL**: Hardcoded to `http://localhost:8080/` in **Code in JavaScript1**. Update this if your static file server runs elsewhere.
- **Image output directory**: Hardcoded to `/data/shared/extracted-images` in the **Execute Command** node.
- **Polling interval**: The Local File Trigger uses polling mode. Adjust the interval in the node settings if needed.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Images not rendering in chat | Static file server not running or wrong base URL | Verify the server is up on port 8080 and update the URL in **Code in JavaScript1** |
| No results from `kb_search` | Collection not created or embeddings not inserted | Check Qdrant for the `multimodal-rag` collection and re-run ingestion |
| Docling conversion fails | Service unreachable | Confirm Docling is running at `http://docling:5001` |
| Embeddings dimension mismatch | Wrong model used | Ensure both Ollama Embeddings nodes use `nomic-embed-text:latest` |
| Files not detected | Polling trigger not active | Activate the workflow and confirm the pending folder path is correct |
