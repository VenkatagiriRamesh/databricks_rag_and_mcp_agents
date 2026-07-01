# Full Setup Guide — Databricks RAG & MCP Agents

Step-by-step instructions to stand up this project from scratch on **Databricks Free
Edition**. Follow the notebooks in order — each one is idempotent, so re-running a step after
fixing a mistake is safe.

## 1. Get a Databricks Free Edition workspace

1. Sign up at [databricks.com/learn/free-edition](https://www.databricks.com/learn/free-edition)
   (or log in if you already have one). Free Edition provisions a single workspace with Unity
   Catalog enabled by default — no account-console setup needed.
2. Confirm Unity Catalog is active: **Catalog** icon in the left sidebar should show at least
   one catalog (usually `workspace`).

## 2. Download the dataset

1. Go to [Medical Cost Personal Datasets](https://www.kaggle.com/datasets/mirichoi0218/insurance)
   on Kaggle (a free Kaggle account is required to download).
2. Download `insurance.csv` (1,338 rows, ~55 KB) to your local machine. No preprocessing needed.

## 3. Import the repo into your workspace

1. In Databricks, go to **Workspace** in the left sidebar.
2. Right-click your user folder → **Import**.
3. Choose **File** and upload each `.ipynb` from `schema_mgt/` and `src/` in this repo
   (or connect the whole GitHub repo via **Repos > Add Repo** if you've pushed it to GitHub).
4. Keep the same folder layout (`schema_mgt/`, `src/`) so the notebook prerequisite notes
   referencing sibling notebooks stay accurate.

## 4. Run `schema_mgt/00_setup_and_data_ingest`

1. Open the notebook and attach it to **Serverless** compute (Free Edition only offers
   serverless — there's no cluster picker needed).
2. Run the configuration cell. It creates the `insurance_rag_agent` catalog, the `raw` /
   `docs` / `agent_tools` schemas, and a Volume at
   `/Volumes/insurance_rag_agent/raw/source_data/`.
3. **Pause here** — go to **Catalog Explorer > insurance_rag_agent > raw > source_data
   (Volume) > Upload** and upload `insurance.csv` from step 2.
4. Run the rest of the notebook. The validation cell at the bottom should show 1,338 rows in
   `insurance_rag_agent.raw.policies_raw`.

## 5. Run `src/01_prepare_policy_documents`

Run top-to-bottom. No manual steps — it reads the raw table, buckets it, text-serializes it,
and writes `insurance_rag_agent.docs.policy_documents`. Validation cell prints 5 sample
`policy_text` rows and a total row count (should also be 1,338).

## 6. Run `src/02_vector_search_index`

1. Run top-to-bottom. This creates your **one** AI Search endpoint (Free Edition allows
   exactly one) and a Delta Sync Index over `policy_text`.
2. Index creation can take a few minutes the first time (embedding 1,338 short documents is
   fast, but endpoint provisioning has some latency). Re-run the last cell if the similarity
   search comes back empty — the index may still be syncing.
3. Confirm the endpoint in the UI: **Compute icon (left sidebar, if you don't see AI Search
   directly) > Vector Search**, or search "Vector Search" in the workspace search bar.

## 7. Verify available models before running `03`

1. Go to **Serving** in the left sidebar.
2. Confirm you can see Foundation Model API endpoints, typically including a chat model (e.g.
   `databricks-meta-llama-3-3-70b-instruct`) and an embedding model
   (`databricks-gte-large-en`, already used in step 6).
3. If the exact chat model name in `src/03_rag_chain_and_deploy`'s config cell isn't listed,
   swap `CHAT_MODEL_ENDPOINT` for whichever chat model your workspace shows — the rest of the
   notebook doesn't need to change.

## 8. Run `src/03_rag_chain_and_deploy`

1. Run top-to-bottom. The local smoke-test cell calls the retriever + chat model directly and
   prints an answer — confirm this looks reasonable before continuing.
2. The logging/registration cell registers `insurance_rag_agent.agent_tools.insurance_rag_model`
   in Unity Catalog.
3. The deploy cell calls `agents.deploy(...)`, which provisions a real serving endpoint named
   `insurance_rag_endpoint`. **This can take 10-15 minutes** the first time (endpoint cold
   start). Track progress in **Serving > insurance_rag_endpoint**.
4. Once the endpoint shows **Ready**, the final validation cell queries it live.

> **Free Edition quota:** model serving is quota-limited per account. If you hit a quota error,
> delete unused serving endpoints (**Serving > select endpoint > Delete**) before retrying.

## 9. Run `src/04_agent_tools_mcp_functions`

Run top-to-bottom. This registers the 3 UC SQL functions and prints the managed MCP endpoint
URL, e.g.:

```
https://<your-workspace-host>/api/2.0/mcp/functions/insurance_rag_agent/agent_tools
```

The validation cell calls `classify_customer` and `generate_analysis_report` directly via SQL
— confirm both return sensible text before moving on.

## 10. Run `src/05_mcp_tool_calling_agent`

1. Run top-to-bottom. No extra credentials are needed inside a Databricks notebook — the
   `WorkspaceClient()` and `DatabricksMCPClient` both pick up the notebook's own auth context
   automatically.
2. Watch the printed transcript: the model should call `ask_insurance_rag` for the first
   prompt, `generate_analysis_report` for the second, and `classify_customer` for the third.
3. If a different tool is called than expected, that's a model reasoning choice, not a bug —
   try rephrasing the prompt to be more explicit (e.g. "generate a report" vs. "tell me about").

## 11. Clean up (optional, recommended on Free Edition)

Free Edition quotas are account-wide and shared across everything you build. When you're done
demoing:

1. **Serving > insurance_rag_endpoint > Delete** — stops the deployed RAG endpoint.
2. **Vector Search > your endpoint > Delete index / Delete endpoint** if you won't need it
   again soon (recreating it later just means re-running `02`).
3. Catalog/tables can stay — they cost storage, not compute, and are cheap to keep.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `CREATE VOLUME` fails | Catalog/schema not created yet | Re-run the config cell in notebook `00` first |
| Similarity search returns nothing | Index still syncing | Wait a few minutes, re-run the validation cell |
| `agents.deploy()` hangs / errors | Free Edition model-serving quota reached | Delete other serving endpoints, retry |
| `ask_insurance_rag` returns an error string | RAG endpoint not `Ready` yet, or endpoint name mismatch | Check **Serving** status; confirm `RAG_ENDPOINT_NAME` in `03` matches what `04`'s function calls |
| MCP client can't list tools | Wrong workspace host, or `agent_tools` schema empty | Re-check `mcp_server_url` printed in notebook `04`; confirm the 3 functions exist via `SHOW FUNCTIONS IN insurance_rag_agent.agent_tools` |
