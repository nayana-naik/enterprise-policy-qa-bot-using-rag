# Starter RAG — Enterprise Policy Q&A Bot (n8n, no-code)

A no-code RAG pipeline built entirely from n8n nodes (Document Loader → Text
Splitter → Embeddings → Vector Store → Retrieval Chain), running on a local
Ollama model by default, with a public chat screen for Q&A.

Dataset: `data/knowledge_items.csv` (your `rag_sample_qas_from_kis.csv`) —
10 IT-support knowledge articles, each with `ki_topic`, `ki_text`,
`sample_question`, `sample_ground_truth`. The last two columns double as a
built-in eval set.

## Stack

- **n8n** — orchestration, no Python required (`localhost:5678`)
- **Ollama** (local, default) — `nomic-embed-text` for embeddings, `llama3.2:3b`
  for chat. Swappable to a public LLM (OpenAI/Anthropic/etc.) — see below.
- **Vector store** — choose one:
  - **Qdrant Cloud free cluster** (default/recommended) — persistent, no
    containers to run yourself, 1GB free tier at
    [cloud.qdrant.io](https://cloud.qdrant.io/accounts)
  - **Qdrant via Docker** (alternate) — same workflows, just point the
    credential at a local container instead of the cloud cluster
  - **n8n's built-in In-Memory Vector Store** — zero extra infra at all, but
    data is lost on n8n restart and only shared between workflows *in the
    same running n8n process*

## Files

```
data/knowledge_items.csv          the dataset
n8n-workflows/
  01-ingest-qdrant.json            Ingest + Chunk + Embed -> Qdrant
  01-ingest-inmemory.json          Ingest + Chunk + Embed -> In-Memory
  02-qna-chat-qdrant.json          Query + Generate, chat screen (Qdrant)
  02-qna-chat-inmemory.json        Query + Generate, chat screen (In-Memory)
  03-eval-batch.json               Optional: LLM-as-judge eval over the CSV
docker-compose.yml                 Fresh-start stack: just n8n (uses Qdrant Cloud by default)
docker-compose.qdrant.yml          ALTERNATE: local Qdrant instead of the cloud free tier
```

## 0. Prerequisites

- [Ollama](https://ollama.com) installed and running on your machine
- Docker (for n8n, and for Qdrant if you pick the local alternate option)
- A free [Qdrant Cloud](https://cloud.qdrant.io/accounts) account — only if
  using the default vector store option (skip if you want local Qdrant or
  In-Memory instead)

Pull the two models used by default:

```bash
ollama pull nomic-embed-text
ollama pull llama3.2:3b
```

(Any Ollama chat model works — bigger models generally give better answers,
but see the note on tool-calling below if you build on top of this with an
Agent node.)

## 1. Set up your vector store

### Option A — Qdrant Cloud free cluster (default)

1. Sign up / log in at [cloud.qdrant.io/accounts](https://cloud.qdrant.io/accounts)
2. Create a cluster — the Free tier (1GB) is enough for this dataset
3. From the cluster's dashboard, copy:
   - **Cluster URL** (looks like `https://xxxxxxxx-xxxx-xxxx.us-east4-0.gcp.cloud.qdrant.io:6333`)
   - **API Key** (create one if you don't already have one)

You'll paste both into an n8n credential in step 3 — no Docker container
needed for Qdrant with this option.

### Option B — Qdrant via Docker (alternate)

```bash
docker compose -f docker-compose.qdrant.yml up -d
```

If n8n also runs in Docker, connect the two:

```bash
docker network connect <your-n8n-docker-network> rag-qdrant
```

(Find your n8n container's network with
`docker inspect <n8n-container> --format '{{json .NetworkSettings.Networks}}'`.)
Your Qdrant credential in n8n will then use URL `http://rag-qdrant:6333` (or
`http://localhost:6333` if n8n runs outside Docker) with an empty API key,
instead of the cloud cluster's URL/key.

### Option C — In-Memory (no vector DB at all)

Skip both of the above — use `01-ingest-inmemory.json` /
`02-qna-chat-inmemory.json` instead of the Qdrant variants in step 4 onward.

## 2. Start n8n

**If you don't have n8n running yet:**

```bash
docker compose up -d
```

n8n is now at **http://localhost:5678**, and reaches Ollama on your host via
`http://host.docker.internal:11434`.

**If you already have n8n running** (e.g. Docker Desktop, `get-n8n.sh`, n8n
Desktop...), just use it — nothing extra to start for n8n itself.

## 3. Load the dataset into n8n

The "Read/Write Files from Disk" node can only read inside n8n's allowed
file path, `~/.n8n-files` (an n8n security default). Copy the CSV there:

```bash
# find your n8n container name, e.g. `docker ps`
docker exec <n8n-container> mkdir -p /home/node/.n8n-files/rag-data
docker cp data/knowledge_items.csv <n8n-container>:/home/node/.n8n-files/rag-data/knowledge_items.csv
```

(If you mapped `./data` to `/home/node/.n8n-files/rag-data` via
`docker-compose.yml`'s volumes, as that file already does, you can skip this —
it's already there.)

## 4. Import the workflows

In n8n: **Workflows → Import from File**, and import whichever pair you want:

- `01-ingest-qdrant.json` + `02-qna-chat-qdrant.json`, **or**
- `01-ingest-inmemory.json` + `02-qna-chat-inmemory.json`

Each workflow references two credentials you need to create once (**Credentials
→ Add Credential**):

- **Ollama** (`ollamaApi`) — Base URL: `http://host.docker.internal:11434`
  (or `http://localhost:11434` if n8n runs outside Docker)
- **Qdrant API** (`qdrantApi`, only needed for the Qdrant variant), named
  **"Qdrant Cloud (free tier)"** to match what the workflow JSON expects:
  - Default (Option A): URL = your cluster URL from step 1, e.g.
    `https://xxxxxxxx-xxxx-xxxx.us-east4-0.gcp.cloud.qdrant.io:6333`; API Key
    = your cluster's API key
  - Alternate (Option B, local Docker): URL = `http://rag-qdrant:6333`
    (or `http://localhost:6333`); API Key = leave blank

Open each imported node that shows a credential warning and pick the
credential you just created (n8n credential IDs don't carry over between
instances on import).

## 5. Ingest → Chunk + Embed

Open **RAG 01 - Ingest Knowledge Base**, click **Execute Workflow**. It will:

1. **Ingest**: read `knowledge_items.csv`, parse rows (`ki_topic`, `ki_text`, ...)
2. **Chunk**: split each `ki_text` into ~500-character chunks (50-char overlap)
   via the Recursive Character Text Splitter, tagging each chunk with its
   `topic` as metadata
3. **Embed**: embed each chunk with `nomic-embed-text` via Ollama
4. **Store**: upsert into the `enterprise_kb` collection (Qdrant) or the
   `enterprise_kb` in-memory store

Re-run this any time you edit the CSV or add new documents (for Qdrant it
upserts; for In-Memory it clears and reloads the whole store each run).

## 6. Query + Generate (the QnA screen)

Open **RAG 02 - QnA Chat**, click the **Active** toggle (top-right) to turn
the workflow on, then open the **"When chat message received"** node and
click **Open Chat** (or copy its webhook URL) — that's your QnA screen,
served by n8n itself at `localhost:5678`.

Under the hood, per question: embed the question → similarity search the
vector store (top 4 chunks) → stuff those chunks into a prompt → generate
the answer with the chat model (Question and Answer Chain node).

> **Note on the AI Agent node**: n8n's AI Agent + Vector-Store-as-Tool pattern
> (the "no-code agentic RAG" approach) is what's described in n8n's own docs,
> and it's wired the same way here conceptually — but in testing, Ollama's
> tool-calling was unreliable with local models (the agent called the
> retrieval tool, got the right chunks back, and then still answered "I don't
> know"). The shipped workflows use the simpler, non-agentic **Question and
> Answer Chain** instead, which was reliable in every test. If you swap to a
> cloud LLM with solid tool-calling (OpenAI, Anthropic), the Agent + Vector
> Store Tool pattern is worth revisiting for multi-tool workflows.

## 7. Swapping Ollama → a public LLM

Each workflow has exactly two Ollama nodes to swap:

1. **Embeddings Ollama** → replace with **Embeddings OpenAI** (or Cohere,
   etc.) — note: you must re-run ingestion if you change embedding models,
   since vector dimensions differ.
2. **Ollama Chat Model** → replace with **OpenAI Chat Model** / **Anthropic
   Chat Model**, pick a model, attach your API credential.

No other node needs to change — the Question and Answer Chain / vector
store nodes are model-agnostic.

## 8. Eval (optional, run later)

`03-eval-batch.json` uses the `sample_question` / `sample_ground_truth`
columns already in the CSV as a built-in eval set:

1. Re-reads the CSV
2. For each row, runs the question through the same retrieval + generation
   path (Qdrant Cloud by default — swap the vector store node/credential for
   local Qdrant or In-Memory if needed)
3. An **LLM Judge** (Basic LLM Chain) scores the generated answer against
   `sample_ground_truth` on a 1–5 scale with a one-line reason
4. Writes `eval_results.csv` to `~/.n8n-files/rag-data/` inside the n8n
   container (`docker cp` it back out, or mount `./data` as shown in
   `docker-compose.yml`)

This hasn't been run end-to-end yet — import it, click **Execute Workflow**,
and check the output. If something's off, the usual suspects are credential
selection (step 3) and the CSV path (step 2).

## Cleanup

If you imported workflows into an n8n instance that already had other
projects on it, you may see leftover `TEST*`-prefixed workflows from
development — safe to delete from the n8n UI, they're not part of this
project.
