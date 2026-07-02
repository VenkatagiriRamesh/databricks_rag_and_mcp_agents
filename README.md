<p align="center">
  <img src="docs/architecture_diagram.png" alt="RAG & MCP Agents Architecture" width="700"/>
</p>

<h1 align="center">Databricks RAG & MCP Agents</h1>

<p align="center">
  <em>A RAG chatbot and a tool-calling MCP agent, both deployed as real Mosaic AI Model
  Serving endpoints on Databricks Free Edition</em>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Databricks-Free_Edition-FF3621?style=for-the-badge&logo=databricks&logoColor=white" alt="Databricks Free Edition"/>
  <img src="https://img.shields.io/badge/Delta_Lake-3.x-00ADD8?style=for-the-badge&logo=delta&logoColor=white" alt="Delta Lake"/>
  <img src="https://img.shields.io/badge/PySpark-3.5-E25A1C?style=for-the-badge&logo=apache-spark&logoColor=white" alt="PySpark"/>
  <img src="https://img.shields.io/badge/Mosaic_AI-RAG_%2B_MCP-6A0DAD?style=for-the-badge" alt="Mosaic AI RAG + MCP"/>
</p>

---

## What this is

I wanted a project that actually used Databricks' RAG and agent stack end to end, not just a
notebook that calls an LLM once and prints an answer. So this one takes a small, plain
insurance dataset — nothing exotic, just 1,338 rows of age/BMI/region/charges — and turns it
into two things you can genuinely chat with inside Databricks:

- A **RAG chatbot** that answers questions about the policy data, backed by a Vector Search
  index and deployed as its own Model Serving endpoint.
- An **MCP agent** that sits on top of a toolbox of Unity Catalog functions — the RAG
  answerer, a customer classifier, and seven different report generators — and figures out
  which tool to use, or asks you what you want first, based on what you type.

Both of those are real, deployed services. You open Databricks' Playground, pick the
endpoint, and talk to it. No extra glue code needed on your end.

## The dataset

[Medical Cost Personal Datasets](https://www.kaggle.com/datasets/mirichoi0218/insurance) on
Kaggle — age, sex, BMI, children, smoker status, region, and charges for 1,338 people. It's a
classic, tiny, entirely structured dataset, which is honestly the whole reason it's
interesting here: RAG assumes you have documents to retrieve, and this dataset doesn't have
any. So the pipeline converts every row into a short paragraph ("a 52-year-old male from the
southeast, a smoker, BMI 34.1, $39,611.75 a year") before it ever touches a vector index.
That's the trick that makes a spreadsheet answerable in natural language.

## How it fits together

Everything lives under one Unity Catalog catalog, `insurance_rag_agent`, split into three
schemas: `raw` for the ingested table, `docs` for the text-serialized rows plus the Vector
Search index built on top of them, and `agent_tools` for everything the agent can call.

The RAG side is straightforward: retrieve the closest policy write-ups from the vector index,
hand them to a Foundation Model API chat model as context, and that whole chain gets logged
with MLflow and deployed as `insurance_rag_endpoint`.

The agent side is where it gets more interesting. Rather than one fixed `generate_report()`
function with a pile of optional filters, `agent_tools` holds ten separate SQL functions —
the RAG answerer, a `list_report_types` menu, seven distinct report generators (regional
cost, smoker comparison, age brackets, family size, BMI risk, a high-cost-segment deep dive,
and gender cost patterns), and the applicant classifier. Because they're all plain Unity
Catalog functions living in one schema, Databricks' managed MCP server exposes every one of
them at a single URL automatically — there's no custom MCP server process to run or maintain.

The agent is told, through its system prompt, not to guess which report someone wants. Ask
for "a report" without saying which kind, and it calls `list_report_types`, reads you the
menu, and waits for you to pick before it builds anything. That was a deliberate choice —
Free Edition allows up to 20 tools per agent, and I'd rather spend that budget on a handful
of small, well-scoped tools than one overloaded function trying to do everything.

## Chatting with it

Once `03` and `05` have deployed their endpoints, go to **Serving** in the sidebar:

- Open `insurance_rag_endpoint` and click **Open in Playground** to ask it anything about the
  policy data.
- Open `insurance_mcp_agent_endpoint` and do the same to talk to the full agent — ask a
  question, ask for "a report" and watch it offer you the menu, or describe a hypothetical
  applicant and have it classify them.

If you'd rather not deploy anything and just want to poke at the tools directly, Playground
can also attach the MCP server to any tools-enabled model on its own: **Tools → + Add tool →
MCP → Managed MCP servers**, then point it at the `insurance_rag_agent` catalog and
`agent_tools` schema. It'll pick up all ten tools by itself.

## Running it yourself

Full click-by-click instructions live in [docs/SETUP.md](docs/SETUP.md). Short version: get a
Databricks Free Edition workspace (Unity Catalog is on by default, nothing to configure),
download `insurance.csv` from Kaggle, import the notebooks below, and run them in order.

```
databricks-rag-and-mcp-agents/
├── schema_mgt/
│   └── 00_setup_and_data_ingest.ipynb
├── src/
│   ├── 01_prepare_policy_documents.ipynb
│   ├── 02_vector_search_index.ipynb
│   ├── 03_rag_chain_and_deploy.ipynb
│   ├── 04_agent_tools_mcp_functions.ipynb
│   └── 05_mcp_agent_deploy.ipynb
├── docs/
│   ├── architecture_diagram.png
│   └── SETUP.md
├── .gitignore
└── README.md
```

| Step | Notebook | What it does |
|:----:|----------|---------------|
| 0 | `schema_mgt/00_setup_and_data_ingest` | Creates the catalog/schemas/volume, lands the raw table |
| 1 | `src/01_prepare_policy_documents` | Buckets the data and writes the RAG documents |
| 2 | `src/02_vector_search_index` | Builds the AI Search endpoint and index |
| 3 | `src/03_rag_chain_and_deploy` | Builds, logs, and deploys the RAG chain |
| 4 | `src/04_agent_tools_mcp_functions` | Registers all ten UC functions |
| 5 | `src/05_mcp_agent_deploy` | Builds, logs, and deploys the MCP agent |

## Worth knowing about Free Edition

- You get exactly one AI Search endpoint, and it only supports Delta Sync indexes (no Direct
  Vector Access) — the notebooks are written around that limit.
- Everything runs on pay-per-token Foundation Model APIs and CPU serverless serving — no
  GPUs, no provisioned throughput, since Free Edition doesn't offer either.
- This project ends up running three serving endpoints at once (the RAG chain, the MCP agent,
  and Vector Search), which is a fair chunk of Free Edition's quota. If you hit a limit,
  delete whichever endpoint you're not actively using.

## If I kept going

A few things I'd add next: a Databricks App for a fully custom chat UI instead of leaning on
Playground, a trained classifier to compare against the zero-shot `ai_classify()` one (the
`charges_tier` column already gives you ground truth to check it against), and an MLflow
evaluation pass to score the RAG answers against a labeled question set instead of just
eyeballing them.

## Credit

Dataset: [Medical Cost Personal Datasets](https://www.kaggle.com/datasets/mirichoi0218/insurance)
on Kaggle, by `mirichoi0218`. Built on Databricks Free Edition, Unity Catalog, Mosaic AI
Vector Search, the Agent Framework, and Databricks-managed MCP servers.
