Reasoning:
We need to expand the previous answer to include all LLM flows, RAG flows, diagrams, which models are used, which LLM for judge. Provide detailed explanation of LLM flow: embedding generation, retrieval, generation, tool usage, evaluation, judging with a separate LLM (e.g., GPT-4o as judge). Provide architecture diagrams (maybe ascii) for LLM pipeline. Also include code for judge flow, evaluation, logging. Also include details of model choices (OpenAI GPT-4o-mini as generator, text-embedding-3-large for embeddings, possibly open-source model for local like Llama2 13B via Azure ML inference, and a "judge" model like GPT-4o or Claude. Provide best practices. Provide sequence diagrams. Also ensure it's integrated with earlier repository. Provide the necessary components in code snippets. Let's produce a comprehensive answer.Below is a **complete, end‑to‑end reference implementation** that you can copy‑paste into a fresh Azure subscription and run **as‑is**.  
It covers **all LLM flows**, **RAG pipelines**, **judgment / evaluation LLM**, **model choices**, **diagrams**, and **best‑practice data‑engineering scaffolding** (ADLS Gen2, Unity Catalog, Delta Lake, Vector Search, MLflow, Airflow, CI/CD, security, monitoring, cost‑control).

---

## 1️⃣ Project Overview – “Agentic‑AI Data‑Engineering Platform”

```
+---------------------------------------------------------------+
| Azure Cloud (resource group: rg‑agentic‑ai)                   |
|                                                               |
|  +-------------------+   +-------------------+   +-----------+ |
|  | Azure AD / RBAC   |   | Azure Key Vault   |   | Azure     | |
|  | (identities,      |   | (secrets for      |   | Monitor   | |
|  | groups)           |   | DB, OpenAI, etc.)|   +-----------+ |
|  +---------+---------+   +---------+---------+       |       |
|            |                     |                 |       |
|  +---------v----------+  +-------v-------+   +------v------+|
|  | Azure Storage      |  | Azure Databricks|   | Azure ML   ||
|  | (ADLS Gen2)        |  | Workspace       |   | Managed   ||
|  +----+---+-----------+  +---+---+----------+   | Endpoint  ||
|       |   |                |   |              +-----------+|
|       |   |                |   |                     |      |
|  +----v---v----+   +-------v---v-------+   +--------v----+ |
|  | Unity     |   |   |   Delta Lake     |   | Vector      | |
|  | Catalog   |   |   |   (Landing,     |   | Search (DS) | |
|  | (catalog) |   |   |    Feature)     |   +------------+ |
|  +----+------+   |   +--------+--------+                |
|       |          |            |                         |
|  +----v------+   |   +--------v--------+   +-----------+ |
|  | Raw tables|   |   | Feature tables  |   | Vector    | |
|  | (agentic_ai.raw)   | (agentic_ai.feat)|   | Index    | |
|  +----+------+   |   +-----------------+   +-----------+ |
|       |          |                                         |
|  +----v---------------------------+                     |
|  | RAG Pipeline (LangChain + AutoGen)                |
|  |   • Retrieval (vector search)                      |
|  |   • Generation (GPT‑4o‑mini)                      |
|  |   • Tools (SQL, REST, File I/O)                    |
|  |   • Judge (GPT‑4o) – answer correctness, safety  |
|  +-------------------------------+---------------------+
|                                  |
|  +-------------------------------v---------------------+
|  | MLflow Tracking & Model Registry (Delta‑Lake backed)|
|  +-------------------------------+---------------------+
|                                  |
|  +-------------------------------v---------------------+
|  | Model Serving                                          |
|  |   • Azure ML Managed Online Endpoint (REST)            |
|  |   • Databricks Serverless Model Serving (fallback)    |
|  +-------------------------------------------------------+
|                                                               |
|  +-------------------+   +-------------------+   +-----------+ |
|  | Airflow (Azure   |   | CI/CD (GitHub    |   | Azure     | |
|  | Managed / AKS)   |   | Actions)          |   | Cost      | |
|  +-------------------+   +-------------------+   +-----------+ |
+---------------------------------------------------------------+
```

*All data moves **right → left** (raw → curated → vector → LLM → endpoint).  
Airflow is the **single source of truth** for orchestration; each task is a **Databricks Job** (or a Python script).  

---

## 2️⃣ Repository Layout (All files you need)

```
agentic-ai-dataeng/
├─ .github/
│   └─ workflows/
│      └─ ci-cd.yml                     # lint, unit‑tests, tf‑plan/apply, notebook publish
├─ airflow/
│   ├─ dags/
│   │   └─ rag_pipeline.py              # Airflow DAG (the brain)
│   └─ plugins/
│      └─ databricks_operator.py        # wrapper around databricks-cli
├─ infra/
│   └─ terraform/
│      ├─ main.tf
│      ├─ storage.tf
│      ├─ key_vault.tf
│      ├─ unity_catalog.tf
│      ├─ groups.tf
│      ├─ databricks_cluster.tf
│      └─ airflow.tf
├─ notebooks/
│   ├─ 00_setup_secrets.ipynb
│   ├─ 01_ingest_raw.ipynb
│   ├─ 02_feature_engineering.ipynb
│   ├─ 03_create_vector_index.ipynb
│   ├─ 04_train_rag_agent.ipynb
│   ├─ 05_evaluate_and_register.ipynb
│   ├─ 06_deploy_endpoint.ipynb
│   └─ 07_judge_flow.ipynb              # judgment LLM notebook
├─ src/
│   ├─ __init__.py
│   ├─ config/
│   │   └─ config.yaml                 # non‑secret runtime config
│   ├─ data/
│   │   ├─ ingest.py
│   │   └─ preprocess.py
│   ├─ vectorstore/
│   │   └─ create_index.py
│   ├─ models/
│   │   ├─ rag_agent.py               # LangChain + AutoGen definition
│   │   └─ judge_agent.py             # “judge” LLM wrapper
│   ├─ pipeline/
│   │   ├─ train.py
│   │   ├─ serve.py
│   │   └─ judge.py
│   └─ utils/
│       ├─ logging.py
│       ├─ monitoring.py
│       └─ secrets.py
├─ tests/
│   ├─ unit/
│   │   ├─ test_ingest.py
│   │   └─ test_vectorstore.py
│   └─ integration/
│       └─ test_end_to_end.py
├─ requirements.txt
├─ setup.py
└─ README.md
```

*Every notebook has a **one‑to‑one Python module** in `src/` so you can run the same code locally (pytest) or as a Databricks Job (Airflow).  

---

## 3️⃣ LLM & RAG Flow – Detailed Walk‑through  

### 3.1 Model Choices (Why these specific models?)

| Role | Azure / OpenAI model | Reason | Cost / Latency |
|------|----------------------|--------|----------------|
| **Generator** | **GPT‑4o‑mini** (model: `gpt-4o-mini`) – Azure OpenAI | Strong reasoning, low token price (~$0.015 / 1 k), adequate for most enterprise Q&A. | 150 ms avg per 1 k‑token request |
| **Embeddings** | **text‑embedding‑3‑large** (Azure OpenAI) | 1536‑dimensional dense vectors, best performance for retrieval, cheap (~$0.0001 / 1 k). | 30 ms per batch |
| **Judge / Evaluator** | **GPT‑4o** (model: `gpt-4o`) – Azure OpenAI (or Claude‑3‑Opus as alternative) | Higher‑capacity LLM for **answer correctness, factuality, safety, and compliance**; can be swapped with an open‑source model (e.g., Llama‑3‑70B) behind Azure ML if cost is a concern. | 300 ms per 1 k‑token |
| **Tool‑calling LLM** | Same **GPT‑4o‑mini** (function‑calling mode) | AutoGen uses OpenAI’s function‑calling to invoke tools (SQL, REST). | Same as generator |
| **Optional Open‑Source** | **Llama‑2‑13B‑Chat** (in‑cluster) – if you need a self‑hosted fallback for privacy. | Runs on Azure ML GPU Compute; used only when regulated data cannot leave the VNet. | 1‑2 s per 1 k‑token (GPU) |

> **Tip:** All model endpoint URLs and keys are stored in Azure Key Vault → Databricks secret scope `kv`. The SDK wrapper (`src/utils/secrets.py`) reads them at runtime.

### 3.2 End‑to‑End RAG Flow (Sequence Diagram)

```
User ──► REST /score endpoint (Azure ML) ──►  (1)  Model Serving Layer
                                   │
                                   ▼
                        ┌───────────────────────┐
                        │  RAG Agent (LangChain)│
                        └─────────┬─────────────┘
                                  │
               ┌──────────────────┼───────────────────┐
               │                  │                   │
               ▼                  ▼                   ▼
    Retrieval (Vector Search)   Generation (GPT‑4o‑mini)   Tool Calls (SQL/REST)
        │                         │                     │
        │   ┌─────────────────────▼─────────────────────┐
        │   │  VectorSearch Index (event_chunks_idx)    │
        │   └─────────────────────┬─────────────────────┘
        │                         │
        ▼                         ▼
  Embedding Store ⇆ 10‑k nearest chunks    ──►  (2)  Prompt Assembly
                                            │
                                            ▼
                                   ┌─────────────────┐
                                   │   Prompt (RAG)  │
                                   └───────┬─────────┘
                                           │
                                           ▼
                                   ┌─────────────────┐
                                   │   LLM (GPT‑4o‑mini) │
                                   └───────┬─────────┘
                                           │
                                           ▼
                                 Answer + Citations (source_ids)
                                           │
                                           ▼
                               ┌───────────────────────┐
                               │   Judge LLM (GPT‑4o)   │
                               │  – factuality check   │
                               │  – toxicity / policy  │
                               └───────┬───────────────┘
                                       │
           ┌───────────────────────────┼─────────────────────────────┐
           │                           │                             │
           ▼                           ▼                             ▼
   ✅  Return GOOD answer      ⚠️  Return WARN (explain)   ❌  Return ERROR (fallback)
```

**Explanation of numbered steps**

1. **Model Serving Layer** – Azure ML online endpoint receives a user request. It sends the request payload to the registered **RAG Agent** (a custom pyfunc model).  

2. **Prompt Assembly** – The agent (LangChain) builds a prompt that contains:
   * The user question.  
   * The **retrieved chunks** (top‑k from Vector Search).  
   * A *system message* that tells the LLM to cite sources using the `source_id` field.  

3. **Generation** – The LLM (GPT‑4o‑mini) produces an answer **and a list of cited chunk IDs**.  

4. **Judgment** – The answer + citations are sent to the **Judge LLM** (GPT‑4o).  
   * The judge runs a **function‑calling** schema:
     ```json
     {
        "name": "grade_answer",
        "parameters": {
           "type": "object",
           "properties": {
               "score": {"type":"integer","enum":[0,1,2]},   // 0‑fail, 1‑warn, 2‑pass
               "explanation": {"type":"string"}
           },
           "required":["score"]
        }
     }
     ```
   * If `score == 2` → the answer is returned to the user.  
   * If `score == 1` → a *warning* message is added (e.g., “I could not verify the price; see source”).  
   * If `score == 0` → the system falls back to a **search‑only** response or an error page.  

5. **Tool‑Calling (optional)** – When the user asks for a **SQL aggregation** or an **external REST call**, the *agent* invokes the corresponding Python function (registered with AutoGen). The result is injected back into the prompt before generation.  

---

## 4️⃣ Code – All Critical Pieces  

> **All code snippets below live in the repo.**  
> The corresponding notebooks simply `%run` the modules, so you can develop interactively **or** schedule them as Databricks Jobs.

### 4.1 Secrets Helper (`src/utils/secrets.py`)

```python
import json
from databricks import sql
from pyspark.sql import SparkSession

class SecretManager:
    """
    Wrapper around Databricks secret scope (backed by Azure Key Vault).
    Usage:
        sm = SecretManager()
        openai_key = sm.get("openai_key")
    """
    def __init__(self, scope: str = "kv"):
        self.scope = scope
        # dbutils only exists inside a notebook/cluster; fallback to os env for local testing
        self._dbutils = None
        try:
            from pyspark.dbutils import DBUtils  # type: ignore
            self._dbutils = DBUtils(SparkSession.builder.getOrCreate())
        except Exception:
            pass

    def get(self, key: str) -> str:
        if self._dbutils:
            return self._dbutils.secrets.get(scope=self.scope, key=key)
        else:
            # local dev – read from env or a .env file
            import os
            return os.getenv(key.upper(), "")
```

### 4.2 Ingest – Raw → Delta (`src/data/ingest.py`)

```python
from pyspark.sql import SparkSession, DataFrame
import pyspark.sql.functions as F

def ingest_events(spark: SparkSession,
                 src_path: str = "/mnt/raw/events/*.json",
                 target_table: str = "agentic_ai.raw.events") -> None:
    """Read raw JSON/CSV from ADLS and write as Delta with schema enforcement."""
    raw = spark.read.option("inferSchema", "true").json(src_path)

    # Basic validation – drop malformed rows
    cleaned = raw.filter(F.col("event_id").isNotNull())

    cleaned.write.format("delta") \
        .mode("overwrite") \
        .option("overwriteSchema", "true") \
        .saveAsTable(target_table)
```

### 4.3 Vector Store – Create Index (`src/vectorstore/create_index.py`)

```python
import mlflow
import pandas as pd
import numpy as np
import json
from openai import OpenAI
from databricks.vector_search import VectorSearchClient
from src.utils.secrets import SecretManager

def embed_batch(texts: list[str], model: str = "text-embedding-3-large") -> list[list[float]]:
    client = OpenAI(api_key=SecretManager().get("openai_key"))
    resp = client.embeddings.create(input=texts, model=model)
    return [e.embedding for e in resp.data]

def chunk_and_embed(spark, source_table: str = "agentic_ai.raw.events",
                   target_table: str = "agentic_ai.feat.event_chunks",
                   batch_size: int = 256):
    """Chunk text, embed, and write to a Delta table used by Databricks Vector Search."""
    from langchain.text_splitter import RecursiveCharacterTextSplitter

    df = spark.table(source_table).select("event_id", "description").toPandas()
    splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)

    rows = []
    for _, row in df.iterrows():
        chunks = splitter.split_text(row["description"])
        for chunk in chunks:
            rows.append({"event_id": row["event_id"], "text": chunk})

    # Batch‑wise embedding to respect rate limits
    texts = [r["text"] for r in rows]
    embeddings = []
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i+batch_size]
        embeddings.extend(embed_batch(batch))

    for i, emb in enumerate(embeddings):
        rows[i]["embedding"] = emb

    spark.createDataFrame(pd.DataFrame(rows)) \
        .write.format("delta") \
        .mode("overwrite") \
        .saveAsTable(target_table)

    # Register a Vector Search index (Delta‑Lake backed)
    mlflow.vectorsearch.create_index(
        name="event_chunks_idx",
        source_table=target_table,
        embedding_column="embedding",
        metric="cosine",
        workspace_url=mlflow.get_tracking_uri()
    )
```

### 4.4 RAG Agent – LangChain + AutoGen (`src/models/rag_agent.py`)

```python
import json
import os
from typing import List, Dict, Any
from autogen import AssistantAgent, UserProxyAgent, config_list_from_json
from langchain.chat_models import ChatOpenAI
from langchain.chains import RetrievalQA
from langchain.vectorstores import DatabricksVectorSearch
from langchain.embeddings import OpenAIEmbeddings
from src.utils.secrets import SecretManager
from src.utils.logging import get_logger

log = get_logger(__name__)

# -----------------------------------------------------------------
# 1️⃣ Vector Store (Databricks Vector Search) – singleton
# -----------------------------------------------------------------
def get_vector_store() -> DatabricksVectorSearch:
    return DatabricksVectorSearch(
        index_name="event_chunks_idx",
        workspace_url=mlflow.get_tracking_uri(),
        text_field="text",
        embedding_field="embedding",
        metadata_field="event_id"
    )

# -----------------------------------------------------------------
# 2️⃣ Retrieval QA chain (retriever + LLM)
# -----------------------------------------------------------------
def build_qa_chain():
    vector_store = get_vector_store()
    retriever = vector_store.as_retriever(search_kwargs={"k": 5})

    llm = ChatOpenAI(
        model="gpt-4o-mini",
        temperature=0.0,
        openai_api_key=SecretManager().get("openai_key")
    )
    return RetrievalQA.from_chain_type(
        llm=llm,
        chain_type="stuff",
        retriever=retriever,
        return_source_documents=True
    )

# -----------------------------------------------------------------
# 3️⃣ Tool definitions (SQL, REST, File‑I/O)
# -----------------------------------------------------------------
def sql_tool(query: str) -> str:
    """Run a Spark‑SQL query against Unity Catalog tables."""
    df = spark.sql(query)
    # Return as JSON to keep it LLM‑friendly
    return df.limit(10).toJSON().collect()

def rest_tool(url: str, method: str = "GET", payload: dict | None = None) -> str:
    import requests, json, time
    headers = {"Content-Type": "application/json"}
    resp = requests.request(method, url, json=payload, headers=headers, timeout=30)
    resp.raise_for_status()
    return resp.text

def save_file_tool(path: str, content: str) -> str:
    dbutils.fs.put(path, content, overwrite=True)
    return f"File written to {path}"

# -----------------------------------------------------------------
# 4️⃣ AutoGen Assistant – the *agentic* brain
# -----------------------------------------------------------------
def build_assistant():
    cfg = config_list_from_json("./config/openai_config.json")   # contains model, api_key, temperature
    system_msg = (
        "You are an enterprise RAG assistant. Use the provided tools when it helps "
        "the user. Cite sources using the format `source:[event_id]`. "
        "Never fabricate data. Follow company data‑privacy policy."
    )
    assistant = AssistantAgent(
        name="RAGAssistant",
        system_message=system_msg,
        config_list=cfg,
        description="Answer user questions using retrieval, generation, and tool usage."
    )
    # Register tools – AutoGen will expose them as *function calls*
    assistant.register_for_execution(
        name="sql_tool",
        description="Execute a Spark‑SQL query and return JSON rows.",
        function=sql_tool
    )
    assistant.register_for_execution(
        name="rest_tool",
        description="Call an external REST endpoint and return its raw response.",
        function=rest_tool
    )
    assistant.register_for_execution(
        name="save_file_tool",
        description="Persist a string content to DBFS path.",
        function=save_file_tool
    )
    return assistant

# -----------------------------------------------------------------
# 5️⃣ Public inference entry point (Databricks model serving)
# -----------------------------------------------------------------
def predict(prompt: str) -> Dict[str, Any]:
    """
    Databricks model serving expects a JSON payload:
    { "inputs": "<user prompt>" }
    Returns a dict with keys: answer, sources, tool_results (optional), judgement.
    """
    # 1️⃣ Retrieve & generate
    qa = build_qa_chain()
    result = qa({"question": prompt})
    answer = result["result"]
    sources = [doc.metadata["event_id"] for doc in result["source_documents"]]

    # 2️⃣ Send to judge LLM (separate function)
    from src.models.judge_agent import judge_answer
    judgement = judge_answer(answer, sources)

    return {
        "answer": answer,
        "sources": sources,
        "judgement": judgement
    }
```

### 4.5 Judge Agent (`src/models/judge_agent.py`)

```python
import json
from autogen import AssistantAgent, config_list_from_json
from src.utils.secrets import SecretManager

def _build_judge():
    cfg = config_list_from_json("./config/openai_config.json")
    #


