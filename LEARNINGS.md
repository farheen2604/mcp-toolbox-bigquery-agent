# 📚 Learnings — MCP Toolbox for Databases: BigQuery

Track 2 Lab 2 — Key concepts, issues faced, and fixes applied.

---

## What I Learned

### 1. What is MCP Toolbox for Databases?
MCP Toolbox is an open source MCP server for databases. It sits between your AI agent and your database, exposing database tools via the MCP protocol. Any MCP-capable client (ADK, IDEs, tools) can connect to it.

Key benefits:
- Connection pooling handled automatically
- Centralized tool configuration via tools.yaml
- Share tools across multiple agents without rewriting code
- Built-in OpenTelemetry support for observability
- Works with BigQuery, AlloyDB, and more

### 2. How tools.yaml works
The tools.yaml file has 3 sections:

**Sources** — define your database connection:
- kind: bigquery
- project: your GCP project ID

**Tools** — define actions the agent can take:
- kind: bigquery-sql
- source: references the source above
- statement: the SQL query to run
- description: tells the LLM when to use this tool

**Toolsets** — groups of tools loaded together:
- Useful when you have multiple agents needing different tool combinations

### 3. Architecture — How it all fits together
```
User Query (natural language)
        │
        ▼
ADK Agent (Gemini 2.5 Flash via Vertex AI)
        │
        ▼ calls
MCP Toolbox Server (localhost:5000)
        │
        ▼ runs SQL
BigQuery Public Dataset
(bigquery-public-data.google_cloud_release_notes.release_notes)
        │
        ▼
Formatted response back to user
```

### 4. Difference — Remote MCP vs Self-hosted MCP Toolbox

| | Remote MCP (Lab 1) | MCP Toolbox (Lab 2) |
|---|---|---|
| Who runs it | Google manages it | You run it locally |
| URL | https://bigquery.googleapis.com/mcp | http://127.0.0.1:5000 |
| Config | Code in tools.py | Declarative tools.yaml |
| Flexibility | Fixed Google tools | You define your own SQL |
| Use case | Quick integration | Custom queries on your data |

### 5. Two terminals are always needed
- Tab 1 → MCP Toolbox server running (`./toolbox --tools-file tools.yaml`)
- Tab 2 → ADK agent running (`adk run gcp_releasenotes_agent_app/`)
Both must be running simultaneously for the agent to work.

### 6. BigQuery Public Datasets
Google Cloud Public Dataset Program makes real datasets freely queryable.
The Release Notes dataset is updated regularly and mirrors the official
Google Cloud Release Notes webpage — queryable with standard SQL.

---

## Issues Faced and Fixes

### Issue 1 — tools.yaml project: null

**Error:**
```
unable to parse source "my-bq-source" as "bigquery":
Field validation for 'Project' failed on the 'required' tag
project: null
```

**Why it occurred:**
The setup script did not populate the project ID in tools.yaml automatically.
The placeholder YOUR_PROJECT_ID was either missed or set to null.

**Fix:**
```bash
PROJECT_ID=$(gcloud config get-value project)
sed -i "s/project: null/project: $PROJECT_ID/" ~/mcp-toolbox/tools.yaml
# OR manually edit:
nano ~/mcp-toolbox/tools.yaml
# Change: project: null
# To:     project: your-actual-project-id
```

**How to avoid next time:**
Always verify tools.yaml after creation:
```bash
cat tools.yaml | grep project
```
Must show your actual project ID, not null or YOUR_PROJECT_ID.

---

### Issue 2 — Vertex AI API not enabled

**Error:**
```
403 PERMISSION_DENIED: Vertex AI API has not been used in project
mcp-toolbox-db-491509 before or it is disabled.
```

**Why it occurred:**
This was a new GCP project created specifically for this lab.
Vertex AI API is not enabled by default on new projects.

**Fix:**
```bash
gcloud services enable aiplatform.googleapis.com \
  --project=mcp-toolbox-db-491509
```
Wait 1-2 minutes then retry.

**How to avoid next time:**
At the start of every new lab with a new project, run:
```bash
gcloud services enable \
  aiplatform.googleapis.com \
  bigquery.googleapis.com \
  --project=$(gcloud config get-value project)
```

---

### Issue 3 — gemini-2.0-flash model not found

**Error:**
```
404 NOT_FOUND: Publisher Model gemini-2.0-flash was not found
or your project does not have access to it.
```

**Why it occurred:**
The codelab used gemini-2.0-flash but this model is not available
in all regions or all projects. The Vertex AI model availability
depends on your project's region and access tier.

**Fix:**
```bash
sed -i 's/gemini-2.0-flash/gemini-2.5-flash/' \
  ~/mcp-toolbox/my-agents/gcp_releasenotes_agent_app/agent.py
```

**How to avoid next time:**
Always use gemini-2.5-flash as default — it is available across
all standard Vertex AI projects and regions. Only switch to other
models if explicitly required by the lab.

---

### Issue 4 — Wrong directory for agent.py

**Error:**
```
-bash: /home/.../my-agents/gcp_releasenotes_agent_app/agent.py:
No such file or directory
```

**Why it occurred:**
The my-agents folder was created inside mcp-toolbox directory,
not in the home directory. The path used assumed ~/my-agents
but the actual path was ~/mcp-toolbox/my-agents.

**Fix:**
```bash
# Always check where you are first
pwd

# Use the correct full path
cat > ~/mcp-toolbox/my-agents/gcp_releasenotes_agent_app/agent.py << 'EOF'
...
EOF
```

**How to avoid next time:**
Always run pwd before creating files with cat > path.
Use tab autocomplete to confirm the path exists.

---

## Pre-Lab Checklist for MCP Toolbox Labs
```bash
# 1. Set and confirm project
gcloud config set project YOUR_PROJECT_ID
gcloud config get-value project

# 2. Enable required APIs
gcloud services enable \
  aiplatform.googleapis.com \
  bigquery.googleapis.com

# 3. Verify tools.yaml has correct project ID
cat tools.yaml | grep project
# Must NOT be null or YOUR_PROJECT_ID

# 4. Start MCP Toolbox in Tab 1
cd ~/mcp-toolbox
./toolbox --tools-file "tools.yaml"

# 5. Start Agent in Tab 2
cd ~/mcp-toolbox/my-agents
source .venv/bin/activate
adk run gcp_releasenotes_agent_app/

# 6. Always use gemini-2.5-flash as default model
