# AI Engineering for Senior Python Developers: From Script to Sentinel

This tutorial is designed for senior Python developers transitioning into AI Engineering. It moves beyond "Hello World" to building robust, production-grade applications using **LangChain**, **RAG (FAISS)**, and **Agents**.

We will build two services based on real-world reliability engineering use cases:
1.  **Code Review Service:** A RAG-powered automated code reviewer.
2.  **Oncall AI Assist:** An agentic workflow for incident triage.

---

## 📚 Section 1: The AI Glossary (Senior Dev Edition)

Before writing code, let's map AI concepts to terms you already know.

### 1. Large Language Model (LLM)
*   **Definition:** A probabilistic model that predicts the next token in a sequence.
*   **Senior Dev Analogy:** Think of it as a `autocomplete` function that has read the entire internet, but it has no memory of past requests (stateless) and no access to the outside world unless you give it tools.
*   **Key Parameters:**
    *   **Temperature:** Randomness (0.0 = Deterministic/Code, 1.0 = Creative/Poetry).
    *   **Context Window:** The maximum amount of text (tokens) looking back the model can "see" at once (e.g., 128k tokens).

### 2. Embeddings & Vector Databases
*   **Embedding:** A function that turns text into a vector (array of floats, e.g., `[0.1, -0.5, ...]` of length 1536). Semantically similar text has vectors that are geometrically close (Cosine Similarity).
*   **Vector Database (e.g., FAISS, qdrant, Pinecone):** A specialized database optimized for "Nearest Neighbor" search.
*   **Senior Dev Analogy:** It's a fuzzy search engine. Instead of `WHERE text LIKE '%error%'`, you query `WHERE meaning IS_SIMILAR_TO 'database connection failed'`.

### 3. RAG (Retrieval-Augmented Generation)
*   **Definition:** A technique to provide private data to an LLM.
*   **Process:** User Query -> Search Vector DB -> Get Relevant docs -> Paste docs into System Prompt -> Send to LLM.
*   **Why?** LLMs freeze in time (training cut-off). RAG gives them access to *your* current runbooks and code.

### 4. Chains vs. Agents vs. Graphs
*   **Chain:** A procedural pipeline (Step A -> Step B -> Step C). Hardcoded logic.
*   **Agent:** An LLM loop that decides *which* tools to call and *in what order* to solve a problem. It has agency.
*   **State Graph (LangGraph):** A Finite State Machine (FSM) where nodes are LLM steps and edges are conditional logic. Used for complex, stateful workflows (like the Oncall Assist).

---

## 🛠️ Section 2: Environment Setup

We will use `langchain`, `langgraph`, `faiss-cpu`, and `openai`.

```bash
# Create a virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install langchain langchain-openai langchain-community langgraph faiss-cpu tiktoken
```

**Environment Variables (`.env`):**
```bash
OPENAI_API_KEY="sk-..."
```

---

## 🏗️ Section 3: Project 1 - The RAG-Powered Code Reviewer

**Goal:** Create a service that reviews code (specifically a Python diff) against a customized "Style Guide" stored in a vector database.

### Step 1: Ingestion (Building the Knowledge Base)
First, we need to index our "Style Guide". In a real app, this might be your `CONTRIBUTING.md`.

```python
# ingest_style_guide.py
from langchain_community.document_loaders import TextLoader
from langchain_text_splitters import CharacterTextSplitter
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS

def build_vector_store():
    # 1. Load the Style Guide
    # Assume styles.md contains rules like "Always utilize type hinting"
    loader = TextLoader("./resources/styles.md")
    documents = loader.load()

    # 2. Split text into chunks (LLMs have token limits)
    text_splitter = CharacterTextSplitter(chunk_size=500, chunk_overlap=0)
    docs = text_splitter.split_documents(documents)

    # 3. Create Embeddings & Store in FAISS
    embeddings = OpenAIEmbeddings()
    vector_store = FAISS.from_documents(docs, embeddings)
    
    # 4. Save to disk (Serialization)
    vector_store.save_local("faiss_style_index")
    print("Vector store saved!")

if __name__ == "__main__":
    build_vector_store()
```

### Step 2: The Review Service (The RAG Chain)
Now, we create the reviewer that uses this knowledge.

```python
# reviewer.py
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings

# 1. Load the Vector Store
embeddings = OpenAIEmbeddings()
vector_store = FAISS.load_local("faiss_style_index", embeddings, allow_dangerous_deserialization=True)
retriever = vector_store.as_retriever(search_kwargs={"k": 2}) # Get top 2 most relevant rules

# 2. Define the LLM
llm = ChatOpenAI(model="gpt-4-turbo", temperature=0)

# 3. Define the Prompt Template
# {context} will be filled by the RAG retrieval
# {diff} will be the user input
template = """
You are a Senior Principal Engineer. Review the following code diff.
Strictly adhere to the internal style guidelines provided below.

INTERNAL STYLE GUIDELINES:
{context}

CODE DIFF TO REVIEW:
{diff}

Provide feedback in markdown format. If the code is perfect, say "LGTM".
"""
prompt = ChatPromptTemplate.from_template(template)

# 4. Build the Chain (LCEL - LangChain Expression Language)
# specific_rules = retriever | format_docs
chain = (
    {"context": retriever, "diff": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

# 5. Execute
sample_diff = """
+ def calculate_sum(a,b):
+    return a+b
"""

result = chain.invoke(sample_diff)
print(result)
```

**What just happened?**
1. The `retriever` saw the diff (or a query derived from it).
2. It fetched relevant style rules from FAISS.
3. It injected those rules into `{context}`.
4. The LLM reviewed the diff *using* those specific rules.

---

## 🤖 Section 4: Project 2 - Oncall AI Assist (Agents & State Graphs)

**Goal:** An intelligent agent that alerts, investigates logs, checks runbooks, and synthesizes a Root Cause Analysis (RCA).

**Architecture:** We will use **LangGraph** to build a State Machine.
*   **State:** The shared memory (alert data, logs found, final diagnosis).
*   **Nodes:** Functions that do work (Classifier, Investigator).
*   **Edges:** Logic flow (If DB error -> check DB logs).

### Step 1: Define the State
This acts as the "Context Object" passed between steps.

```python
from typing import TypedDict, List
from langchain_core.messages import BaseMessage

class IncidentState(TypedDict):
    alert_payload: str
    classification: str     # e.g., "Database", "Network", "Application"
    logs: List[str]         # Evidence collected
    runbook_context: str    # RAG retrieved content
    final_report: str
```

### Step 2: Define Tools (Simulated MCP)
In a real app, these would connect to PagerDuty/Datadog APIs.

```python
from langchain_core.tools import tool

@tool
def fetch_database_logs(service: str):
    """Fetches text logs for database services."""
    # Mock return
    return f"[ERROR] ConnectionRefused: Postgres connection limit exceeded for {service}."

@tool
def fetch_app_logs(service: str):
    """Fetches standard application logs."""
    return f"[INFO] {service} health check passing."
```

### Step 3: Define Nodes (The "Brain" Logic)

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

llm = ChatOpenAI(model="gpt-4-turbo")

def classifier_node(state: IncidentState):
    """Determines the type of incident based on the alert."""
    payload = state["alert_payload"]
    response = llm.invoke(f"Classify this alert into 'Database' or 'Application': {payload}")
    return {"classification": response.content.strip()}

def investigator_node(state: IncidentState):
    """Executes tools based on classification."""
    category = state["classification"]
    evidence = []
    
    if "Database" in category:
        # verify with LLM or hardcode tool call
        evidence.append(fetch_database_logs.invoke({"service": "payment-db"}))
    else:
        evidence.append(fetch_app_logs.invoke({"service": "frontend"}))
        
    return {"logs": evidence}

def reporter_node(state: IncidentState):
    """Synthesizes the final RCA One-Pager."""
    prompt = f"""
    Write an incident report.
    Alert: {state['alert_payload']}
    Evidence: {state['logs']}
    """
    response = llm.invoke(prompt)
    return {"final_report": response.content}
```

### Step 4: The Graph (The Workflow)

```python
from langgraph.graph import StateGraph, END

# Initialize Graph
workflow = StateGraph(IncidentState)

# Add Nodes
workflow.add_node("classify", classifier_node)
workflow.add_node("investigate", investigator_node)
workflow.add_node("report", reporter_node)

# Add Edges (Control Flow)
workflow.set_entry_point("classify")
workflow.add_edge("classify", "investigate")
workflow.add_edge("investigate", "report")
workflow.add_edge("report", END)

# Compile
app = workflow.compile()
```

### Step 5: Execute
```python
alert = "PAGERDUTY ALERT: High Latency on payment-db service. Error rate > 5%."

inputs = {"alert_payload": alert, "logs": [], "classification": "", "final_report": ""}
result = app.invoke(inputs)

print("--- FINAL REPORT ---")
print(result["final_report"])
```

**Why LangGraph?**
Unlike a simple chain, LangGraph allows for **Loops** and **Human-in-the-loop**.
*   *Example:* If the `investigator` finds no logs, you can draw an edge back to `investigate` with different query parameters, creating a self-correcting loop.

---

## 🚀 Key Takeaways for Senior Devs

1.  **Deterministic vs. Probabilistic:** Coding is deterministic. Prompting is probabilistic. Always implement **Guardrails** (like PydanticOutputParser or validation nodes) to ensure the LLM output matches your type expectations.
2.  **Context is King:** The quality of your RAG retrieval determines the genius of your bot. Garbage in (bad search results), Garbage out (bad hallucinations).
3.  **Latency Matters:** LLMs are slow. Use `async` extensively. In LangGraph, nodes can run in parallel (e.g., fetch logs AND search runbooks simultaneously).
4.  **Evaluation:** How do you test this? You need "LLM-as-a-Judge". Create a test set of inputs and use GPT-4 to grade the responses of your system.

## Recommended Further Reading
*   **LangChain Expression Language (LCEL):** The unix-pipe syntax for combining AI components.
*   **Model Context Protocol (MCP):** The new standard for connecting AI models to data sources (Datadog, Jira, etc).
