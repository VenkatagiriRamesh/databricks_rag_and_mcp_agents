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
3. Click into the chat model and open its **"Use this model"** / query-example panel and copy
   its exact `model=` and `base_url=` values — they vary by workspace. Two patterns we've seen:
   - **AI Gateway / Unity-Catalog-governed** (this project was verified on a workspace like
     this): `model="insurance_rag_agent.agent_tools.meta-llama-3-3-70b-instruct_backend"`,
     `base_url="https://<workspace-host>/ai-gateway/mlflow/v1"`.
   - **Classic Foundation Model API**: `model="databricks-meta-llama-3-3-70b-instruct"`,
     `base_url="https://<workspace-host>/serving-endpoints"`.
4. In `src/03_rag_chain_and_deploy` and `src/05_mcp_agent_deploy`, set `CHAT_MODEL_ENDPOINT`
   to the `model=` value and `MODEL_QUERY_BASE_PATH` to whatever comes after the host in the
   `base_url=` value (e.g. `/ai-gateway/mlflow/v1` or `/serving-endpoints`) from your workspace's
   snippet — both notebooks build the OpenAI client from these two constants.

## 8. Run `src/03_rag_chain_and_deploy`

1. Run top-to-bottom. The local smoke-test cell calls the retriever + chat model directly and
   prints an answer — confirm this looks reasonable before continuing.
2. The logging/registration cell declares the vector index and chat model as `resources` and
   registers `insurance_rag_agent.agent_tools.insurance_rag_model` in Unity Catalog — this
   auto-provisions scoped credentials for the deployed agent, so there's no manual token step.
3. The deploy cell calls `agents.deploy(...)`, which provisions a real serving endpoint named
   `insurance_rag_endpoint`. **This can take 10-15 minutes** the first time (endpoint cold
   start). Track progress in **Serving > insurance_rag_endpoint**.
4. Once the endpoint shows **Ready**, the final validation cell queries it live.
5. **Try it in Playground:** go to **Serving > insurance_rag_endpoint > Open in Playground**
   and chat with it directly — no extra code needed.

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

## 10. Run `src/05_mcp_agent_deploy`

1. Run top-to-bottom. The local smoke-test cell instantiates the agent and runs the 3 demo
   prompts directly in the notebook — confirm the model calls `ask_insurance_rag` for the
   first prompt, `generate_analysis_report` for the second, and `classify_customer` for the
   third (a different tool getting called is a model reasoning choice, not a bug — try
   rephrasing the prompt to be more explicit, e.g. "generate a report" vs. "tell me about").
2. The logging/registration cell resolves the MCP server's underlying resources via
   `DatabricksMCPClient().get_databricks_resources(...)` and declares them (plus the chat
   model) as `resources`, then registers `insurance_rag_agent.agent_tools.insurance_mcp_agent`
   in Unity Catalog.
3. The deploy cell calls `agents.deploy(...)`, provisioning a real serving endpoint named
   `insurance_mcp_agent_endpoint`. **This can take 10-15 minutes** the first time. Track
   progress in **Serving > insurance_mcp_agent_endpoint**.
4. Once the endpoint shows **Ready**, the final validation cell re-runs the same 3 prompts
   against the *deployed* service (not the local instance) to prove it works end-to-end.
5. **Try it in Playground:** go to **Serving > insurance_mcp_agent_endpoint > Open in
   Playground** and chat with the agent directly — ask it a question, ask for a report, or
   describe an applicant to classify.
6. **Bonus, no deployment needed:** in **Playground**, pick any tools-enabled model, then
   **Tools > + Add tool > MCP > Managed MCP servers**, and enter the `insurance_rag_agent`
   catalog / `agent_tools` schema — Playground auto-discovers the 3 UC functions as tools you
   can call directly, without the agent endpoint at all.

> **Free Edition quota:** this project now runs 3 endpoints at once (`insurance_rag_endpoint`,
> `insurance_mcp_agent_endpoint`, and the Vector Search endpoint). If deploying the agent hits
> a quota error, delete an unused serving endpoint first, then retry.

## 11. Clean up (optional, recommended on Free Edition)

Free Edition quotas are account-wide and shared across everything you build. When you're done
demoing:

1. **Serving > insurance_rag_endpoint > Delete** — stops the deployed RAG endpoint.
2. **Serving > insurance_mcp_agent_endpoint > Delete** — stops the deployed MCP agent endpoint.
3. **Vector Search > your endpoint > Delete index / Delete endpoint** if you won't need it
   again soon (recreating it later just means re-running `02`).
4. Catalog/tables can stay — they cost storage, not compute, and are cheap to keep.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `CREATE VOLUME` fails | Catalog/schema not created yet | Re-run the config cell in notebook `00` first |
| Similarity search returns nothing | Index still syncing | Wait a few minutes, re-run the validation cell |
| `agents.deploy()` hangs / errors, or `InvalidParameterValue: Scale to zero must be enabled` | Free Edition requires `scale_to_zero=True`, or model-serving quota reached | Confirm `scale_to_zero=True` is set in the deploy cell; if it's a quota error, delete an unused serving endpoint (`insurance_rag_endpoint`, `insurance_mcp_agent_endpoint`, or the Vector Search endpoint) and retry |
| `NameError: name 'deployment_info' is not defined` | Notebook session restarted/reattached after a successful deploy, wiping in-memory variables | You don't need to redeploy — just re-run the config cell, then the final validation cell, which key off the static `RAG_ENDPOINT_NAME` / `AGENT_ENDPOINT_NAME` constants instead |
| `OpenAIError: Missing credentials` | `w.config.token` is empty under this workspace's OAuth auth | Already handled in these notebooks via `w.config.authenticate()` / `resources=` — if you see this in your own code, use one of those instead of `.config.token` |
| `ask_insurance_rag` returns an error string | RAG endpoint not `Ready` yet, or endpoint name mismatch | Check **Serving** status; confirm `RAG_ENDPOINT_NAME` in `03` matches what `04`'s function calls |
| MCP client can't list tools | Wrong workspace host, or `agent_tools` schema empty | Re-check `mcp_server_url` printed in notebook `04`; confirm the 3 functions exist via `SHOW FUNCTIONS IN insurance_rag_agent.agent_tools` |
