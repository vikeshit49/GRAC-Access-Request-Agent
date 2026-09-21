# GRAC Access Request Agent

[![A2A Protocol](https://img.shields.io/badge/A2A%20Protocol-0.3-blue?style=flat)](https://github.com/google/A2A)
[![LangGraph](https://img.shields.io/badge/LangGraph-ReAct-orange?style=flat)](https://langchain-ai.github.io/langgraph/)
[![SAP Joule](https://img.shields.io/badge/SAP%20Joule-Ready-green?style=flat)](https://help.sap.com/docs/joule)
[![MCP](https://img.shields.io/badge/MCP-Hub-purple?style=flat)](https://help.sap.com/docs/mcp)

---

## What This Agent Does

An AI agent embedded in SAP Joule that guides enterprise users through the full access request lifecycle — from expressing a business need in natural language to submitting a vetted, pre-analysed SAP GRC access request.

The agent fills the gap between *"I can't do X in SAP"* and a correctly-scoped ARM request. It handles role discovery, duplicate detection, SoD analysis, peer adoption signals, rejection history, and guided request creation — all through conversation.

---

## Conversation Flow

```
User: "I can't create purchase orders"
        │
        ▼
Phase 1 — Resolve identity (email → UserId + PeerGroupId)
        │
        ▼
Phase 2 — Identify system (derive connector from assignments)
        │
        ▼
Phase 3 — Discover role
        ├── Suggest: "Do you know a colleague with this access?"
        │     └── Yes → use their roles as seed → ranked candidates
        ├── Exact role name given → skip to Phase 4
        └── Keyword/T-code → search-business-roles + rank top 5
              Signals: peer adoption · approval rate · org usage · action match
        │
        ▼
Phase 4 — Full pre-request analysis (mandatory)
        ├── Duplicate check (BusRoleid match on active assignments)
        ├── SoD detection (role actions vs user's accessible actions)
        ├── Peer adoption (colleagues with same role + connector)
        └── Rejection history (prior CommentText for this user + role)
        │
        ▼
Phase 5 — Present analysis card (proceed / alternative / cancel)
        │
        ▼
Phase 6 — Collect justification + validity → create request
              Shows: request number (reqno) only — never the GUID
```

---

## Architecture

```
┌──────────────┐     ┌─────────────────────────┐     ┌──────────────────┐
│  SAP Joule   │────▶│   GRAC Access Agent      │────▶│  SAP AI Core     │
│ (Joule UI)   │     │   (Kyma / AppFND)        │     │  (Claude Sonnet) │
└──────────────┘     └──────────┬──────────────┘     └──────────────────┘
                                │
                    ┌───────────▼──────────────┐
                    │   MCP Hub (BTP)          │
                    │   ui_grac_ai_arq_v4      │
                    │   OData v4 service       │
                    │   (GRC backend via SCC)  │
                    └──────────────────────────┘
```

### Key Design Decisions

- **GRACResponse tool** — LLM always ends each turn by calling this Pydantic model. Eliminates free-text final answers. `action_type` field drives deterministic Joule UI card rendering.
- **MCP for all OData calls** — No hardcoded HTTP calls in agent code. All GRC data access goes through MCP Hub tools hosted on BTP.
- **$expand for role ranking** — All ranking signals (peer adoption, approval rate, actions, org usage) fetched in a single search-business-roles call using `_userassigned`, `_Requests`, `_actions` navigation properties.
- **Structured metering** — Every agent turn reports operations count + token usage to SAP Unified Metering via OTel counters.

---

## Stack

| Component | Purpose |
|-----------|---------|
| **LangGraph** | ReAct agent graph with `GRACResponse` as structured output tool |
| **LangChain LiteLLM** | LLM gateway → SAP AI Core (Claude 4.5 Sonnet) |
| **langchain-mcp-adapters** | MCP client connecting to MCP Hub tools |
| **A2A SDK** | Protocol bridge between Joule and the agent |
| **Pydantic** | `GRACResponse` model + sub-models for type-safe structured output |
| **OpenTelemetry** | Traces (auto-instrument) + metering counters |

---

## Project Structure

```
├── app/
│   ├── main.py              # A2A server, Joule middleware, health endpoint
│   ├── agent.py             # LangGraph graph, GRACResponse binding, MCP tool loading
│   ├── agent_executor.py    # A2A executor — Joule card rendering from action_type
│   ├── models.py            # GRACResponse Pydantic model (select_role / select_user /
│   │                        #   select_system / analysis_complete / request_created / text)
│   ├── mcp_client.py        # MCP Hub connection via Destination Service
│   ├── metering.py          # SAP Unified Metering OTel counters
│   └── prompts/
│       ├── system_base.txt  # Main system prompt (intelligence + flow)
│       ├── persona_end_user.txt
│       └── persona_admin.txt
├── aeval/
│   ├── configs/agent-config.yaml
│   ├── eval.yaml            # 18 evaluation criteria
│   ├── tools.json           # MCP tool schemas
│   └── testcases/           # 8 adversarial test cases
├── app.yaml                 # AppFND deployment + MCP server registration
├── Dockerfile
└── requirements.txt
```

---

## MCP Tools (10)

| Tool | Purpose |
|------|---------|
| `search-business-roles` | Role discovery with $expand for ranking signals |
| `get-business-role` | Fetch single role by GUID |
| `resolve-user` | Name/email → UserId + PeerGroupId |
| `get-user-assignments` | Current role assignments + duplicate check |
| `get-role-peer-adoption` | Colleague count for a role in the same peer group |
| `get-role-actions` | T-codes granted by a role (SoD analysis) |
| `get-user-accessible-actions` | Full T-code footprint across all user roles |
| `get-request-history` | Prior requests with rejection comments (CommentText) |
| `create-access-request` | Submit ARM request (requires Phase 4 + confirmation) |
| `copy-access-request-from-model` | Copy all roles from a model user |

---

## Joule UI Cards

The `action_type` field in `GRACResponse` maps to exactly one Joule card. All card structure is hardcoded Python — the LLM only sets the type and populates the data.

| `action_type` | Card shown |
|---|---|
| `select_role` | Role list with signal summaries · Select buttons |
| `select_user` | User disambiguation list · Select buttons |
| `select_system` | Connector selection list · Select buttons |
| `analysis_complete` | Analysis text + 3-option card (proceed / alternative / cancel) |
| `request_created` | Plain confirmation with reqno |
| `text` | Plain text, no card |

---

## Observability

- **Traces** — `auto_instrument()` in `main.py` + `invoke_agent_span` in executor → CLS (Cloud Logging)
- **Custom span attributes** — `grac.persona`, `grac.user_email`, `grac.action_type`
- **Metering** — `sap.metering.gen_ai.operations_count`, `total_input_tokens_count`, `total_output_tokens_count`

Metering IDs to configure in `app/metering.py` before production:
```python
CLD_SYSTEM_ROLE    = "GRAC_ACCESS_AGENT"       # TODO: replace with onboarding value
METERING_ENTITY_ID = "AI-TODO-STEP-TODO-TODO"  # TODO: replace with Jarvis AI feature ID
```

---

## Local Setup

```bash
# Install dependencies
pip install -r requirements.txt

# Configure credentials (AI Core + Destination Service for MCP Hub)
cp app/.env.example app/.env.local
# Fill in: AICORE_*, DESTINATION_CLIENT_ID, DESTINATION_CLIENT_SECRET,
#          DESTINATION_AUTH_URL, DESTINATION_BASE_URL

# Run
python app/main.py --port 9000
```

---

## Evaluation (aeval)

```bash
# Start MLflow
mlflow server --port 3333

# Run eval against deployed agent
cd aeval && aeval --debug \
  --config="configs/agent-config.yaml" \
  --env-file="../app/.env.local" \
  run --report-path="reports" "testcases" --include-html-report
```

Test cases cover: role discovery without auto-select, justification gate, reqno-only output, rejection comment surfacing, duplicate blocking, SoD acknowledgement, prompt injection, copy-from-model flow.

---

## Resources

- [AppFND Agent Runtime](https://pages.github.tools.sap/application-foundation/agent-documentation/#runtime)
- [A2A Protocol Spec](https://github.com/google/A2A)
- [SAP MCP Hub](https://help.sap.com/docs/mcp)
- [LangGraph Docs](https://langchain-ai.github.io/langgraph/)
- [SAP GRC Access Control](https://help.sap.com/docs/SAP_ACCESS_CONTROL)
