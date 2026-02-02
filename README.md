# LangGraph Integration Plan for S-Delivery-System

## Overview

Replace the current if-else based AI chat system with LangGraph for better workflow management, multi-step reasoning, and human-in-the-loop capabilities.

---

## Current State

```
User Message → interpret_prompt() → if-else routing → Action → Response
```

**Problems:**
- No state persistence across requests
- No multi-step workflows
- No approval flows before dangerous actions (SMS)
- Hard to test and maintain

---

## Target Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    LANGGRAPH WORKFLOW                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  User Message ──▶ ROUTER ──▶ Planning Agent ──▶ Response       │
│                     │                                           │
│                     ├──────▶ Dispatch Agent ──▶ Response        │
│                     │                                           │
│                     ├──────▶ Compliance Agent ──▶ Response      │
│                     │                                           │
│                     └──────▶ Communication Agent                │
│                                    │                            │
│                              [Approval?]                        │
│                                    │                            │
│                              ▼     ▼                            │
│                           YES     NO ──▶ Response               │
│                            │                                    │
│                       Execute SMS ──▶ Response                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## New Directory Structure

```
backend/app/
├── agents/                          # NEW
│   ├── __init__.py
│   ├── state.py                     # WorkflowState TypedDict
│   ├── graph.py                     # Main graph definition
│   ├── nodes/
│   │   ├── __init__.py
│   │   ├── router.py                # Intent detection
│   │   ├── planning.py              # Driver selection, plans
│   │   ├── dispatch.py              # Check-in/out, assignments
│   │   ├── compliance.py            # Hour tracking, violations
│   │   ├── communication.py         # SMS with approval
│   │   └── response_generator.py    # Final formatting
│   ├── tools/
│   │   ├── __init__.py
│   │   ├── sql_tools.py             # DB queries
│   │   ├── scoring_tools.py         # Driver scoring
│   │   ├── sms_tools.py             # Twilio SMS
│   │   └── compliance_tools.py      # Hour calculations
│   └── checkpoints/
│       ├── __init__.py
│       └── postgres_checkpoint.py   # State persistence
├── langgraph_routes.py              # NEW: /api/v2/chat endpoints
└── services/
    └── llm_langgraph.py             # NEW: LangGraph LLM config
```

---

## Phase 1: Foundation (Week 1-2)

### Tasks

1. **Add dependencies to `backend/requirements.txt`:**
   ```
   langgraph>=0.2.0
   langchain-core>=0.3.0
   langchain-groq>=0.2.0
   ```

2. **Create `backend/app/agents/state.py`:**
   ```python
   class WorkflowState(TypedDict):
       # Context
       organization_id: int
       user_id: int
       conversation_id: int
       session_id: str

       # Input
       user_message: str
       message_history: List[Dict]

       # Routing
       intent: str  # selection, notification, compliance_check, etc.
       confidence: float
       extracted_params: Dict[str, Any]

       # Plan context
       active_plan: Optional[PlanInfo]
       drivers: List[DriverInfo]

       # Human-in-the-loop
       requires_approval: bool
       approval_type: Optional[str]
       pending_sms: Optional[SMSPendingApproval]
       sms_approved: bool

       # Output
       response_text: str
       response_data: Dict[str, Any]
       errors: List[str]
       node_history: List[str]
   ```

3. **Create `backend/app/agents/nodes/router.py`:**
   - Migrate logic from `interpret_prompt()` in `llm.py`
   - Keep heuristic fallback for when LLM unavailable
   - Return intent + extracted_params

4. **Create `backend/app/agents/graph.py`:**
   ```python
   workflow = StateGraph(WorkflowState)
   workflow.add_node("router", router_node)
   workflow.add_node("response_generator", response_generator)
   workflow.set_entry_point("router")
   workflow.add_edge("router", "response_generator")
   workflow.add_edge("response_generator", END)
   ```

### Files to Create
- `backend/app/agents/__init__.py`
- `backend/app/agents/state.py`
- `backend/app/agents/graph.py`
- `backend/app/agents/nodes/__init__.py`
- `backend/app/agents/nodes/router.py`
- `backend/app/agents/nodes/response_generator.py`

### Verification
```bash
cd backend
python -c "from app.agents.graph import create_delivery_graph; g = create_delivery_graph(); print('OK')"
```

---

## Phase 2: Core Agents (Week 3-4)

### Tasks

1. **Create `backend/app/agents/nodes/planning.py`:**
   - Handle intents: `selection`, `revise`, `remove_worst`, `exclude`, `availability`
   - Use existing `compute_driver_score()` from `scoring.py`
   - Use existing `create_plan()` logic from `main.py` (lines 3378-3450)

2. **Create `backend/app/agents/nodes/dispatch.py`:**
   - Handle intents: `dispatch_checkin`, `dispatch_checkout`, `dispatch_callout`, `schedule`
   - Use existing RosterAssignment logic

3. **Create `backend/app/agents/nodes/compliance.py`:**
   - Handle intents: `compliance_check`, `compliance_report`
   - Use existing compliance calculation logic from `compliance_routes.py`

4. **Create tools in `backend/app/agents/tools/`:**
   ```python
   # sql_tools.py
   @tool
   def query_drivers(org_id: int, status: str, day: str) -> List[Dict]: ...

   @tool
   def create_plan(org_id: int, target: int, date: str) -> Dict: ...

   # scoring_tools.py
   @tool
   def score_drivers(driver_ids: List[int]) -> List[Dict]: ...
   ```

5. **Update graph with conditional routing:**
   ```python
   workflow.add_conditional_edges(
       "router",
       route_by_intent,
       {
           "planning_agent": "planning_agent",
           "dispatch_agent": "dispatch_agent",
           "compliance_agent": "compliance_agent",
           "response_generator": "response_generator",
       }
   )
   ```

### Files to Create
- `backend/app/agents/nodes/planning.py`
- `backend/app/agents/nodes/dispatch.py`
- `backend/app/agents/nodes/compliance.py`
- `backend/app/agents/tools/__init__.py`
- `backend/app/agents/tools/sql_tools.py`
- `backend/app/agents/tools/scoring_tools.py`

### Files to Reference (Read-Only)
- `backend/app/main.py` (lines 2408-2984) - Current chat logic
- `backend/app/services/scoring.py` - Driver scoring
- `backend/app/compliance_routes.py` - Compliance logic

### Verification
```bash
pytest tests/test_agents/test_planning.py -v
pytest tests/test_agents/test_dispatch.py -v
```

---

## Phase 3: Communication & Human-in-the-Loop (Week 5)

### Tasks

1. **Create `backend/app/agents/nodes/communication.py`:**
   ```python
   async def communication_agent(state: WorkflowState, config: dict):
       if state.get("sms_approved"):
           # Execute SMS send
           result = await send_sms(state["pending_sms"])
           return {...state, "sms_result": result}

       # Prepare for approval
       recipients = await get_recipients(...)
       message = await compose_message(...)

       return {
           ...state,
           "pending_sms": {"recipients": recipients, "message": message},
           "requires_approval": True,
           "approval_type": "sms",
           "approval_prompt": f"Send to {len(recipients)} drivers?"
       }
   ```

2. **Create `backend/app/agents/checkpoints/postgres_checkpoint.py`:**
   ```python
   from langgraph.checkpoint.postgres import PostgresSaver

   def get_checkpointer():
       if DATABASE_URL:
           return PostgresSaver.from_conn_string(DATABASE_URL)
       return MemorySaver()  # Fallback for dev
   ```

3. **Create `backend/app/agents/tools/sms_tools.py`:**
   - Wrap existing `sms_client.send_bulk()`
   - Add logging to MessageLog table

4. **Update graph for approval flow:**
   ```python
   workflow.add_conditional_edges(
       "communication_agent",
       lambda s: "await_approval" if s["requires_approval"] else "response_generator",
       {...}
   )
   ```

### Files to Create
- `backend/app/agents/nodes/communication.py`
- `backend/app/agents/checkpoints/__init__.py`
- `backend/app/agents/checkpoints/postgres_checkpoint.py`
- `backend/app/agents/tools/sms_tools.py`

### Verification
```bash
# Test approval flow
pytest tests/test_agents/test_communication.py -v
```

---

## Phase 4: API Integration (Week 6)

### Tasks

1. **Create `backend/app/langgraph_routes.py`:**
   ```python
   router = APIRouter(prefix="/api/v2/chat")

   @router.post("/message")
   async def langgraph_message(conv_id: int, text: str, ...):
       # Build initial state
       # Run graph
       # Store messages
       # Return response

   @router.post("/approve")
   async def approve_action(session_id: str, approved: bool, ...):
       # Resume checkpointed workflow
       # Execute if approved
   ```

2. **Register routes in `backend/app/main.py`:**
   ```python
   from .langgraph_routes import router as langgraph_router
   app.include_router(langgraph_router)
   ```

3. **Add feature flag:**
   ```python
   # .env
   USE_LANGGRAPH=false  # Toggle for gradual rollout
   ```

4. **Create backwards-compatible wrapper:**
   ```python
   @app.post("/api/chat/message")
   async def chat_message_compat(...):
       if os.getenv("USE_LANGGRAPH") == "true":
           return await langgraph_message(...)
       return await existing_chat_message(...)
   ```

### Files to Create
- `backend/app/langgraph_routes.py`

### Files to Modify
- `backend/app/main.py` (add router include + feature flag)
- `backend/.env` (add USE_LANGGRAPH)

### Verification
```bash
# Start server
uvicorn app.main:app --reload

# Test v2 endpoint
curl -X POST http://localhost:8000/api/v2/chat/message \
  -F conv_id=1 -F text="give me 20 drivers"

# Test health
curl http://localhost:8000/api/v2/chat/health
```

---

## Phase 5: Testing & Rollout (Week 7-8)

### Tasks

1. **Create test suite:**
   ```
   tests/
   ├── test_agents/
   │   ├── test_router.py
   │   ├── test_planning.py
   │   ├── test_dispatch.py
   │   ├── test_compliance.py
   │   ├── test_communication.py
   │   └── test_graph_integration.py
   ```

2. **Shadow mode testing:**
   - Log both v1 and v2 responses
   - Compare intent detection accuracy
   - Monitor latency

3. **Gradual rollout:**
   - Week 7: 10% traffic → 25% → 50%
   - Week 8: 100% traffic
   - Deprecate v1 endpoints

4. **Frontend updates (if needed):**
   - Handle `requires_approval` response
   - Add approval UI for SMS sends
   - Call `/api/v2/chat/approve` endpoint

### Verification
```bash
# Run full test suite
pytest tests/test_agents/ -v --cov=app.agents

# Load test
locust -f tests/load/test_langgraph.py
```

---

## Critical Files Reference

| File | Purpose | Action |
|------|---------|--------|
| `backend/app/main.py` | Current chat endpoints (lines 2408-2984) | Reference, then modify |
| `backend/app/services/llm.py` | `interpret_prompt()` function | Migrate to router.py |
| `backend/app/services/scoring.py` | `compute_driver_score()` | Wrap as tool |
| `backend/app/services/sms.py` | SMS sending | Wrap as tool |
| `backend/app/models.py` | All DB models | Reference only |
| `backend/app/compliance_routes.py` | Compliance logic | Migrate to compliance.py |

---

## Dependencies to Add

```txt
# backend/requirements.txt (additions)
langgraph>=0.2.0
langchain-core>=0.3.0
langchain-groq>=0.2.0
langchain-community>=0.3.0
```

---

## Environment Variables

```env
# New variables for LangGraph
USE_LANGGRAPH=false          # Feature flag
LANGGRAPH_CHECKPOINT=postgres # or "memory" for dev

# Existing (already configured)
LLM_API_BASE=https://api.groq.com/openai/v1
LLM_API_KEY=gsk_xxx
LLM_MODEL=llama-3.1-8b-instant
DATABASE_URL=postgresql://...
```

---

## Success Metrics

| Metric | Target |
|--------|--------|
| Intent detection accuracy | > 95% |
| Response latency (p95) | < 2 seconds |
| Error rate | < 1% |
| Human-in-the-loop SMS | 100% approval before send |
| Test coverage | > 80% |

---

## Rollback Plan

If issues arise:
1. Set `USE_LANGGRAPH=false` in environment
2. Restart server
3. All traffic returns to v1 endpoints
4. Investigate and fix
5. Re-enable gradually
