# Local RAG with MongoDB, Ollama & LangChain.js

A private, zero-cost Retrieval-Augmented Generation (RAG) project for Windows.

Ingest PDFs, generate embeddings locally with Ollama, store vectors in MongoDB running in Docker, and query your documents with a local LLM. No OpenAI API key required.

> **Why Docker on Windows?** MongoDB’s native vector search engine is not available as a native Windows binary. Running MongoDB in Docker gives you full vector search support locally.

---

## Features

- 📄 PDF ingestion and text chunking
- 🧠 Local embeddings via Ollama
- 🗄️ MongoDB vector search running in Docker
- 🤖 Local LLM generation via Ollama
- 🔗 LangChain.js orchestration
- 🔒 Private by default — no external API calls
- 💸 Zero API cost
- 🪟 Windows-friendly setup using Docker Desktop + WSL2

---

## Architecture

```text
PDF
 │
 ▼
Document Loader → Text Splitter → Ollama Embeddings
                                      │
                                      ▼
                              MongoDB Vector Store
                                      │
Question → Embedding → Vector Search ─┘
                                      │
                                      ▼
                         Context + Question → Ollama LLM
                                      │
                                      ▼
                                   Answer
```

---

## Tech Stack

| Layer | Tool |
|---|---|
| Runtime | Node.js |
| Orchestration | LangChain.js |
| LLM | Ollama (`llama3.1` or similar) |
| Embeddings | Ollama (`nomic-embed-text`) |
| Vector Database | MongoDB with vector search |
| Container | Docker Desktop |
| Document Parsing | `pdf-parse` |

---

## Prerequisites

- Windows 10/11 with WSL2 enabled
- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [Node.js 20+](https://nodejs.org/)
- [Ollama](https://ollama.com/)
- Git

Verify installations:

```bash
node --version
npm --version
docker --version
ollama --version
```

---

## Quick Start

### 1. Clone the repository

```bash
git clone <your-repo-url>
cd local-rag-mongodb-ollama
npm install
```

### 2. Pull local Ollama models

```bash
ollama pull llama3.1
ollama pull nomic-embed-text
```

> You can use another LLM, but make sure the embedding model dimensions match your MongoDB vector index. `nomic-embed-text` uses **768 dimensions**.

### 3. Start MongoDB with vector search

```bash
docker compose up -d
```

Initialize the single-node replica set if this is the first run:

```bash
docker exec -it rag-mongodb mongosh --eval "rs.initiate()"
```

### 4. Configure environment variables

```bash
cp .env.example .env
```

Edit `.env` if needed.

### 5. Ingest a PDF

```bash
npm run ingest -- ./data/sample.pdf
```

### 6. Create the MongoDB vector index

Open MongoDB Shell:

```bash
mongosh "mongodb://localhost:27017/rag"
```

Create the vector search index:

```javascript
db.documents.createSearchIndex(
  "vector_index",
  "vectorSearch",
  {
    fields: [
      {
        type: "vector",
        path: "embedding",
        numDimensions: 768,
        similarity: "cosine"
      }
    ]
  }
)
```

> If your collection does not exist yet, run ingestion once first.
>
> If you use a different embedding model, update `numDimensions`:
> - `nomic-embed-text` → `768`
> - `all-minilm` → `384`

### 7. Query your documents

```bash
npm run query -- "What is this document about?"
```

---

## Example `docker-compose.yml`

```yaml
services:
  mongodb:
    image: mongodb/mongodb-community-server-search:7.0
    container_name: rag-mongodb
    command: ["mongod", "--replSet", "rs0", "--bind_ip_all"]
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db
    restart: unless-stopped

volumes:
  mongodb_data:
```

> Adjust the image tag to match the MongoDB vector-search image available in your environment.

Start it with:

```bash
docker compose up -d
```

---

## Environment Variables

Create `.env` from `.env.example`:

```env
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_LLM_MODEL=llama3.1
OLLAMA_EMBEDDING_MODEL=nomic-embed-text

MONGODB_URI=mongodb://localhost:27017
MONGODB_DB=rag
MONGODB_COLLECTION=documents
VECTOR_INDEX_NAME=vector_index
```

---

## Project Structure

```text
.
├─ data/
│  └─ sample.pdf
├─ src/
│  ├─ ingest.js
│  ├─ query.js
│  └─ lib/
│     ├─ mongodb.js
│     ├─ ollama.js
│     └─ rag.js
├─ docker-compose.yml
├─ .env.example
├─ package.json
└─ README.md
```

> If your project uses TypeScript, rename files to `.ts` and update the npm scripts accordingly.

---

## NPM Scripts

Example `package.json` scripts:

```json
{
  "scripts": {
    "ingest": "node src/ingest.js",
    "query": "node src/query.js"
  }
}
```

Usage:

```bash
npm run ingest -- ./data/sample.pdf
npm run query -- "Summarize the key points"
```

---

## How It Works

1. A PDF is loaded and split into smaller text chunks.
2. Each chunk is converted into an embedding using Ollama.
3. Embeddings and chunk metadata are stored in MongoDB.
4. A vector search index is created on the `embedding` field.
5. When a user asks a question, the question is embedded.
6. MongoDB vector search retrieves the most relevant chunks.
7. The retrieved context is passed to the local LLM through Ollama.
8. The LLM generates an answer grounded in the retrieved document content.

> The LangChain.js `MongoDBAtlasVectorSearch` class works with the local Docker-based MongoDB vector search instance despite the “Atlas” name.

---

## Troubleshooting

| Problem | Likely Fix |
|---|---|
| `MongoServerError: no such index` | Create the vector search index named `vector_index` |
| Dimension mismatch error | Make `numDimensions` match your embedding model. Re-ingest after changing models |
| `ECONNREFUSED` to Ollama | Start Ollama and confirm it runs on `http://localhost:11434` |
| Docker container exits immediately | Ensure Docker Desktop and WSL2 are running; check `docker logs rag-mongodb` |
| PDF produces no text | Use a text-based PDF, not a scanned image PDF. OCR may be required |
| Slow generation | Use a smaller Ollama model, reduce chunk size, or retrieve fewer chunks |
| Vector index not returning results | Confirm the index name and that documents were ingested after the index was created |

Useful debug commands:

```bash
docker ps
docker logs rag-mongodb
ollama list
mongosh "mongodb://localhost:27017/rag"
```

---

## Security Notes

This project is intended for **local development and learning**.

- Do not expose MongoDB or Ollama directly to the public internet.
- Add authentication before deploying outside your machine.
- Treat ingested documents as sensitive if they contain private data.

---

## Roadmap

- [ ] Add a React UI
- [ ] Add chat history
- [ ] Support multiple PDFs and folders
- [ ] Add source citations in answers
- [ ] Add streaming responses
- [ ] Add Docker Compose setup for the full app
- [ ] Add evaluation scripts for retrieval quality

---

## Contributing

Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

```bash
git checkout -b feature/my-feature
git commit -m "Add my feature"
git push origin feature/my-feature
```

---

## License

MIT License. See `LICENSE` for details.

---

## Acknowledgements

- [Ollama](https://ollama.com/)
- [MongoDB](https://www.mongodb.com/)
- [LangChain.js](https://js.langchain.com/)
- [Docker](https://www.docker.com/)
