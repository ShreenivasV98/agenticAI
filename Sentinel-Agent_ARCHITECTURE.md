# Sentinel-Agent — Architecture

## System Overview

Sentinel-Agent is built on a **LangGraph StateGraph** — a directed graph where each node is an agent and edges are conditional routing decisions based on the evolving `AgentState`.

The system has three layers:
1. **Ingestion Layer** — FastAPI receives alerts; Watcher Agent deduplicates and normalises
2. **Orchestration Layer** — LangGraph executes the agent graph; Redis persists incident state
3. **Integration Layer** — Tools connect agents to external systems (Prometheus, ServiceNow, Slack)

---

## LangGraph State Machine

```
                         ┌─────────────────┐
                         │   ALERT INPUT   │
                         │  POST /alert    │
                         └────────┬────────┘
                                  │
                         ┌────────▼────────┐
                         │    WATCHER      │
                         │    AGENT        │
                         │                 │
                         │  • Deduplication│
                         │  • Normalisation│
                         │  • Source enrich│
                         └────────┬────────┘
                                  │
                    ┌─────────────▼─────────────┐
                    │       VALIDATOR AGENT      │
                    │                           │
                    │  • Metric history lookup  │
                    │  • Flap detection         │
                    │  • Severity confirmation  │
                    └──────┬────────────┬───────┘
                           │            │
                   is_valid=False   is_valid=True
                           │            │
                    ┌──────▼──┐  ┌──────▼──────────┐
                    │  NOISE  │  │  ANALYST AGENT  │
                    │  DROP   │  │                 │
                    │  (end)  │  │ • RAG log search│
                    └─────────┘  │ • LLM reasoning │
                                 │ • RCA summary   │
                                 │ • Confidence    │
                                 └──────┬──────────┘
                                        │
                           ┌────────────▼────────────┐
                           │    DISPATCHER AGENT     │
                           │                         │
                           │ • Priority assignment   │
                           │ • Team routing          │
                           │ • Ticket creation       │
                           │   (ServiceNow / Jira)   │
                           └────────────┬────────────┘
                                        │
                           ┌────────────▼────────────┐
                           │    NOTIFIER AGENT       │
                           │                         │
                           │ • Slack/Teams message   │
                           │ • Rich card with RCA    │
                           │ • Ticket link + SLA     │
                           └────────────┬────────────┘
                                        │
                           ┌────────────▼────────────┐
                           │   ESCALATOR AGENT       │
                           │   (background monitor)  │
                           │                         │
                           │ • Watches SLA timer     │
                           │ • Escalates on breach   │
                           │ • Pages human on-call   │
                           └────────────┬────────────┘
                                        │
                              ┌─────────▼────────┐
                              │  RESOLVED / END  │
                              └──────────────────┘
```

---

## Graph Construction (LangGraph)

```python
from langgraph.graph import StateGraph, END

builder = StateGraph(AgentState)

# Register nodes
builder.add_node("watcher",    watcher_agent)
builder.add_node("validator",  validator_agent)
builder.add_node("analyst",    analyst_agent)
builder.add_node("dispatcher", dispatcher_agent)
builder.add_node("notifier",   notifier_agent)
builder.add_node("escalator",  escalator_agent)

# Entry point
builder.set_entry_point("watcher")

# Linear edges
builder.add_edge("watcher",    "validator")
builder.add_edge("analyst",    "dispatcher")
builder.add_edge("dispatcher", "notifier")
builder.add_edge("notifier",   "escalator")
builder.add_edge("escalator",  END)

# Conditional edge: validator → analyst OR drop
builder.add_conditional_edges(
    "validator",
    route_after_validation,           # returns "analyst" or END
    {"analyst": "analyst", END: END}
)

graph = builder.compile(
    checkpointer=RedisCheckpointer(),  # persists state per thread_id
    interrupt_before=["dispatcher"]    # human-in-the-loop pause for P1
)
```

---

## Agent Internals

### Validator Agent

```
Input:  alert (name, host, severity, labels)
        
Step 1: Tool call → prometheus_tool.query_range(
            metric=alert.metric_name,
            host=alert.host,
            duration="30m"
        )
        → Returns: time-series of metric values

Step 2: Flap detection
        → If metric crossed threshold < 3 times in 30m → mark as flapping → is_valid=False

Step 3: LLM reasoning
        prompt: "Given metric history {history} and alert {alert}, 
                 is this a genuine incident or transient noise? 
                 Respond with: valid|noise, reason, enriched_context"
        
Step 4: Parse structured output → update AgentState
```

### Analyst Agent (RAG-powered RCA)

```
Input:  validated alert + enriched_context

Step 1: Construct log search query from alert context
        query = f"{alert.host} {alert.alert_name} error {alert.labels['service']}"

Step 2: Tool call → log_retrieval_tool.search(
            query=query,
            top_k=8,
            time_range="last_2h"
        )
        → Returns: List[LogChunk] from ChromaDB

Step 3: LLM reasoning chain
        system: "You are an SRE performing root cause analysis."
        user:   "Alert: {alert}\n\nLog evidence:\n{log_chunks}\n\n
                 Metric context: {metric_history}\n\n
                 Provide: root_cause, confidence (0-1), 
                 recommended_action, affected_components"

Step 4: Structured output parsing → update AgentState
```

### Dispatcher Agent

```
Input:  rca_summary, recommended_action, severity

Step 1: Priority mapping
        P1: severity=critical AND rca_confidence > 0.8
        P2: severity=critical OR (severity=warning AND confidence > 0.7)
        P3: severity=warning
        P4: everything else

Step 2: Team routing table lookup (from config)
        service=payment-api → @payments-oncall
        service=auth        → @platform-oncall

Step 3: Tool call → servicenow_tool.create_incident(
            short_description=f"[AUTO] {alert.alert_name} on {alert.host}",
            description=rca_summary,
            priority=priority,
            assignment_group=team,
            work_notes=f"RCA Confidence: {rca_confidence}\nAction: {recommended_action}"
        )
        → Returns: ticket_id, ticket_url
```

---

## Memory Architecture

```
Per-incident Redis key: incident:{incident_id}:memory

Stores:
  - Full AgentState snapshot after each agent node
  - LangChain conversation history (for multi-turn analyst reasoning)
  - Metric data cache (avoid re-fetching Prometheus within same incident)
  - TTL: 24 hours

Format: JSON serialised AgentState
Access: RedisCheckpointer (native LangGraph checkpointer)

Thread model:
  - Each alert gets a unique thread_id = incident_id
  - LangGraph uses thread_id to isolate state between incidents
  - Concurrent incidents run independently in separate threads
```

---

## Tool Definitions

```python
# All tools are decorated with @tool for LangChain ToolNode compatibility

@tool
def query_prometheus(metric_name: str, host: str, duration_minutes: int) -> str:
    """Query Prometheus for metric time series. 
    Returns JSON string of {timestamp, value} pairs."""

@tool  
def search_logs(query: str, top_k: int, time_range_hours: int) -> str:
    """Semantic search over log corpus in ChromaDB.
    Returns top_k most relevant log chunks as JSON."""

@tool
def create_servicenow_incident(
    short_description: str, description: str, 
    priority: str, assignment_group: str, work_notes: str
) -> str:
    """Create incident in ServiceNow via REST API.
    Returns ticket_id and ticket_url."""

@tool
def send_slack_notification(
    channel: str, alert_name: str, host: str,
    rca_summary: str, ticket_url: str, priority: str
) -> str:
    """Send rich Block Kit Slack notification.
    Returns message timestamp."""
```

---

## Observability Stack

```
LangSmith Tracing:
  Every agent invocation → LangSmith trace
  Captures: LLM input/output, tool calls, latency, token usage
  Project: sentinel-agent-{env}

Prometheus Metrics (exposed at :9090/metrics):
  sentinel_alerts_received_total{severity, source}
  sentinel_alerts_valid_total{severity}
  sentinel_alerts_noise_total{}
  sentinel_rca_latency_seconds{confidence_bucket}
  sentinel_tickets_created_total{priority, system}
  sentinel_escalations_total{reason}
  sentinel_agent_errors_total{agent, error_type}

Structured Logging (JSON):
  Every agent step → log with:
    incident_id, agent, step, duration_ms, 
    llm_model, tokens_used, tool_calls, outcome
```

---

## Deployment

### Local (Docker Compose)

```yaml
services:
  api:          # FastAPI — port 8000
  chromadb:     # Log vector store — port 8001
  redis:        # State checkpointer + memory — port 6379
  prometheus:   # Metrics scraper — port 9090
  dashboard:    # Streamlit trace viewer — port 8501
```

### Production (Kubernetes)

```
deploy/helm/sentinel-agent/
  Chart.yaml
  values.yaml              # Configurable replicas, resource limits, image tags
  templates/
    deployment.yaml        # API + worker deployments
    service.yaml           # ClusterIP services
    configmap.yaml         # config.yaml mounted as ConfigMap
    secret.yaml            # API keys from K8s secrets
    hpa.yaml               # HorizontalPodAutoscaler for API
    servicemonitor.yaml    # Prometheus ServiceMonitor
```

---

## Key Design Decisions

| Decision | Rationale |
|---|---|
| LangGraph StateGraph over custom orchestrator | Built-in persistence, conditional routing, human-in-the-loop, streaming |
| RedisCheckpointer | LangGraph-native; enables state recovery if agent crashes mid-run |
| RAG for log RCA | Logs are too large for context window; retrieval focuses LLM on relevant evidence |
| Interrupt before dispatcher for P1 | P1 tickets need human confirmation; all others are fully automated |
| Structured output from all agents | Every LLM call uses `with_structured_output()` — no string parsing |
| Tools as `@tool` decorated functions | LangChain ToolNode handles retry, error propagation, schema validation |
