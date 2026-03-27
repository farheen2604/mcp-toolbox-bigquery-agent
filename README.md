# 📰 GCP Release Notes Agent

An AI agent built with **Google ADK** and **Gemini 2.5 Flash** that uses
**MCP Toolbox for Databases** to query the Google Cloud Release Notes
BigQuery public dataset and answer natural language questions about
recent Google Cloud updates.

---

## 🤖 What It Does

Ask the agent any question about recent Google Cloud release notes and it
will query BigQuery in real time and return a formatted response.

**Example queries:**
- "Get me the Google Cloud release notes"
- "What are the latest Vertex AI updates?"
- "Any new Compute Engine features this week?"

---

## 🏗️ Architecture
```
User Query (natural language)
        │
        ▼
ADK Agent — gcp_releasenotes_agent (Gemini 2.5 Flash)
        │
        ▼ MCP protocol
MCP Toolbox Server (localhost:5000)
        │  tool: search_release_notes_bq
        ▼ SQL query
BigQuery Public Dataset
bigquery-public-data.google_cloud_release_notes.release_notes
        │
        ▼
Formatted release notes response
```

---

## 📁 Project Structure
```
mcp-toolbox/
├── toolbox                          # MCP Toolbox binary
├── tools.yaml                       # MCP Toolbox configuration
└── my-agents/
    ├── README.md                    # This file
    ├── LEARNINGS.md                 # Issues faced and fixes
    ├── .gitignore
    └── gcp_releasenotes_agent_app/
        ├── agent.py                 # Agent definition
        ├── __init__.py              # Package marker
        └── .env                    # Project config (not committed)
```

---

## 🚀 Setup & Run

### Prerequisites
- Google Cloud project with billing enabled
- Cloud Shell
- Vertex AI API enabled
- BigQuery API enabled

### Step 1 — Enable APIs
```bash
gcloud services enable \
  aiplatform.googleapis.com \
  bigquery.googleapis.com
```

### Step 2 — Download MCP Toolbox
```bash
mkdir mcp-toolbox && cd mcp-toolbox
export VERSION=0.23.0
curl -O https://storage.googleapis.com/genai-toolbox/v$VERSION/linux/amd64/toolbox
chmod +x toolbox
```

### Step 3 — Configure tools.yaml
```bash
PROJECT_ID=$(gcloud config get-value project)

cat > tools.yaml << YAML
sources:
  my-bq-source:
    kind: bigquery
    project: $PROJECT_ID

tools:
  search_release_notes_bq:
    kind: bigquery-sql
    source: my-bq-source
    statement: |
      SELECT
        product_name, description, published_at
      FROM
        \`bigquery-public-data\`.\`google_cloud_release_notes\`.\`release_notes\`
      WHERE
        DATE(published_at) >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
      GROUP BY product_name, description, published_at
      ORDER BY published_at DESC
    description: |
      Use this tool to get information on Google Cloud Release Notes.

toolsets:
  my_bq_toolset:
    - search_release_notes_bq
YAML
```

### Step 4 — Start MCP Toolbox (Tab 1)
```bash
./toolbox --tools-file "tools.yaml"
```

Server starts on port 5000. You should see:
```
INFO "Server ready to serve!"
```

### Step 5 — Set up Agent (Tab 2)
```bash
mkdir my-agents && cd my-agents
python -m venv .venv
source .venv/bin/activate
pip install google-adk toolbox-core

adk create gcp_releasenotes_agent_app
# Select: gemini-2.5-flash, Vertex AI, default project and region
```

### Step 6 — Update agent.py
```python
from google.adk.agents import Agent
from toolbox_core import ToolboxSyncClient

toolbox = ToolboxSyncClient("http://127.0.0.1:5000")
tools = toolbox.load_toolset('my_bq_toolset')

root_agent = Agent(
    name="gcp_releasenotes_agent",
    model="gemini-2.5-flash",
    description="Agent to answer questions about Google Cloud Release notes.",
    instruction="You are a helpful agent who can answer user questions about the Google Cloud Release notes. Use the tools to answer the question",
    tools=tools,
)
```

### Step 7 — Run the Agent
```bash
adk run gcp_releasenotes_agent_app/
```

Type your query:
```
get me the google cloud release notes
```

---

## 🧪 Sample Output
```
[user]: get me the google cloud release notes

[gcp_releasenotes_agent]: Here are the recent Google Cloud Release Notes:

Vertex AI: New Gemini 2.5 Flash model now available in us-central1.
Published: 2026-03-27

Compute Engine: Dynamic NICs now generally available.
Published: 2026-03-26

Cloud Run: Support for GPU workloads now in Preview.
Published: 2026-03-25
...
```

---

## 🛠️ Tech Stack

| Component | Technology |
|---|---|
| Agent Framework | Google ADK |
| LLM | Gemini 2.5 Flash via Vertex AI |
| MCP Server | MCP Toolbox for Databases v0.23.0 |
| Database | BigQuery Public Dataset |
| Dataset | google_cloud_release_notes.release_notes |
| Language | Python 3.12 |
| Environment | Google Cloud Shell |

---

## ⚠️ Known Issues & Fixes

See [LEARNINGS.md](./LEARNINGS.md) for detailed fixes for:
- tools.yaml project: null error
- Vertex AI API not enabled
- gemini-2.0-flash model not found
- Wrong directory path for agent files

---

## 📚 References

- [MCP Toolbox for Databases](https://github.com/googleapis/genai-toolbox)
- [BigQuery Public Datasets](https://cloud.google.com/bigquery/public-data)
- [Google Cloud Release Notes](https://cloud.google.com/release-notes)
- [Google ADK Documentation](https://google.github.io/adk-docs/)
- [Codelab: MCP Toolbox for Databases BigQuery](https://codelabs.developers.google.com/mcp-toolbox-bigquery-dataset)
