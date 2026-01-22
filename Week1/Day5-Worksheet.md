# Day 5 Worksheet: Agent Lifecycle and State Management

## Objectives

- Implement agent lifecycle management
- Build state persistence with database
- Create execution tracking system
- Design recovery mechanisms
- Complete Week 1 synthesis project

## Prerequisites Checklist

Before starting, ensure you have:

- [ ] Completed Days 1-4 exercises
- [ ] Understanding of all previous patterns
- [ ] SQLite or database available
- [ ] All previous code committed

## Exercise 1: Agent Lifecycle Management

### Step 1: Implement Lifecycle Manager

Create `lifecycle/lifecycle_manager.py`:

```python
from enum import Enum
from datetime import datetime
from typing import Dict, Any, Optional, List

class AgentState(Enum):
    CREATED = "created"
    INITIALIZED = "initialized"
    ACTIVE = "active"
    PAUSED = "paused"
    TERMINATED = "terminated"
    ERROR = "error"

class LifecycleManager:
    """Manages agent lifecycle (inspired by Isorythm patterns)"""
    
    def __init__(self):
        self.agents: Dict[str, Dict[str, Any]] = {}
        self.state_history: Dict[str, List[Dict[str, Any]]] = {}
    
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
            "activated_at": None,
            "terminated_at": None
        }
        
        self.agents[agent_id] = agent_record
        self._record_state_change(agent_id, AgentState.CREATED, "Agent created")
        
        return agent_id
    
    def initialize_agent(self, agent_id: str) -> bool:
        """Initialize agent"""
        if agent_id not in self.agents:
            return False
        
        agent = self.agents[agent_id]
        if agent["state"] != AgentState.CREATED:
            return False
        
        agent["state"] = AgentState.INITIALIZED
        agent["initialized_at"] = datetime.now()
        self._record_state_change(agent_id, AgentState.INITIALIZED, "Agent initialized")
        
        return True
    
    def activate_agent(self, agent_id: str) -> bool:
        """Activate agent"""
        if agent_id not in self.agents:
            return False
        
        agent = self.agents[agent_id]
        if agent["state"] not in [AgentState.INITIALIZED, AgentState.PAUSED]:
            return False
        
        agent["state"] = AgentState.ACTIVE
        agent["activated_at"] = datetime.now()
        self._record_state_change(agent_id, AgentState.ACTIVE, "Agent activated")
        
        return True
    
    def pause_agent(self, agent_id: str) -> bool:
        """Pause agent"""
        if agent_id not in self.agents:
            return False
        
        agent = self.agents[agent_id]
        if agent["state"] != AgentState.ACTIVE:
            return False
        
        agent["state"] = AgentState.PAUSED
        self._record_state_change(agent_id, AgentState.PAUSED, "Agent paused")
        
        return True
    
    def terminate_agent(self, agent_id: str) -> bool:
        """Terminate agent"""
        if agent_id not in self.agents:
            return False
        
        agent = self.agents[agent_id]
        agent["state"] = AgentState.TERMINATED
        agent["terminated_at"] = datetime.now()
        self._record_state_change(agent_id, AgentState.TERMINATED, "Agent terminated")
        
        return True
    
    def _record_state_change(self, agent_id: str, new_state: AgentState, reason: str):
        """Record state change in history"""
        if agent_id not in self.state_history:
            self.state_history[agent_id] = []
        
        self.state_history[agent_id].append({
            "state": new_state.value,
            "timestamp": datetime.now(),
            "reason": reason
        })
    
    def get_agent_state(self, agent_id: str) -> Optional[Dict[str, Any]]:
        """Get current agent state"""
        return self.agents.get(agent_id)
    
    def get_state_history(self, agent_id: str) -> List[Dict[str, Any]]:
        """Get state change history"""
        return self.state_history.get(agent_id, [])
```

### Deliverable 1.1: Lifecycle Management

- [ ] Lifecycle manager implemented
- [ ] All state transitions working
- [ ] State history tracking
- [ ] Tests passing

**Commit your work:**
```bash
git add .
git commit -m "Implement agent lifecycle management"
```

## Exercise 2: State Persistence with Database

### Step 1: Create Database Models

Create `persistence/models.py`:

```python
from sqlalchemy import create_engine, Column, Integer, String, DateTime, JSON, ForeignKey
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, relationship

Base = declarative_base()

class AgentStateModel(Base):
    """Agent state persistence (inspired by Isorythm)"""
    __tablename__ = "agent_states"
    
    id = Column(Integer, primary_key=True)
    agent_id = Column(String, unique=True, index=True)
    state = Column(String)  # created, initialized, active, paused, terminated
    state_data = Column(JSON)
    created_at = Column(DateTime)
    updated_at = Column(DateTime)

class ExecutionModel(Base):
    """Execution tracking (inspired by Isorythm's WorkflowRun/TaskRun)"""
    __tablename__ = "executions"
    
    id = Column(Integer, primary_key=True)
    agent_id = Column(String, ForeignKey("agent_states.agent_id"))
    task_id = Column(String)
    status = Column(String)  # running, completed, failed
    started_at = Column(DateTime)
    completed_at = Column(DateTime)
    result = Column(JSON)
    error = Column(String)
    
    # Relationships
    events = relationship("ExecutionEventModel", back_populates="execution")

class ExecutionEventModel(Base):
    """Execution events (inspired by Isorythm's RunEvent)"""
    __tablename__ = "execution_events"
    
    id = Column(Integer, primary_key=True)
    execution_id = Column(Integer, ForeignKey("executions.id"))
    event_type = Column(String)  # step_started, step_completed, error, etc.
    event_data = Column(JSON)
    timestamp = Column(DateTime)
    
    # Relationships
    execution = relationship("ExecutionModel", back_populates="events")

# Database setup
def create_database(db_path: str = "agent_system.db"):
    """Create database and tables"""
    engine = create_engine(f"sqlite:///{db_path}")
    Base.metadata.create_all(engine)
    return engine

def create_session(engine):
    """Create database session"""
    Session = sessionmaker(bind=engine)
    return Session()
```

### Step 2: Implement State Manager

Create `persistence/state_manager.py`:

```python
from persistence.models import (
    AgentStateModel, ExecutionModel, ExecutionEventModel,
    create_database, create_session
)
from typing import Dict, Any, Optional, List
from datetime import datetime

class StateManager:
    """Manages state persistence (inspired by Isorythm)"""
    
    def __init__(self, db_path: str = "agent_system.db"):
        self.engine = create_database(db_path)
        self.Session = create_session(self.engine)
        self.cache: Dict[str, Any] = {}
    
    def save_agent_state(self, agent_id: str, state: str, state_data: Dict[str, Any]):
        """Save agent state"""
        session = self.Session()
        try:
            agent_state = session.query(AgentStateModel).filter_by(agent_id=agent_id).first()
            
            if agent_state:
                agent_state.state = state
                agent_state.state_data = state_data
                agent_state.updated_at = datetime.now()
            else:
                agent_state = AgentStateModel(
                    agent_id=agent_id,
                    state=state,
                    state_data=state_data,
                    created_at=datetime.now(),
                    updated_at=datetime.now()
                )
                session.add(agent_state)
            
            session.commit()
            self.cache[agent_id] = {"state": state, "data": state_data}
        finally:
            session.close()
    
    def load_agent_state(self, agent_id: str) -> Optional[Dict[str, Any]]:
        """Load agent state"""
        # Check cache
        if agent_id in self.cache:
            return self.cache[agent_id]
        
        # Load from database
        session = self.Session()
        try:
            agent_state = session.query(AgentStateModel).filter_by(agent_id=agent_id).first()
            if agent_state:
                state = {
                    "state": agent_state.state,
                    "data": agent_state.state_data,
                    "created_at": agent_state.created_at.isoformat() if agent_state.created_at else None,
                    "updated_at": agent_state.updated_at.isoformat() if agent_state.updated_at else None
                }
                self.cache[agent_id] = state
                return state
        finally:
            session.close()
        
        return None
    
    def start_execution(
        self,
        agent_id: str,
        task_id: str
    ) -> int:
        """Start execution tracking"""
        session = self.Session()
        try:
            execution = ExecutionModel(
                agent_id=agent_id,
                task_id=task_id,
                status="running",
                started_at=datetime.now()
            )
            session.add(execution)
            session.commit()
            return execution.id
        finally:
            session.close()
    
    def log_event(
        self,
        execution_id: int,
        event_type: str,
        event_data: Dict[str, Any]
    ):
        """Log execution event"""
        session = self.Session()
        try:
            event = ExecutionEventModel(
                execution_id=execution_id,
                event_type=event_type,
                event_data=event_data,
                timestamp=datetime.now()
            )
            session.add(event)
            session.commit()
        finally:
            session.close()
    
    def complete_execution(
        self,
        execution_id: int,
        result: Dict[str, Any],
        status: str = "completed",
        error: Optional[str] = None
    ):
        """Complete execution"""
        session = self.Session()
        try:
            execution = session.query(ExecutionModel).filter_by(id=execution_id).first()
            if execution:
                execution.status = status
                execution.completed_at = datetime.now()
                execution.result = result
                execution.error = error
                session.commit()
        finally:
            session.close()
    
    def get_execution_history(self, agent_id: str) -> List[Dict[str, Any]]:
        """Get execution history"""
        session = self.Session()
        try:
            executions = session.query(ExecutionModel).filter_by(agent_id=agent_id).all()
            return [
                {
                    "id": e.id,
                    "task_id": e.task_id,
                    "status": e.status,
                    "started_at": e.started_at.isoformat() if e.started_at else None,
                    "completed_at": e.completed_at.isoformat() if e.completed_at else None,
                    "result": e.result,
                    "error": e.error
                }
                for e in executions
            ]
        finally:
            session.close()
```

### Deliverable 2.1: State Persistence

- [ ] Database models created
- [ ] State manager implemented
- [ ] State save/load working
- [ ] Execution tracking working
- [ ] Tests passing

**Commit your work:**
```bash
git add .
git commit -m "Implement state persistence with database"
```

## Exercise 3: Execution Tracking System

### Step 1: Create Execution Tracker

Create `tracking/execution_tracker.py`:

```python
from persistence.state_manager import StateManager
from typing import Dict, Any, List, Optional
from datetime import datetime

class ExecutionTracker:
    """Tracks agent execution (inspired by Isorythm's RunEvent system)"""
    
    def __init__(self, state_manager: StateManager):
        self.state_manager = state_manager
        self.active_executions: Dict[str, int] = {}  # agent_id -> execution_id
    
    def start_tracking(
        self,
        agent_id: str,
        task_id: str,
        task_description: str
    ) -> int:
        """Start tracking execution"""
        execution_id = self.state_manager.start_execution(agent_id, task_id)
        self.active_executions[agent_id] = execution_id
        
        self.state_manager.log_event(
            execution_id,
            "execution_started",
            {
                "agent_id": agent_id,
                "task_id": task_id,
                "description": task_description
            }
        )
        
        return execution_id
    
    def log_step(
        self,
        agent_id: str,
        step_number: int,
        step_description: str,
        step_result: Dict[str, Any]
    ):
        """Log execution step"""
        execution_id = self.active_executions.get(agent_id)
        if execution_id:
            self.state_manager.log_event(
                execution_id,
                "step_completed",
                {
                    "step_number": step_number,
                    "description": step_description,
                    "result": step_result
                }
            )
    
    def log_error(
        self,
        agent_id: str,
        error: str,
        error_context: Dict[str, Any]
    ):
        """Log execution error"""
        execution_id = self.active_executions.get(agent_id)
        if execution_id:
            self.state_manager.log_event(
                execution_id,
                "error",
                {
                    "error": error,
                    "context": error_context
                }
            )
    
    def complete_tracking(
        self,
        agent_id: str,
        result: Dict[str, Any],
        status: str = "completed",
        error: Optional[str] = None
    ):
        """Complete execution tracking"""
        execution_id = self.active_executions.get(agent_id)
        if execution_id:
            self.state_manager.complete_execution(
                execution_id,
                result,
                status,
                error
            )
            self.active_executions.pop(agent_id, None)
    
    def get_agent_history(self, agent_id: str) -> List[Dict[str, Any]]:
        """Get execution history for agent"""
        return self.state_manager.get_execution_history(agent_id)
```

### Deliverable 3.1: Execution Tracking

- [ ] Execution tracker implemented
- [ ] Step logging working
- [ ] Error logging working
- [ ] History retrieval working
- [ ] Tests passing

**Commit your work:**
```bash
git add .
git commit -m "Implement execution tracking system"
```

## Exercise 4: Recovery Mechanisms

### Step 1: Create Recovery Manager

Create `recovery/recovery_manager.py`:

```python
from persistence.state_manager import StateManager
from tracking.execution_tracker import ExecutionTracker
from typing import Dict, Any, Optional
from datetime import datetime

class RecoveryManager:
    """Manages recovery from failures (inspired by Isorythm reliability patterns)"""
    
    def __init__(
        self,
        state_manager: StateManager,
        execution_tracker: ExecutionTracker
    ):
        self.state_manager = state_manager
        self.execution_tracker = execution_tracker
        self.checkpoints: Dict[str, Dict[str, Any]] = {}
    
    def create_checkpoint(self, agent_id: str, checkpoint_name: str = "auto") -> str:
        """Create recovery checkpoint"""
        state = self.state_manager.load_agent_state(agent_id)
        if not state:
            return None
        
        checkpoint_id = f"{agent_id}-{checkpoint_name}-{datetime.now().timestamp()}"
        
        checkpoint = {
            "checkpoint_id": checkpoint_id,
            "agent_id": agent_id,
            "state": state,
            "created_at": datetime.now().isoformat()
        }
        
        self.checkpoints[checkpoint_id] = checkpoint
        
        # Save to database (simplified - in production would use Checkpoint model)
        self.state_manager.save_agent_state(
            f"checkpoint-{checkpoint_id}",
            "checkpoint",
            checkpoint
        )
        
        return checkpoint_id
    
    def restore_from_checkpoint(self, checkpoint_id: str) -> Dict[str, Any]:
        """Restore agent from checkpoint"""
        checkpoint = self.checkpoints.get(checkpoint_id)
        if not checkpoint:
            # Try to load from database
            checkpoint_state = self.state_manager.load_agent_state(f"checkpoint-{checkpoint_id}")
            if checkpoint_state:
                checkpoint = checkpoint_state["data"]
        
        if checkpoint:
            agent_id = checkpoint["agent_id"]
            state = checkpoint["state"]
            
            # Restore state
            self.state_manager.save_agent_state(
                agent_id,
                state["state"],
                state["data"]
            )
            
            return {
                "status": "restored",
                "agent_id": agent_id,
                "checkpoint_id": checkpoint_id,
                "restored_at": datetime.now().isoformat()
            }
        
        return {
            "status": "checkpoint_not_found",
            "checkpoint_id": checkpoint_id
        }
    
    def recover_agent(self, agent_id: str) -> Dict[str, Any]:
        """Recover agent from failure"""
        # Load current state
        state = self.state_manager.load_agent_state(agent_id)
        if not state:
            return {"status": "no_state", "recovered": False}
        
        # Check execution history for failures
        history = self.execution_tracker.get_agent_history(agent_id)
        if not history:
            return {"status": "no_history", "recovered": False}
        
        last_execution = history[-1]
        
        if last_execution["status"] == "failed":
            # Attempt recovery
            return self._recover_from_failure(agent_id, last_execution, state)
        
        return {"status": "no_failure", "recovered": True}
    
    def _recover_from_failure(
        self,
        agent_id: str,
        failed_execution: Dict[str, Any],
        state: Dict[str, Any]
    ) -> Dict[str, Any]:
        """Recover from specific failure"""
        error = failed_execution.get("error", "unknown")
        
        recovery_strategy = self._determine_recovery_strategy(error)
        
        return {
            "status": "recovery_attempted",
            "agent_id": agent_id,
            "strategy": recovery_strategy,
            "error": error,
            "recovered": recovery_strategy != "manual_intervention"
        }
    
    def _determine_recovery_strategy(self, error: str) -> str:
        """Determine recovery strategy based on error"""
        error_lower = error.lower()
        
        if "timeout" in error_lower:
            return "retry_with_increased_timeout"
        elif "resource" in error_lower or "memory" in error_lower:
            return "wait_and_retry"
        elif "network" in error_lower or "connection" in error_lower:
            return "retry_with_backoff"
        else:
            return "manual_intervention"
```

### Deliverable 4.1: Recovery Mechanisms

- [ ] Recovery manager implemented
- [ ] Checkpoint creation working
- [ ] Checkpoint restoration working
- [ ] Failure recovery logic
- [ ] Tests passing

**Commit your work:**
```bash
git add .
git commit -m "Implement recovery mechanisms"
```

## Exercise 5: Week 1 Synthesis Project

### Step 1: Integrate All Components

Create a complete system that integrates:
- Agent lifecycle management
- State persistence
- Execution tracking
- Recovery mechanisms
- All previous patterns (communication, coordination, bounded autonomy, human-in-the-loop)

### Step 2: Create System Documentation

Document your complete system:
- Architecture overview
- Component descriptions
- Integration patterns
- Usage examples
- Testing approach

### Deliverable 5.1: Week 1 Synthesis

- [ ] All components integrated
- [ ] Complete system working
- [ ] Documentation complete
- [ ] Tests comprehensive
- [ ] Code committed

**Commit your work:**
```bash
git add .
git commit -m "Week 1 synthesis: Complete multi-agent system"
```

## Reflection Questions

1. **How does agent lifecycle management** improve system reliability?
2. **What are the trade-offs** between in-memory and database state persistence?
3. **How does execution tracking** help with debugging and analysis?
4. **What recovery strategies** are most effective for different failure types?
5. **How do Isorythm's patterns** compare to your implementation?
6. **What would you improve** in your complete system?

## Troubleshooting

### Issue: Database connection errors
- **Solution**: Check database path and permissions
- **Check**: SQLAlchemy engine configuration

### Issue: State not persisting
- **Solution**: Verify commit() is called
- **Check**: Transaction handling

### Issue: Execution tracking missing events
- **Solution**: Ensure all code paths log events
- **Check**: Error handling includes logging

## Next Steps

- Review all Week 1 materials
- Complete synthesis project
- Commit all work to Git
- Prepare for Week 2: Agent Architecture Patterns

## Deliverables Checklist

- [ ] Agent lifecycle management implemented
- [ ] State persistence with database working
- [ ] Execution tracking system complete
- [ ] Recovery mechanisms implemented
- [ ] Week 1 synthesis project complete
- [ ] All code committed to Git repository
- [ ] Documentation complete
- [ ] Ready for Week 2

---

**Congratulations!** You've completed Week 1 and built a complete multi-agent system. Next week we'll explore advanced architecture patterns.

