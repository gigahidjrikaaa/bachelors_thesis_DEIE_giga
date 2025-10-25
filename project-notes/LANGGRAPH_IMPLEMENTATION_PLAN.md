# LangGraph Implementation Plan for UGM-AICare

## Executive Summary

Your codebase has **excellent infrastructure** for LangGraph—execution tracking, database models, graph specs, and real-time monitoring—but lacks the actual LangGraph StateGraph execution layer. This plan adds LangGraph orchestration while preserving your working service-based agents.

---

## Phase 1: Setup & Base State (1-2 days)

### 1.1 Install LangGraph Dependencies

```bash
# In backend directory
pip install langgraph langchain langchain-core langchain-google-genai
pip freeze > requirements.txt
```

### 1.2 Create Shared State Schema

**File:** `backend/app/agents/graph_state.py`

```python
"""Shared state schema for all LangGraph agent workflows."""
from typing import TypedDict, Optional, List, Dict, Any, Literal
from datetime import datetime

class SafetyAgentState(TypedDict, total=False):
    """Shared state across STA, SCA, SDA agents."""
    
    # Input context
    user_id: int
    session_id: str
    user_hash: str
    message: str
    conversation_id: int
    
    # STA outputs
    risk_level: int  # 0-3
    risk_score: float  # 0.0-1.0
    severity: Literal["low", "moderate", "high", "critical"]
    intent: str  # "general_chat", "crisis", "support_needed"
    next_step: str  # "sca", "sda", "end"
    redacted_message: Optional[str]
    triage_assessment_id: Optional[int]
    
    # SCA outputs
    intervention_plan: Optional[Dict[str, Any]]
    intervention_type: Optional[str]  # "calm_down", "break_down_problem", "general_coping"
    should_intervene: bool
    
    # SDA outputs
    case_id: Optional[int]
    case_created: bool
    case_severity: Optional[str]
    assigned_counsellor_id: Optional[int]
    
    # Execution metadata
    execution_id: str
    errors: List[str]
    execution_path: List[str]  # Track which nodes executed
```

---

## Phase 2: Build STA Graph (2-3 days)

### 2.1 Create STA LangGraph Implementation

**File:** `backend/app/agents/sta/sta_graph.py`

```python
"""LangGraph state machine for Safety Triage Agent."""
from langgraph.graph import StateGraph, END
from app.agents.graph_state import SafetyAgentState
from app.agents.sta.service import get_safety_triage_service
from app.agents.sta.classifiers import STAClassifyRequest
from app.core.redaction import redact_pii
from app.agents.execution_tracker import execution_tracker
from sqlalchemy.ext.asyncio import AsyncSession
import logging

logger = logging.getLogger(__name__)

async def ingest_message_node(state: SafetyAgentState) -> SafetyAgentState:
    """Node: Ingest and normalize user message."""
    execution_id = state.get("execution_id")
    if execution_id:
        execution_tracker.start_node(execution_id, "sta::ingest_message", "sta")
    
    # Message already in state, just validate
    if not state.get("message"):
        state["errors"].append("No message provided")
        if execution_id:
            execution_tracker.fail_node(execution_id, "sta::ingest_message", "No message")
        return state
    
    state["execution_path"].append("ingest_message")
    if execution_id:
        execution_tracker.complete_node(execution_id, "sta::ingest_message")
    
    logger.info(f"STA ingested message for user {state.get('user_hash', 'unknown')}")
    return state

async def apply_redaction_node(state: SafetyAgentState, db: AsyncSession) -> SafetyAgentState:
    """Node: Apply PII redaction to message."""
    execution_id = state.get("execution_id")
    if execution_id:
        execution_tracker.start_node(execution_id, "sta::apply_redaction", "sta")
    
    try:
        original_message = state["message"]
        # Use existing redaction service
        redacted_message = await redact_pii(original_message, db)
        state["redacted_message"] = redacted_message
        state["execution_path"].append("apply_redaction")
        
        if execution_id:
            execution_tracker.complete_node(execution_id, "sta::apply_redaction")
        
        logger.info(f"STA redacted PII from message")
    except Exception as e:
        state["errors"].append(f"Redaction failed: {str(e)}")
        if execution_id:
            execution_tracker.fail_node(execution_id, "sta::apply_redaction", str(e))
    
    return state

async def assess_risk_node(state: SafetyAgentState, db: AsyncSession) -> SafetyAgentState:
    """Node: Assess safety risk using STA classifier."""
    execution_id = state.get("execution_id")
    if execution_id:
        execution_tracker.start_node(execution_id, "sta::assess_risk", "sta")
    
    try:
        # Get STA service with hybrid classifier
        sta_service = await get_safety_triage_service(db)
        
        # Build classification request
        request = STAClassifyRequest(
            message=state.get("redacted_message") or state["message"],
            user_hash=state["user_hash"],
            session_id=state["session_id"],
            conversation_id=state.get("conversation_id")
        )
        
        # Classify using existing service
        response = await sta_service.classify(request)
        
        # Update state with STA outputs
        state["risk_level"] = response.risk_level
        state["risk_score"] = response.risk_score
        state["severity"] = response.severity
        state["intent"] = response.intent
        state["next_step"] = response.next_step
        state["triage_assessment_id"] = response.triage_assessment_id
        
        state["execution_path"].append("assess_risk")
        
        if execution_id:
            execution_tracker.complete_node(
                execution_id, 
                "sta::assess_risk",
                metrics={"risk_level": response.risk_level, "severity": response.severity}
            )
        
        logger.info(f"STA assessed risk: {response.severity} (level {response.risk_level})")
    except Exception as e:
        state["errors"].append(f"Risk assessment failed: {str(e)}")
        state["next_step"] = "end"  # Safe fallback
        if execution_id:
            execution_tracker.fail_node(execution_id, "sta::assess_risk", str(e))
    
    return state

def decide_routing(state: SafetyAgentState) -> str:
    """Conditional edge: Route based on risk level and next_step."""
    execution_id = state.get("execution_id")
    
    next_step = state.get("next_step", "end")
    severity = state.get("severity", "low")
    
    # Track edge decision
    if execution_id:
        execution_tracker.trigger_edge(
            execution_id, 
            f"sta::decide_routing->{next_step}",
            condition_result=True
        )
    
    logger.info(f"STA routing decision: {next_step} (severity: {severity})")
    
    # High/critical always escalate to SDA
    if severity in ("high", "critical"):
        return "escalate_sda"
    
    # Moderate routes to SCA if needed
    if next_step == "sca":
        return "route_sca"
    
    return "end"

# Build the graph
def create_sta_graph(db: AsyncSession) -> StateGraph:
    """Create the STA LangGraph state machine."""
    
    workflow = StateGraph(SafetyAgentState)
    
    # Add nodes
    workflow.add_node("ingest_message", ingest_message_node)
    workflow.add_node("apply_redaction", lambda state: apply_redaction_node(state, db))
    workflow.add_node("assess_risk", lambda state: assess_risk_node(state, db))
    
    # Define edges
    workflow.set_entry_point("ingest_message")
    workflow.add_edge("ingest_message", "apply_redaction")
    workflow.add_edge("apply_redaction", "assess_risk")
    
    # Conditional routing from assess_risk
    workflow.add_conditional_edges(
        "assess_risk",
        decide_routing,
        {
            "escalate_sda": END,  # Will be handled by orchestrator
            "route_sca": END,     # Will be handled by orchestrator
            "end": END
        }
    )
    
    return workflow.compile()
```

### 2.2 Integrate STA Graph into Service

**File:** `backend/app/agents/sta/sta_graph_service.py`

```python
"""Service wrapper for STA LangGraph execution."""
from typing import Dict, Any
from sqlalchemy.ext.asyncio import AsyncSession
from app.agents.sta.sta_graph import create_sta_graph
from app.agents.graph_state import SafetyAgentState
from app.agents.execution_tracker import execution_tracker
from uuid import uuid4
import logging

logger = logging.getLogger(__name__)

class STAGraphService:
    """Execute STA workflow via LangGraph."""
    
    def __init__(self, db: AsyncSession):
        self.db = db
        self.graph = create_sta_graph(db)
    
    async def execute(
        self,
        user_id: int,
        session_id: str,
        user_hash: str,
        message: str,
        conversation_id: int
    ) -> SafetyAgentState:
        """Execute STA graph workflow."""
        
        # Start execution tracking
        execution_id = execution_tracker.start_execution(
            graph_id="sta",
            agent_name="Safety Triage Agent",
            input_data={"message": message, "user_hash": user_hash}
        )
        
        # Initialize state
        initial_state: SafetyAgentState = {
            "user_id": user_id,
            "session_id": session_id,
            "user_hash": user_hash,
            "message": message,
            "conversation_id": conversation_id,
            "execution_id": execution_id,
            "errors": [],
            "execution_path": [],
            "should_intervene": False,
            "case_created": False
        }
        
        try:
            # Execute graph
            logger.info(f"Executing STA graph for user_hash={user_hash}")
            final_state = await self.graph.ainvoke(initial_state)
            
            execution_tracker.complete_execution(execution_id, success=len(final_state.get("errors", [])) == 0)
            
            return final_state
            
        except Exception as e:
            logger.error(f"STA graph execution failed: {e}", exc_info=True)
            execution_tracker.complete_execution(execution_id, success=False)
            raise
```

### 2.3 Add Graph Endpoint

**File:** `backend/app/routes/agents_graph.py` (new)

```python
"""LangGraph agent execution endpoints."""
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from app.database import get_async_db
from app.agents.sta.sta_graph_service import STAGraphService
from pydantic import BaseModel

router = APIRouter(prefix="/api/v1/agents/graph", tags=["Agent Graphs"])

class STAGraphRequest(BaseModel):
    user_id: int
    session_id: str
    user_hash: str
    message: str
    conversation_id: int

@router.post("/sta/execute")
async def execute_sta_graph(
    request: STAGraphRequest,
    db: AsyncSession = Depends(get_async_db)
):
    """Execute STA workflow via LangGraph."""
    try:
        service = STAGraphService(db)
        result = await service.execute(
            user_id=request.user_id,
            session_id=request.session_id,
            user_hash=request.user_hash,
            message=request.message,
            conversation_id=request.conversation_id
        )
        
        return {
            "success": len(result.get("errors", [])) == 0,
            "execution_path": result.get("execution_path", []),
            "risk_assessment": {
                "risk_level": result.get("risk_level"),
                "risk_score": result.get("risk_score"),
                "severity": result.get("severity"),
                "intent": result.get("intent"),
                "next_step": result.get("next_step")
            },
            "triage_assessment_id": result.get("triage_assessment_id"),
            "errors": result.get("errors", [])
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

---

## Phase 3: Build Multi-Agent Orchestrator (3-4 days)

### 3.1 Create Master Orchestrator Graph

**File:** `backend/app/agents/orchestrator_graph.py`

```python
"""Master orchestrator graph coordinating STA → SCA → SDA."""
from langgraph.graph import StateGraph, END
from app.agents.graph_state import SafetyAgentState
from app.agents.sta.sta_graph import create_sta_graph
from app.agents.sca.sca_graph import create_sca_graph  # To be created
from app.agents.sda.sda_graph import create_sda_graph  # To be created
from app.agents.execution_tracker import execution_tracker
from sqlalchemy.ext.asyncio import AsyncSession
import logging

logger = logging.getLogger(__name__)

async def execute_sta_subgraph(state: SafetyAgentState, db: AsyncSession) -> SafetyAgentState:
    """Execute STA as a subgraph."""
    execution_id = state.get("execution_id")
    if execution_id:
        execution_tracker.start_node(execution_id, "orchestrator::sta", "orchestrator")
    
    try:
        sta_graph = create_sta_graph(db)
        sta_result = await sta_graph.ainvoke(state)
        
        # Merge STA outputs into state
        state.update(sta_result)
        
        if execution_id:
            execution_tracker.complete_node(execution_id, "orchestrator::sta")
        
        logger.info(f"Orchestrator completed STA: {sta_result.get('severity')}")
    except Exception as e:
        logger.error(f"STA subgraph failed: {e}", exc_info=True)
        state["errors"].append(f"STA failed: {str(e)}")
        if execution_id:
            execution_tracker.fail_node(execution_id, "orchestrator::sta", str(e))
    
    return state

async def execute_sca_subgraph(state: SafetyAgentState, db: AsyncSession) -> SafetyAgentState:
    """Execute SCA as a subgraph."""
    execution_id = state.get("execution_id")
    if execution_id:
        execution_tracker.start_node(execution_id, "orchestrator::sca", "orchestrator")
    
    try:
        sca_graph = create_sca_graph(db)
        sca_result = await sca_graph.ainvoke(state)
        
        state.update(sca_result)
        
        if execution_id:
            execution_tracker.complete_node(execution_id, "orchestrator::sca")
        
        logger.info(f"Orchestrator completed SCA: intervention={sca_result.get('should_intervene')}")
    except Exception as e:
        logger.error(f"SCA subgraph failed: {e}", exc_info=True)
        state["errors"].append(f"SCA failed: {str(e)}")
        if execution_id:
            execution_tracker.fail_node(execution_id, "orchestrator::sca", str(e))
    
    return state

async def execute_sda_subgraph(state: SafetyAgentState, db: AsyncSession) -> SafetyAgentState:
    """Execute SDA as a subgraph."""
    execution_id = state.get("execution_id")
    if execution_id:
        execution_tracker.start_node(execution_id, "orchestrator::sda", "orchestrator")
    
    try:
        sda_graph = create_sda_graph(db)
        sda_result = await sda_graph.ainvoke(state)
        
        state.update(sda_result)
        
        if execution_id:
            execution_tracker.complete_node(execution_id, "orchestrator::sda")
        
        logger.info(f"Orchestrator completed SDA: case_id={sda_result.get('case_id')}")
    except Exception as e:
        logger.error(f"SDA subgraph failed: {e}", exc_info=True)
        state["errors"].append(f"SDA failed: {str(e)}")
        if execution_id:
            execution_tracker.fail_node(execution_id, "orchestrator::sda", str(e))
    
    return state

def should_route_to_sca(state: SafetyAgentState) -> str:
    """Conditional: Should we invoke SCA?"""
    next_step = state.get("next_step", "end")
    severity = state.get("severity", "low")
    
    # High/critical skip SCA and go straight to SDA
    if severity in ("high", "critical"):
        return "route_sda"
    
    # Moderate with next_step=sca
    if next_step == "sca":
        return "invoke_sca"
    
    return "end"

def should_route_to_sda(state: SafetyAgentState) -> str:
    """Conditional: Should we create a case in SDA?"""
    severity = state.get("severity", "low")
    
    if severity in ("high", "critical"):
        return "invoke_sda"
    
    return "end"

def create_orchestrator_graph(db: AsyncSession) -> StateGraph:
    """Create master orchestrator graph."""
    
    workflow = StateGraph(SafetyAgentState)
    
    # Add subgraph nodes
    workflow.add_node("execute_sta", lambda state: execute_sta_subgraph(state, db))
    workflow.add_node("execute_sca", lambda state: execute_sca_subgraph(state, db))
    workflow.add_node("execute_sda", lambda state: execute_sda_subgraph(state, db))
    
    # Start with STA
    workflow.set_entry_point("execute_sta")
    
    # Route after STA
    workflow.add_conditional_edges(
        "execute_sta",
        should_route_to_sca,
        {
            "invoke_sca": "execute_sca",
            "route_sda": "execute_sda",
            "end": END
        }
    )
    
    # Route after SCA
    workflow.add_conditional_edges(
        "execute_sca",
        should_route_to_sda,
        {
            "invoke_sda": "execute_sda",
            "end": END
        }
    )
    
    # SDA is always terminal
    workflow.add_edge("execute_sda", END)
    
    return workflow.compile()
```

---

## Phase 4: Testing & Validation (2-3 days)

### 4.1 Create Test Suite

**File:** `backend/tests/test_langgraph_sta.py`

```python
"""Test STA LangGraph execution."""
import pytest
from app.agents.sta.sta_graph_service import STAGraphService

@pytest.mark.asyncio
async def test_sta_crisis_detection(async_db_session):
    """Test crisis message routing."""
    service = STAGraphService(async_db_session)
    
    result = await service.execute(
        user_id=1,
        session_id="test-session",
        user_hash="test-hash-123",
        message="I really really want to die u know",
        conversation_id=1
    )
    
    assert result["risk_level"] >= 2  # High or critical
    assert result["severity"] in ("high", "critical")
    assert result["next_step"] in ("sda", "sca")
    assert "assess_risk" in result["execution_path"]

@pytest.mark.asyncio
async def test_sta_normal_message(async_db_session):
    """Test normal conversation routing."""
    service = STAGraphService(async_db_session)
    
    result = await service.execute(
        user_id=1,
        session_id="test-session",
        user_hash="test-hash-123",
        message="I'm feeling okay today, just checking in",
        conversation_id=1
    )
    
    assert result["risk_level"] == 0  # Low risk
    assert result["severity"] == "low"
    assert result["next_step"] == "end"
```

### 4.2 Manual Testing via Admin Dashboard

Your existing `LangGraphViewer.tsx` will show:

- **Real-time node execution** (nodes light up as they execute)
- **Edge transitions** (conditional edges animate)
- **Execution metrics** (timing, success/failure)
- **WebSocket updates** from `execution_tracker`

Test by calling:

```bash
curl -X POST http://localhost:8000/api/v1/agents/graph/sta/execute \
  -H "Content-Type: application/json" \
  -d '{
    "user_id": 1,
    "session_id": "test-123",
    "user_hash": "hash-123",
    "message": "I want to die",
    "conversation_id": 1
  }'
```

Watch the admin dashboard at `/admin/langgraph` to see graph execution in real-time!

---

## Phase 5: Complete SCA & SDA Graphs (3-4 days)

### 5.1 SCA Graph Structure

**File:** `backend/app/agents/sca/sca_graph.py`

```python
"""LangGraph state machine for Support Coach Agent."""
# Similar structure to STA:
# Nodes: ingest_triage_signal → retrieve_history → compose_plan → safety_review → schedule_followup
# Uses existing SupportCoachService
```

### 5.2 SDA Graph Structure

**File:** `backend/app/agents/sda/sda_graph.py`

```python
"""LangGraph state machine for Service Desk Agent."""
# Nodes: ingest_case → assign_counsellor → monitor_sla → capture_updates
# Uses existing AgentOrchestrator case creation logic
```

---

## Phase 6: Thesis Integration (1-2 days)

### 6.1 Update Chapter 3 Section 3.4.2

**Replace current text with:**

> "The Safety Agent Suite is orchestrated using **LangGraph**, a framework for building stateful, graph-based agent workflows. Each agent (STA, SCA, SDA) is implemented as a compiled StateGraph with nodes representing processing steps and conditional edges defining routing logic.
>
> The master orchestrator coordinates agent execution as follows:
>
> 1. **STA (Safety Triage Agent)** executes first, analyzing user messages for crisis signals via a hybrid rule-based/ML classifier
> 2. Conditional routing based on risk severity:
>    - High/Critical → Direct escalation to SDA
>    - Moderate + support needed → Route to SCA
>    - Low → Conversation continues normally
> 3. **SCA (Support Coach Agent)** generates personalized intervention plans when invoked
> 4. **SDA (Service Desk Agent)** creates cases for high-severity situations
>
> All executions are tracked via `ExecutionStateTracker`, which persists node/edge execution data to PostgreSQL and streams real-time updates via WebSocket to the admin dashboard. Graph specifications are defined in `safety_graph_specs.py` and consumed by both the execution layer (LangGraph) and visualization layer (React Flow)."

### 6.2 Update RQ2 Evaluation

**New RQ2:** "What is the reliability of LangGraph orchestration as measured by node execution success rate and conditional edge accuracy?"

**Metrics:**

- Node execution success rate across 200 test conversations
- Conditional edge routing accuracy (does STA→SDA trigger for all high/critical cases?)
- Average execution time per agent graph
- Error recovery patterns (graceful degradation when SCA fails)

---

## Implementation Timeline

| Phase | Duration | Milestone |
|-------|----------|-----------|
| Phase 1: Setup | 1-2 days | Dependencies installed, base state schema created |
| Phase 2: STA Graph | 2-3 days | STA LangGraph functional, tested via endpoint |
| Phase 3: Orchestrator | 3-4 days | STA→SCA→SDA workflow operational |
| Phase 4: Testing | 2-3 days | Test suite passing, admin dashboard validates |
| Phase 5: SCA+SDA | 3-4 days | All 3 agents orchestrated via LangGraph |
| Phase 6: Thesis | 1-2 days | Chapter 3 rewritten, RQ2 redefined |
| **Total** | **12-18 days** | **Full LangGraph implementation** |

---

## Critical Success Factors

1. **Preserve Existing Services**: Your STA classifier, SCA intervention generator, and SDA case manager remain unchanged. LangGraph wraps them as nodes.

2. **Leverage Execution Tracker**: You already built the monitoring infrastructure! Just call `execution_tracker.start_node()` / `.complete_node()` inside graph nodes.

3. **Database Models Ready**: Your `LangGraphExecution`, `LangGraphNodeExecution`, `LangGraphEdgeExecution` models are perfect. No schema changes needed.

4. **Admin Dashboard Works**: The React Flow viewer already consumes graph specs and WebSocket updates. It will visualize LangGraph execution automatically.

---

## What This Fixes in Your Thesis

✅ **Architecture-Implementation Mismatch**: Thesis now accurately describes LangGraph orchestration  
✅ **RQ2 Evaluation**: Can measure graph execution reliability, conditional routing accuracy  
✅ **Academic Rigor**: No longer claiming unimplemented features  
✅ **Visual Proof**: Admin dashboard shows actual graph execution, not just specs  

---

## Next Steps

1. **Start with Phase 1**: Install `langgraph` and create `graph_state.py`
2. **Build STA Graph First**: Get one agent working end-to-end before orchestration
3. **Test Extensively**: Use your existing crisis test cases ("I want to die", etc.)
4. **Iterate**: Once STA works, orchestrator becomes straightforward

Your infrastructure is **thesis-defense ready**. You just need to connect the execution layer!
