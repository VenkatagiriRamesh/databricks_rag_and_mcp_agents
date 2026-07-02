# Full Setup Guide — Databricks RAG & MCP Agents

Click-by-click instructions to stand this project up from a blank Databricks Free Edition
workspace. Follow the notebooks in order — each one is idempotent, so if something goes
wrong partway through, just fix it and re-run from that notebook. You won't end up with
duplicate tables or half-created endpoints.

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

## 7. Verify the chat model name before running `03`

1. Go to **Serving** in the left sidebar.
2. Confirm you can see Foundation Model API endpoints, typically including a chat model (e.g.
   `databricks-meta-llama-3-3-70b-instruct`) and an embedding model
   (`databricks-gte-large-en`, already used in step 6).
3. `CHAT_MODEL_ENDPOINT` in `src/03_rag_chain_and_deploy` and `src/05_mcp_agent_deploy` is set
   to `databricks-meta-llama-3-3-70b-instruct` — the classic, workspace-wide Foundation Model
   name, queried via `w.serving_endpoints.get_open_ai_client()`. This works even on workspaces
   whose Serving page shows a "moved to AI Gateway" banner. Some workspaces' **"Use this
   model"** panel instead shows a personal-looking alias like
   `<catalog>.<schema>.<model>_backend` with a `base_url` under `/ai-gateway/mlflow/v1` —
   **don't use that one**. That alias is scoped to whichever identity triggered its creation
   and isn't reliably reachable from the *deployed model's* own service principal, which
   causes deployment to fail outright with `NOT_FOUND: Dependent serving endpoint ... does
   not exist.` if declared as a resource, or a runtime 404 if called manually.
4. If `databricks-meta-llama-3-3-70b-instruct` genuinely isn't available in your workspace,
   swap `CHAT_MODEL_ENDPOINT` for whichever classic Foundation Model name your Serving page
   lists — just confirm it's a real entry on that page, not a generated alias from a "use this
   model" code snippet.

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

Run top-to-bottom. This registers all ten UC SQL functions — the RAG answerer, the
`list_report_types` menu, the seven report generators, and the classifier — and prints the
managed MCP endpoint URL, e.g.:

```
https://<your-workspace-host>/api/2.0/mcp/functions/insurance_rag_agent/agent_tools
```

The validation cell calls `list_report_types`, `classify_customer`, and
`generate_smoker_comparison_report` directly via SQL — confirm all three return sensible text
before moving on.

## 10. Run `src/05_mcp_agent_deploy`

1. Run top-to-bottom. The local smoke-test cell instantiates the agent and runs 4 demo
   prompts directly in the notebook — confirm the model calls `ask_insurance_rag` for the
   first prompt, `list_report_types` for the second (it should read back the numbered menu
   rather than just generating something), `generate_smoker_comparison_report` for the third,
   and `classify_customer` for the fourth. A different tool getting called is a model
   reasoning choice, not a bug — try rephrasing the prompt to be more explicit if it happens.
2. The logging/registration cell resolves the MCP server's underlying resources via
   `DatabricksMCPClient().get_databricks_resources(...)` and declares them (plus the chat
   model) as `resources`, then registers `insurance_rag_agent.agent_tools.insurance_mcp_agent`
   in Unity Catalog.
3. The deploy cell calls `agents.deploy(...)`, provisioning a real serving endpoint named
   `insurance_mcp_agent_endpoint`. **This can take 10-15 minutes** the first time. Track
   progress in **Serving > insurance_mcp_agent_endpoint**.
4. Once the endpoint shows **Ready**, the final validation cell re-runs the same 4 prompts
   against the *deployed* service (not the local instance) to prove it works end-to-end.
5. **Try it in Playground:** go to **Serving > insurance_mcp_agent_endpoint > Open in
   Playground** and chat with the agent directly. Ask it something like "I want a report" and
   watch it come back with the numbered menu instead of just picking one for you — that's the
   behavior the system prompt is enforcing.
6. **Bonus, no deployment needed:** in **Playground**, pick any tools-enabled model, then
   **Tools > + Add tool > MCP > Managed MCP servers**, and enter the `insurance_rag_agent`
   catalog / `agent_tools` schema — Playground auto-discovers all ten UC functions as tools
   you can call directly, without the agent endpoint at all.

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
| `NOT_FOUND: Dependent serving endpoint ... does not exist` (deployment itself fails) | `CHAT_MODEL_ENDPOINT` is set to a personal-looking AI-Gateway alias instead of the classic Foundation Model name | Set `CHAT_MODEL_ENDPOINT = 'databricks-meta-llama-3-3-70b-instruct'` (see step 7) — `DatabricksServingEndpoint` in `resources=` can only resolve real classic serving endpoints |
| `openai.NotFoundError: '<model>' does not exist` at runtime (deployment succeeds, but the deployed model errors when queried) | Same cause as above, caught earlier by `resources=` validation in current code — if you see this, `CHAT_MODEL_ENDPOINT` was reverted to an alias somewhere | Re-check `CHAT_MODEL_ENDPOINT` matches step 7 exactly, re-log and redeploy |
| `ask_insurance_rag` returns an error string | RAG endpoint not `Ready` yet, or endpoint name mismatch | Check **Serving** status; confirm `RAG_ENDPOINT_NAME` in `03` matches what `04`'s function calls |
| MCP client can't list tools | Wrong workspace host, or `agent_tools` schema empty | Re-check `mcp_server_url` printed in notebook `04`; confirm all ten functions exist via `SHOW FUNCTIONS IN insurance_rag_agent.agent_tools` |
| Agent picks a report without asking, or picks the wrong one | System prompt not being followed, or the prompt wasn't ambiguous enough to trigger the menu | Try a more open-ended prompt ("I need a report" rather than naming a topic); if it's still skipping the menu, double-check `SYSTEM_PROMPT` made it into the deployed model (redeploy `05` after any change) |
