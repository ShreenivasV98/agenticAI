# Sentinel-Agent 🛡️

> **A production-grade LangGraph multi-agent system for autonomous IT operations — monitors infrastructure, triages alerts, performs root cause analysis, and creates remediation tickets. Zero manual handoff.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/)
[![LangGraph](https://img.shields.io/badge/LangGraph-0.1-orange.svg)](https://langchain-ai.github.io/langgraph/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.111-green.svg)](https://fastapi.tiangolo.com/)

---

## What is Sentinel-Agent?

Sentinel-Agent is a production-ready multi-agent system that automates the Level 1 and Level 2 incident response workflow:

```
Alert fires → Agent validates → Agent diagnoses → Agent creates ticket → Agent notifies
```

No human touches this flow unless the system explicitly escalates.

It is directly inspired by real enterprise agentic automation work and is designed to be:
- **Deployable** — Docker Compose for local, Kubernetes manifests for prod
- **Observable** — LangSmith traces + Prometheus metrics + structured logging
- **Auditable** — every agent decision logged with reasoning chain
- **Extensible** — add new agents as plugins without modifying core orchestration

---

## Agent Roster

| Agent | Responsibility |
|---|---|
| **Watcher Agent** | Polls alert sources (Prometheus, webhook, Kafka topic); applies deduplication and noise filtering |
| **Validator Agent** | Enriches alert with system context; determines if alert is real or flapping noise |
| **Analyst Agent** | Performs root cause analysis using log retrieval (RAG over log corpus) + LLM reasoning |
| **Dispatcher Agent** | Creates and assigns incident ticket in ServiceNow / Jira; sets priority and SLA timer |
| **Notifier Agent** | Sends structured Slack/Teams notification with RCA summary and ticket link |
| **Escalator Agent** | Monitors SLA breach; escalates to human on-call if no resolution within threshold |

---

## Features

| Feature | Details |
|---|---|
| **LangGraph state machine** | Typed `AgentState`, conditional edges, human-in-the-loop pause points |
| **Pluggable alert sources** | Prometheus Alertmanager, webhooks, Kafka consumer |
| **RAG-powered log analysis** | Analyst Agent retrieves from ChromaDB log corpus for RCA |
| **ServiceNow / Jira integration** | Dispatcher creates tickets via REST API |
| **Slack / Teams notifications** | Notifier sends rich-card alerts with RCA + ticket link |
| **LangSmith observability** | Full trace of every agent call, tool use, and LLM invocation |
| **Prometheus metrics** | `alerts_processed_total`, `rca_latency_seconds`, `escalations_total` |
| **Redis memory** | Conversation context persisted across agent hops per incident |
| **Docker + K8s ready** | Compose for local dev; Helm chart in `deploy/` for cluster deployment |

---

## Architecture

See [`ARCHITECTURE.md`](ARCHITECTURE.md) for the full state machine diagram and agent interaction design.

---

## Quick Start

```bash
git clone https://github.com/ShreenivasV98/sentinel-agent
cd sentinel-agent

cp .env.example .env
# Required: OPENAI_API_KEY, LANGSMITH_API_KEY
# Optional: SERVICENOW_URL, JIRA_URL, SLACK_WEBHOOK_URL

docker-compose up -d

# Trigger a test alert
curl -X POST http://localhost:8000/alert \
  -H "Content-Type: application/json" \
  -d '{
    "alert_name": "HighCPUUsage",
    "severity": "critical",
    "host": "prod-api-03",
    "metric_value": 94.2,
    "labels": {"env": "production", "service": "payment-api"}
  }'

# Watch the agent graph execute
open http://localhost:8501   # Streamlit trace viewer
```

---

## Project Structure

```
sentinel-agent/
├── api/
│   ├── main.py                   # FastAPI: /alert, /incidents, /health
│   └── models.py                 # Pydantic alert + incident schemas
│
├── graph/
│   ├── state.py                  # AgentState TypedDict definition
│   ├── graph.py                  # LangGraph StateGraph construction
│   └── edges.py                  # Conditional routing logic
│
├── agents/
│   ├── base.py                   # BaseAgent: logging, error handling, retry
│   ├── watcher.py                # Alert ingestion + deduplication
│   ├── validator.py              # Alert validation + enrichment
│   ├── analyst.py                # RCA via RAG + LLM reasoning
│   ├── dispatcher.py             # Ticket creation (ServiceNow / Jira)
│   ├── notifier.py               # Slack / Teams notification
│   └── escalator.py             # SLA monitoring + human escalation
│
├── tools/
│   ├── prometheus_tool.py        # Query Prometheus API for metric history
│   ├── log_retrieval_tool.py     # RAG over ChromaDB log corpus
│   ├── servicenow_tool.py        # ServiceNow REST API wrapper
│   ├── jira_tool.py              # Jira REST API wrapper
│   └── slack_tool.py             # Slack Block Kit notification sender
│
├── memory/
│   └── redis_memory.py           # Redis-backed conversation memory per incident_id
│
├── observability/
│   ├── metrics.py                # Prometheus metrics registry
│   └── tracing.py                # LangSmith tracer setup
│
├── dashboard/
│   └── app.py                    # Streamlit: live incident trace viewer
│
├── tests/
│   ├── test_graph.py             # Graph execution unit tests
│   ├── test_agents.py            # Individual agent mock tests
│   └── fixtures/
│       └── sample_alerts.json
│
├── deploy/
│   ├── helm/                     # Helm chart for Kubernetes
│   └── k8s/                      # Raw Kubernetes manifests
│
├── docker-compose.yml
├── Dockerfile
├── pyproject.toml
└── .env.example
```

---

## AgentState Schema

```python
class AgentState(TypedDict):
    # Input
    alert_id: str
    alert_name: str
    severity: str
    host: str
    metric_value: float
    labels: Dict[str, str]

    # Validation
    is_valid: bool
    validation_reason: str
    enriched_context: Dict[str, Any]

    # Analysis
    rca_summary: str
    rca_confidence: float
    log_evidence: List[str]
    recommended_action: str

    # Dispatch
    ticket_id: str
    ticket_url: str
    assigned_team: str
    priority: Literal["P1", "P2", "P3", "P4"]

    # Notification
    notification_sent: bool
    notification_ts: str

    # Control
    escalated: bool
    human_review_required: bool
    error: Optional[str]
    trace_id: str
```

---

## Configuration

```yaml
# config.yaml
agents:
  validator:
    llm: gpt-4o-mini
    temperature: 0.0
  analyst:
    llm: gpt-4o
    temperature: 0.0
    rag_top_k: 8
    use_reranker: true
  dispatcher:
    ticket_system: servicenow    # or jira
    default_priority: P2
  escalator:
    sla_breach_minutes: 30
    escalation_channel: "#oncall-engineering"

memory:
  backend: redis
  ttl_seconds: 86400           # 24 hours per incident

observability:
  langsmith_project: sentinel-agent-prod
  prometheus_port: 9090
```

---

## Tech Stack

- **LangGraph** — stateful multi-agent orchestration
- **LangChain** — tool definitions, LLM abstraction, RAG chain
- **LangSmith** — end-to-end agent trace observability
- **ChromaDB** — log corpus vector store for RCA retrieval
- **Redis** — incident-scoped agent memory
- **FastAPI** — alert ingestion and incident query API
- **Prometheus** — operational metrics
- **Streamlit** — live incident trace dashboard
- **Docker Compose / Helm** — deployment

---

## Roadmap

- [ ] PagerDuty integration
- [ ] Auto-remediation actions (restart pod, clear cache, scale up)
- [ ] Multi-LLM routing (analyst on GPT-4o, validator on GPT-4o-mini)
- [ ] Incident similarity search (find past incidents like this one)
- [ ] Human-in-the-loop approval gate for P1 incidents

---

## License

MIT © [Shreenivas V](https://github.com/ShreenivasV98)
