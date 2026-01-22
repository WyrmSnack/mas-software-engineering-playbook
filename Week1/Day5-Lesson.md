# Day 5: Agent Lifecycle and State Management

## Learning Objectives

By the end of this lesson, you will be able to:
- Understand agent lifecycle (creation, execution, termination)
- Implement state persistence patterns
- Build execution tracking and history systems
- Design recovery and fault tolerance mechanisms
- Apply Isorythm's database model patterns
- Handle agent state across system restarts

## Agent Lifecycle

Agents go through distinct lifecycle phases that must be managed properly for reliable operation.

### Lifecycle Phases

1. **Creation**: Agent instantiation and initialization
2. **Initialization**: Setup and configuration
3. **Active**: Agent executing tasks
4. **Paused**: Temporarily suspended
5. **Terminated**: Agent stopped and cleaned up

### Implementation Pattern

```python
from enum import Enum
from datetime import datetime
from typing import Dict, Any, Optional

class AgentState(Enum):
    CREATED = "created"
    INITIALIZED = "initialized"
    ACTIVE = "active"
    PAUSED = "paused"
    TERMINATED = "terminated"
    ERROR = "error"

class LifecycleManager:
    """Manages agent lifecycle"""
    
    def __init__(self):
        self.agents: Dict[str, Dict[str, Any]] = {}
    
    def create_agent(
        self,
        agent_id: str,
        agent_type: str,
        config: Dict[str, Any]
    ) -> str:
        """Create new agent"""
        agent_record = {
            "agent_id": agent_id,
            "type": agent_type,
            "state": AgentState.CREATED,
            "config": config,
            "created_at": datetime.now(),
            "initialized_at": None,
            "terminated_at": None
        }
        
        self.agents[agent_id] = agent_record
        return agent_id
    
    def initialize_agent(self, agent_id: str) -> bool:
        """Initialize agent"""
        if agent_id not in self.agents:
            return False
        
        agent = self.agents[agent_id]
        agent["state"] = AgentState.INITIALIZED
        agent["initialized_at"] = datetime.now()
        return True
    
    def activate_agent(self, agent_id: str) -> bool:
        """Activate agent"""
        if agent_id not in self.agents:
            return False
        
        agent = self.agents[agent_id]
        if agent["state"] not in [AgentState.INITIALIZED, AgentState.PAUSED]:
            return False
        
        agent["state"] = AgentState.ACTIVE
        return True
    
    def pause_agent(self, agent_id: str) -> bool:
        """Pause agent"""
        if agent_id not in self.agents:
            return False
        
        agent = self.agents[agent_id]
        if agent["state"] != AgentState.ACTIVE:
            return False
        
        agent["state"] = AgentState.PAUSED
        return True
    
    def terminate_agent(self, agent_id: str) -> bool:
        """Terminate agent"""
        if agent_id not in self.agents:
            return False
        
        agent = self.agents[agent_id]
        agent["state"] = AgentState.TERMINATED
        agent["terminated_at"] = datetime.now()
        return True
```

**Isorythm Pattern**: Isorythm tracks agent lifecycle through database models, with agents existing as persistent entities that can be referenced across workflow runs.

## State Persistence

**Research Finding**: Reliability patterns from production deployments emphasize the importance of state persistence for recovery and observability.

### State Persistence Patterns

1. **In-Memory**: Fast but lost on restart
2. **Database**: Persistent across restarts
3. **File System**: Simple persistence
4. **Hybrid**: Critical state in DB, cache in memory

### Isorythm's Database Model Pattern

Isorythm uses SQLAlchemy models for state persistence:

**Key Models**:
- **WorkflowRun**: Tracks workflow execution
- **TaskRun**: Tracks individual task execution
- **RunEvent**: Structured execution events/logs

**Pattern**:
```python
# Isorythm-inspired database models
from sqlalchemy import Column, Integer, String, DateTime, JSON, ForeignKey
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship

Base = declarative_base()

class WorkflowRun(Base):
    """Workflow execution tracking"""
    __tablename__ = "workflow_runs"
    
    id = Column(Integer, primary_key=True)
    workflow_id = Column(Integer, ForeignKey("workflows.id"))
    status = Column(String)  # pending, running, completed, failed
    started_at = Column(DateTime)
    completed_at = Column(DateTime)
    result = Column(JSON)
    
    # Relationships
    task_runs = relationship("TaskRun", back_populates="workflow_run")

class TaskRun(Base):
    """Individual task execution tracking"""
    __tablename__ = "task_runs"
    
    id = Column(Integer, primary_key=True)
    workflow_run_id = Column(Integer, ForeignKey("workflow_runs.id"))
    task_id = Column(Integer, ForeignKey("tasks.id"))
    agent_id = Column(Integer, ForeignKey("agents.id"))
    status = Column(String)
    started_at = Column(DateTime)
    completed_at = Column(DateTime)
    result = Column(JSON)
    error = Column(String)
    
    # Relationships
    workflow_run = relationship("WorkflowRun", back_populates="task_runs")
    events = relationship("RunEvent", back_populates="task_run")

class RunEvent(Base):
    """Structured execution events"""
    __tablename__ = "run_events"
    
    id = Column(Integer, primary_key=True)
    task_run_id = Column(Integer, ForeignKey("task_runs.id"))
    event_type = Column(String)  # step_started, step_completed, error, etc.
    event_data = Column(JSON)
    timestamp = Column(DateTime)
    
    # Relationships
    task_run = relationship("TaskRun", back_populates="events")
```

### Implementation Pattern

```python
class StateManager:
    """Manages agent state persistence"""
    
    def __init__(self, db_session):
        self.db = db_session
        self.cache: Dict[str, Any] = {}  # In-memory cache
    
    def save_agent_state(self, agent_id: str, state: Dict[str, Any]):
        """Save agent state to database"""
        # Save to database
        agent_state = AgentState(
            agent_id=agent_id,
            state_data=state,
            updated_at=datetime.now()
        )
        self.db.merge(agent_state)
        self.db.commit()
        
        # Update cache
        self.cache[agent_id] = state
    
    def load_agent_state(self, agent_id: str) -> Optional[Dict[str, Any]]:
        """Load agent state from database"""
        # Check cache first
        if agent_id in self.cache:
            return self.cache[agent_id]
        
        # Load from database
        agent_state = self.db.query(AgentState).filter_by(agent_id=agent_id).first()
        if agent_state:
            state = agent_state.state_data
            self.cache[agent_id] = state
            return state
        
        return None
    
    def clear_agent_state(self, agent_id: str):
        """Clear agent state"""
        self.db.query(AgentState).filter_by(agent_id=agent_id).delete()
        self.db.commit()
        self.cache.pop(agent_id, None)
```

## Execution Tracking and History

**Isorythm Pattern**: Isorythm tracks execution through WorkflowRun, TaskRun, and RunEvent models, providing complete execution history.

### Execution Tracking

```python
class ExecutionTracker:
    """Tracks agent execution history"""
    
    def __init__(self, db_session):
        self.db = db_session
    
    def start_execution(self, workflow_id: str, agent_id: str, task_id: str) -> int:
        """Start execution tracking"""
        execution = Execution(
            workflow_id=workflow_id,
            agent_id=agent_id,
            task_id=task_id,
            status="running",
            started_at=datetime.now()
        )
        self.db.add(execution)
        self.db.commit()
        return execution.id
    
    def log_event(
        self,
        execution_id: int,
        event_type: str,
        event_data: Dict[str, Any]
    ):
        """Log execution event"""
        event = ExecutionEvent(
            execution_id=execution_id,
            event_type=event_type,
            event_data=event_data,
            timestamp=datetime.now()
        )
        self.db.add(event)
        self.db.commit()
    
    def complete_execution(
        self,
        execution_id: int,
        result: Dict[str, Any],
        status: str = "completed"
    ):
        """Mark execution as complete"""
        execution = self.db.query(Execution).filter_by(id=execution_id).first()
        if execution:
            execution.status = status
            execution.completed_at = datetime.now()
            execution.result = result
            self.db.commit()
    
    def get_execution_history(self, agent_id: str) -> List[Dict[str, Any]]:
        """Get execution history for agent"""
        executions = self.db.query(Execution).filter_by(agent_id=agent_id).all()
        return [
            {
                "id": e.id,
                "task_id": e.task_id,
                "status": e.status,
                "started_at": e.started_at.isoformat(),
                "completed_at": e.completed_at.isoformat() if e.completed_at else None,
                "result": e.result
            }
            for e in executions
        ]
```

## Recovery and Fault Tolerance

### Recovery Patterns

1. **Checkpoint Recovery**: Restore from saved checkpoints
2. **State Reconstruction**: Rebuild state from execution history
3. **Partial Recovery**: Recover completed work, retry failed parts
4. **Compensating Actions**: Undo partial work on failure

### Implementation Pattern

```python
class RecoveryManager:
    """Manages recovery from failures"""
    
    def __init__(self, state_manager: StateManager, execution_tracker: ExecutionTracker):
        self.state_manager = state_manager
        self.execution_tracker = execution_tracker
    
    def recover_agent(self, agent_id: str) -> Dict[str, Any]:
        """Recover agent from failure"""
        # Load last known state
        state = self.state_manager.load_agent_state(agent_id)
        if not state:
            return {"status": "no_state", "recovered": False}
        
        # Check execution history
        history = self.execution_tracker.get_execution_history(agent_id)
        last_execution = history[-1] if history else None
        
        if last_execution and last_execution["status"] == "failed":
            # Attempt recovery
            return self._recover_from_failure(agent_id, last_execution, state)
        
        return {"status": "recovered", "state": state, "recovered": True}
    
    def _recover_from_failure(
        self,
        agent_id: str,
        failed_execution: Dict[str, Any],
        state: Dict[str, Any]
    ) -> Dict[str, Any]:
        """Recover from specific failure"""
        # Determine recovery strategy
        error = failed_execution.get("error", "unknown")
        
        if "timeout" in error.lower():
            # Retry with increased timeout
            return {
                "status": "retry",
                "strategy": "increase_timeout",
                "recovered": True
            }
        elif "resource" in error.lower():
            # Wait and retry
            return {
                "status": "retry",
                "strategy": "wait_and_retry",
                "recovered": True
            }
        else:
            # Manual intervention required
            return {
                "status": "manual_intervention",
                "recovered": False,
                "error": error
            }
    
    def create_checkpoint(self, agent_id: str) -> str:
        """Create recovery checkpoint"""
        state = self.state_manager.load_agent_state(agent_id)
        checkpoint_id = f"checkpoint-{agent_id}-{datetime.now().timestamp()}"
        
        checkpoint = Checkpoint(
            checkpoint_id=checkpoint_id,
            agent_id=agent_id,
            state=state,
            created_at=datetime.now()
        )
        self.state_manager.db.add(checkpoint)
        self.state_manager.db.commit()
        
        return checkpoint_id
    
    def restore_from_checkpoint(self, checkpoint_id: str) -> Dict[str, Any]:
        """Restore agent from checkpoint"""
        checkpoint = self.state_manager.db.query(Checkpoint).filter_by(
            checkpoint_id=checkpoint_id
        ).first()
        
        if checkpoint:
            self.state_manager.save_agent_state(
                checkpoint.agent_id,
                checkpoint.state
            )
            return {
                "status": "restored",
                "agent_id": checkpoint.agent_id,
                "checkpoint_id": checkpoint_id
            }
        
        return {"status": "checkpoint_not_found", "restored": False}
```

## Isorythm's Execution Tracking

**Isorythm Implementation**: Isorythm provides comprehensive execution tracking:

1. **WorkflowRun**: Tracks overall workflow execution
   - Status, timestamps, results
   - Links to task runs

2. **TaskRun**: Tracks individual task execution
   - Agent assignment, status, results
   - Links to workflow run and events

3. **RunEvent**: Structured event logging
   - Event types, data, timestamps
   - Complete execution trace

**API Endpoints**:
- `GET /workflows/{id}/runs` - Get execution history
- `GET /workflows/{id}/runs/{run_id}/tasks` - Get task runs
- WebSocket updates for real-time tracking

## State Management Best Practices

### 1. Persist Critical State

- Agent configuration
- Execution progress
- Results and outputs
- Error information

### 2. Cache Frequently Accessed State

- Current agent status
- Recent execution results
- Active task assignments

### 3. Version State

- Track state versions
- Support state migration
- Handle schema changes

### 4. Cleanup Old State

- Archive completed executions
- Remove stale checkpoints
- Manage database growth

## Key Takeaways

1. **Agent lifecycle** includes creation, initialization, active, paused, and terminated states
2. **State persistence** is critical for recovery and observability
3. **Execution tracking** provides complete history for debugging and analysis
4. **Recovery mechanisms** enable fault tolerance and system resilience
5. **Isorythm's database models** demonstrate production-ready state management
6. **Checkpoint recovery** enables restoration from failures
7. **State versioning** supports system evolution

## Next Steps

- Complete the Day 5 worksheet
- Implement state persistence
- Build execution tracking
- Create recovery mechanisms
- Complete Week 1 synthesis project
- Prepare for Week 2: Agent Architecture Patterns

## Additional Resources

- [Isorythm Patterns Guide](Resources/Isorythm-Patterns-Guide.md)
- [Production Patterns Guide](Resources/Production-Patterns.md)
- [MAS-GSE Research Findings](Resources/MAS-GSE-Research-Findings.md)

---

**Ready for hands-on practice?** Complete the [Day 5 Worksheet](Day5-Worksheet.md)!

