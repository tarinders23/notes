# Oncall AI Assist - Implementation Deep Dive

## 1. Problem Statement
On-call engineers were spending significant time ("Toil") on initial triage: translating PagerDuty alerts to service logs, finding relevant runbooks, and checking for recent deployments. This manual context-gathering increased Mean Time To Resolution (MTTR).

**Goal:** Automate the "Context Gathering" and "Initial Hypothesis" phases of incident response.

## Current Infrastructure
1. We already have a service to which listens on slack commands and filter message in a channel and send to AWS Lambda for processing (SQS queue).
2. AWS Lambda can return responses back to slack channel (via SQS queue).


## 2. Solution Architecture
The system is built as an **Event-Driven AI Agent** that acts as a "Virtual SRE".

### High-Level Workflow
1.  **Trigger:** PagerDuty webhook -> Slack Channel -> bot service (existing) -> SQS Queue -> AWS Lambda.
2.  **Orchestration:** The Slack Bot sends the alert payload to a **Python/FastAPI Backend (AWS Lambda)**.
3.  **Agent Execution:** The backend initializes a **LangChain** agent.
4.  **Tool Execution (MCP):** The agent uses the **Model Context Protocol (MCP)** to securely query infrastructure tools.
5.  **RAG Layer:** Retrieves historical context (past incidents, runbooks).
6.  **Output:** Posts a structured "Incident Analysis One-Pager" to the Slack thread.

## 3. Tech Stack
*   **Language:** Python 3.12+
*   **LLM Framework:** **LangChain** (for stateful multi-step reasoning).
*   **Tool Interface:** **Model Context Protocol (MCP)** (Standardizing how the LLM talks to Datadog/Git/Jira).
*   **Vector DB:** **FAISS** (In-memory only).
*   **LLM:** GPT-4o (for complex reasoning) and GPT-3.5-Turbo (for summarization).
*   **Integration:** Slack Bolt SDK, PagerDuty Webhooks.
*   **Infrastructure:** AWS Lambda (Event listener).

## 4. Implementation Details

### A. The "Brain" (LangChain Agent)
Instead of a simple "Chain", I implemented a **State Graph** system.
*   **State:** `CurrentIncidentState` (contains `alert_json`, `logs`, `retrieved_docs`).
*   **Nodes:**
    1.  `classifier`: Analyzes the alert type (Database? API? Network?).
    2.  `investigator`: Calls MCP tools based on classification.
    3.  `synthesizer`: Combines tool outputs with RAG context.
    4.  `reviewer`: Self-correction step (Did I answer the root cause? Do I need more logs?).

### B. Data & RAG Pipeline (Context Layer)
We needed the AI to know *our* systems, not just generic debugging.
*   **Sources:** Confluence (Runbooks), Jira (Past Post-mortems/RCAs), **GitLab (Source Code, Readme/Config)**.
*   **Embedding:** `text-embedding-3-small` (OpenAI).
*   **Retrieval:** Search (alert + logs) to ensure specific error codes 
*   **Retrieval Strategy:** 
    1.  **Initial Search:** Use alert title + error codes to fetch top 5 relevant docs.
    2.  **Contextual Augmentation:** After log retrieval, re-query with log snippets to get more targeted runbooks.
*  **Reranking:** Used a simple TF-IDF based reranker to prioritize documents that mention exact error codes or service names.


### C. MCP (Model Context Protocol) Integration
I standardized all infrastructure access using MCP servers. This decoupled the Agent logic from the API implementation details.
*   **`mcp-datadog`**: 
    *   Tool: `get_logs(service_name, start_time, end_time, filter_pattern)` - Fetches actual error logs.
    *   Tool: `get_metrics(metric_name, window)` - Checks CPU/Memory spikes.
*   **`mcp-gitlab`**:
    *   Tool: `check_recent_deployments(repo, window)` - "Was code deployed in the last 30 mins?" (Correlation != Causation, but high signal).
    *   Tool: `get_file_context(repo, file_path, line_range)` - Fetches code snippets around stack trace errors to understand logic.
    *   Tool: `search_codebase(repo, query)` - Get the code related to the alert and error
        - Searches using vector embeddings for semantic relevance using vector DB.
*   **`mcp-pagerduty`**:
    *   Tool: `get_oncall_schedule(service)` - "Who else should I tag?"

### D. Managing Context & Noise (The "Log Dump" Problem)
Raw logs are too noisy for LLMs.
*   **Log Filtering:** Implemented a pre-processing step using lightweight Regex or smaller models to filter out "INFO" and "DEBUG" logs, keeping only "ERROR" and "WARN" + Stack Traces.
*   **Context Window:** Used a sliding window approach. If logs > 100k tokens, we summarize chunks and pass the summary to the final reasoning step.

## 5. Key Challenges & Optimizations

### Challenge 1: Hallucinations (Making up commands or errors)
*   **Fix:** **Grounding**. The System Prompt enforces: *"You must quote the exact log line ID when citing an error."* RAG outputs include metadata links to the source (Confluence URL).

### Challenge 2: Latency
*   **Fix:** **Parallel Tool Calling**. The `investigator` node triggers `fetch_logs` and `search_runbooks` concurrently using `asyncio.gather`, reducing average analysis time from 45s to 15s.

### Challenge 3: Slack formatting
*   **Fix:** Used Slack "Block Kit" for the output. The bot doesn't just dump text; it renders:
    *   🔴 **Critical Error** (Section)
    *   🔍 **Evidence** (Collapsible Code Block)
    *   🛠️ **Suggested Fix** (Button: "Run Restart Playbook" - *Human in loop approval req*)

## 6. Business Impact
*   **Result:** Reduced MTTR (Mean Time To Resolution) by **~40%** for common incidents (e.g., Disk Full, High Latency).
*   **Adoption:** Used by 15+ engineering squads.
*   **Culture:** Shifted on-call from "Panic Response" to "Review & Confirm".



Comparison methodology (before/after cohort matching)
Summary of Critical Fixes Needed:
Priority	Issue	Quick Fix
P0	No error handling	Add fallback mechanisms
P0	FAISS persistence	S3 backup/restore
P0	Missing security	Add Secrets Manager + IAM
P1	State graph logic unclear	Document transitions
P1	No observability	Add CloudWatch metrics
P2	Dual LLM justification	Cost-benefit analysis
P2	GitLab indexing scope	Filter strategy
