# Day 4 Worksheet: Building Your First Multi-Agent System

## Objectives

- Implement bounded autonomy pattern (≤10 steps)
- Add human-in-the-loop checkpoints
- Build tool registry system
- Create complete integrated system
- Test end-to-end functionality

## Prerequisites Checklist

Before starting, ensure you have:

- [ ] Completed Days 1-3 exercises
- [ ] Understanding of architectures and patterns
- [ ] All previous code committed
- [ ] Development environment ready

## Exercise 1: Implement Bounded Autonomy Agent

### Step 1: Create Bounded Autonomy Base

Create `agents/bounded_autonomy_agent.py`:

```python
from agents.base_agent import BaseAgent
from typing import Dict, Any, List, Optional
from datetime import datetime
import time

class BoundedAutonomyAgent(BaseAgent):
    """Agent with bounded autonomy (≤10 steps) - 68% of successful deployments"""
    
    MAX_STEPS = 10  # Research finding
    
    def __init__(
        self,
        agent_id: str,
        capabilities: List[str],
        max_steps: int = MAX_STEPS,
        message_bus=None
    ):
        super().__init__(agent_id, capabilities, message_bus)
        self.max_steps = max_steps
        self.step_count = 0
        self.step_history: List[Dict[str, Any]] = []
        self.checkpoint_interval = 5
    
    def execute_step(self, action: Dict[str, Any]) -> Dict[str, Any]:
        """Execute step with bounded autonomy check"""
        if self.step_count >= self.max_steps:
            return {
                "status": "escalated",
                "reason": f"Max steps ({self.max_steps}) reached",
                "step_count": self.step_count,
                "agent_id": self.agent_id
            }
        
        self.step_count += 1
        step_result = {
            "step_number": self.step_count,
            "action": action,
            "timestamp": datetime.now(),
            "status": "in_progress"
        }
        
        try:
            result = self._perform_action(action)
            step_result["status"] = "completed"
            step_result["result"] = result
        except Exception as e:
            step_result["status"] = "failed"
            step_result["error"] = str(e)
        
        self.step_history.append(step_result)
        
        # Checkpoint validation
        if self.step_count % self.checkpoint_interval == 0:
            checkpoint_result = self._checkpoint_validation()
            if not checkpoint_result["valid"]:
                return {
                    "status": "halted",
                    "reason": checkpoint_result["reason"],
                    "step_count": self.step_count
                }
        
        return step_result
    
    def _checkpoint_validation(self) -> Dict[str, Any]:
        """Validate at checkpoint"""
        # Check if recent steps are successful
        recent_steps = self.step_history[-self.checkpoint_interval:]
        failed_steps = [s for s in recent_steps if s["status"] == "failed"]
        
        if len(failed_steps) > self.checkpoint_interval // 2:
            return {
                "valid": False,
                "reason": "Too many failures at checkpoint"
            }
        
        return {"valid": True, "reason": "Checkpoint passed"}
    
    def _perform_action(self, action: Dict[str, Any]) -> Dict[str, Any]:
        """Perform action (override in subclasses)"""
        return {"result": "Action performed"}
    
    def reset_autonomy(self):
        """Reset step count (after human approval)"""
        self.step_count = 0
        print(f"[{self.agent_id}] Autonomy reset")
    
    def get_autonomy_status(self) -> Dict[str, Any]:
        """Get current autonomy status"""
        return {
            "agent_id": self.agent_id,
            "step_count": self.step_count,
            "max_steps": self.max_steps,
            "remaining_steps": self.max_steps - self.step_count,
            "autonomy_percentage": (self.step_count / self.max_steps) * 100
        }
```

### Step 2: Test Bounded Autonomy

Create `tests/test_bounded_autonomy.py`:

```python
from agents.bounded_autonomy_agent import BoundedAutonomyAgent

def test_bounded_autonomy():
    agent = BoundedAutonomyAgent("test-agent", ["test"], max_steps=5)
    
    # Execute steps
    for i in range(7):  # Try to exceed max
        result = agent.execute_step({"action": f"step-{i}"})
        print(f"Step {i+1}: {result.get('status')}")
        
        if result.get("status") == "escalated":
            print("✅ Bounded autonomy working - escalation triggered")
            break
    
    status = agent.get_autonomy_status()
    assert status["step_count"] <= 5
    print("✅ Bounded autonomy test passed")

if __name__ == "__main__":
    test_bounded_autonomy()
```

### Deliverable 1.1: Bounded Autonomy

- [ ] Bounded autonomy agent implemented
- [ ] Step counting working
- [ ] Escalation on max steps
- [ ] Checkpoint validation
- [ ] Tests passing

**Commit your work:**
```bash
git add .
git commit -m "Implement bounded autonomy pattern (≤10 steps)"
```

## Exercise 2: Human-in-the-Loop Checkpoints

### Step 1: Create Human Checkpoint System

Create `coordination/human_checkpoint.py`:

```python
from typing import Dict, Any, Optional, Callable
from dataclasses import dataclass
from datetime import datetime
from enum import Enum

class ApprovalStatus(Enum):
    PENDING = "pending"
    APPROVED = "approved"
    REJECTED = "rejected"
    EXPIRED = "expired"

@dataclass
class ApprovalRequest:
    request_id: str
    context: Dict[str, Any]
    approval_type: str
    status: ApprovalStatus = ApprovalStatus.PENDING
    timestamp: datetime = None
    approved_by: Optional[str] = None
    response_time: Optional[float] = None
    
    def __post_init__(self):
        if self.timestamp is None:
            self.timestamp = datetime.now()

class HumanCheckpoint:
    """Human-in-the-loop checkpoint (74% of successful systems use this)"""
    
    def __init__(self, auto_approve: bool = False):
        self.pending_approvals: Dict[str, ApprovalRequest] = {}
        self.approval_history: List[ApprovalRequest] = []
        self.auto_approve = auto_approve  # For testing
    
    def request_approval(
        self,
        request_id: str,
        context: Dict[str, Any],
        approval_type: str = "standard",
        timeout: int = 300  # 5 minutes
    ) -> bool:
        """Request human approval"""
        request = ApprovalRequest(
            request_id=request_id,
            context=context,
            approval_type=approval_type
        )
        
        self.pending_approvals[request_id] = request
        
        print(f"\n[Human Checkpoint] Approval Required")
        print(f"  Request ID: {request_id}")
        print(f"  Type: {approval_type}")
        print(f"  Description: {context.get('description', 'N/A')}")
        print(f"  Context: {context}")
        
        if self.auto_approve:
            return self._auto_approve(request)
        
        # In production, would integrate with UI
        response = input("\nApprove? (y/n): ").lower().strip()
        approved = response == 'y'
        
        request.status = ApprovalStatus.APPROVED if approved else ApprovalStatus.REJECTED
        request.approved_by = "human" if approved else None
        request.response_time = (datetime.now() - request.timestamp).total_seconds()
        
        self.approval_history.append(request)
        del self.pending_approvals[request_id]
        
        return approved
    
    def _auto_approve(self, request: ApprovalRequest) -> bool:
        """Auto-approve for testing"""
        request.status = ApprovalStatus.APPROVED
        request.approved_by = "auto"
        request.response_time = 0.0
        self.approval_history.append(request)
        del self.pending_approvals[request.request_id]
        return True
    
    def get_approval_stats(self) -> Dict[str, Any]:
        """Get approval statistics"""
        total = len(self.approval_history)
        approved = sum(1 for r in self.approval_history if r.status == ApprovalStatus.APPROVED)
        rejected = total - approved
        
        avg_response_time = (
            sum(r.response_time for r in self.approval_history if r.response_time)
            / total if total > 0 else 0
        )
        
        return {
            "total_requests": total,
            "approved": approved,
            "rejected": rejected,
            "approval_rate": (approved / total * 100) if total > 0 else 0,
            "avg_response_time_seconds": avg_response_time
        }
```

### Step 2: Integrate with Agents

Update your agent classes to use human checkpoints:

```python
class HumanInTheLoopAgent(BoundedAutonomyAgent):
    """Agent with human-in-the-loop checkpoints"""
    
    def __init__(
        self,
        agent_id: str,
        capabilities: List[str],
        human_checkpoint: HumanCheckpoint,
        message_bus=None
    ):
        super().__init__(agent_id, capabilities, message_bus=message_bus)
        self.human_checkpoint = human_checkpoint
    
    def execute_with_approval(self, action: Dict[str, Any]) -> Dict[str, Any]:
        """Execute action requiring approval"""
        if self._requires_approval(action):
            approved = self.human_checkpoint.request_approval(
                request_id=f"{self.agent_id}-{action.get('id', 'unknown')}",
                context={
                    "action": action,
                    "agent_id": self.agent_id,
                    "description": action.get("description", "")
                },
                approval_type=self._get_approval_type(action)
            )
            
            if not approved:
                return {
                    "status": "rejected",
                    "reason": "Human approval denied",
                    "action": action
                }
        
        return self.execute_step(action)
    
    def _requires_approval(self, action: Dict[str, Any]) -> bool:
        """Determine if action requires approval"""
        high_risk_keywords = ["delete", "modify", "critical", "production", "security", "deploy"]
        action_desc = str(action.get("description", "")).lower()
        return any(keyword in action_desc for keyword in high_risk_keywords)
    
    def _get_approval_type(self, action: Dict[str, Any]) -> str:
        """Get approval type"""
        desc = str(action.get("description", "")).lower()
        if "delete" in desc or "remove" in desc:
            return "destructive"
        elif "security" in desc or "access" in desc:
            return "security"
        elif "deploy" in desc or "production" in desc:
            return "deployment"
        else:
            return "standard"
```

### Deliverable 2.1: Human-in-the-Loop

- [ ] Human checkpoint system implemented
- [ ] Approval request/response working
- [ ] Integration with agents
- [ ] Approval statistics tracking
- [ ] Tests passing

**Commit your work:**
```bash
git add .
git commit -m "Implement human-in-the-loop checkpoints"
```

## Exercise 3: Build Tool Registry

### Step 1: Create Tool Registry

Create `tools/tool_registry.py`:

```python
from typing import Dict, List, Optional, Any
from dataclasses import dataclass
from abc import ABC, abstractmethod

@dataclass
class ToolSpec:
    """Tool specification (inspired by Isorythm)"""
    tool_id: str
    name: str
    description: str
    input_schema: Dict[str, Any]
    output_schema: Dict[str, Any]
    required_capabilities: List[str] = None
    framework: str = "agnostic"  # "crewai", "langchain", "agnostic"
    
    def __post_init__(self):
        if self.required_capabilities is None:
            self.required_capabilities = []

class Tool(ABC):
    """Abstract tool interface"""
    
    def __init__(self, spec: ToolSpec):
        self.spec = spec
    
    @abstractmethod
    def execute(self, **kwargs) -> Dict[str, Any]:
        """Execute tool"""
        pass

class ToolRegistry:
    """Centralized tool registry (inspired by Isorythm)"""
    
    def __init__(self):
        self.tools: Dict[str, ToolSpec] = {}
        self.tool_instances: Dict[str, Tool] = {}
        self.tool_counter = 0
    
    def register_tool(self, tool: Tool, tool_id: Optional[str] = None) -> str:
        """Register a tool"""
        if tool_id:
            tool.spec.tool_id = tool_id
        else:
            self.tool_counter += 1
            tool.spec.tool_id = f"tool-{self.tool_counter}"
        
        self.tools[tool.spec.tool_id] = tool.spec
        self.tool_instances[tool.spec.tool_id] = tool
        return tool.spec.tool_id
    
    def get_tool(self, tool_id: str) -> Optional[Tool]:
        """Get tool instance by ID"""
        return self.tool_instances.get(tool_id)
    
    def get_tool_spec(self, tool_id: str) -> Optional[ToolSpec]:
        """Get tool specification by ID"""
        return self.tools.get(tool_id)
    
    def get_tool_by_name(self, name: str) -> Optional[Tool]:
        """Get tool by name"""
        for tool_id, spec in self.tools.items():
            if spec.name == name:
                return self.tool_instances[tool_id]
        return None
    
    def list_tools(self) -> List[ToolSpec]:
        """List all registered tools"""
        return list(self.tools.values())
    
    def get_tools_for_capabilities(self, capabilities: List[str]) -> List[Tool]:
        """Get tools matching capabilities"""
        matching_tools = []
        for tool_id, spec in self.tools.items():
            if any(cap in spec.required_capabilities for cap in capabilities):
                matching_tools.append(self.tool_instances[tool_id])
        return matching_tools
```

### Step 2: Create Example Tools

Create `tools/example_tools.py`:

```python
from tools.tool_registry import Tool, ToolSpec

class CalculatorTool(Tool):
    """Simple calculator tool"""
    
    def __init__(self):
        spec = ToolSpec(
            tool_id="calculator",
            name="Calculator",
            description="Perform mathematical calculations",
            input_schema={"expression": "string"},
            output_schema={"result": "number"},
            required_capabilities=["calculation"]
        )
        super().__init__(spec)
    
    def execute(self, expression: str) -> Dict[str, Any]:
        """Execute calculation"""
        try:
            result = eval(expression)  # In production, use safe evaluator
            return {"result": result, "status": "success"}
        except Exception as e:
            return {"result": None, "status": "error", "error": str(e)}

class FileReaderTool(Tool):
    """File reading tool"""
    
    def __init__(self):
        spec = ToolSpec(
            tool_id="file-reader",
            name="File Reader",
            description="Read files from filesystem",
            input_schema={"file_path": "string"},
            output_schema={"content": "string", "lines": "number"},
            required_capabilities=["file_operations"]
        )
        super().__init__(spec)
    
    def execute(self, file_path: str) -> Dict[str, Any]:
        """Read file"""
        try:
            with open(file_path, 'r', encoding='utf-8') as f:
                content = f.read()
                lines = content.split('\n')
            return {
                "content": content,
                "lines": len(lines),
                "status": "success"
            }
        except Exception as e:
            return {"content": None, "status": "error", "error": str(e)}
```

### Deliverable 3.1: Tool Registry

- [ ] Tool registry implemented
- [ ] Tool specification structure
- [ ] Tool registration working
- [ ] Example tools created
- [ ] Capability-based tool lookup
- [ ] Tests passing

**Commit your work:**
```bash
git add .
git commit -m "Implement tool registry system"
```

## Exercise 4: Complete System Integration

### Step 1: Build Complete System

Create `systems/complete_system.py`:

```python
from agents.bounded_autonomy_agent import BoundedAutonomyAgent
from agents.human_in_the_loop_agent import HumanInTheLoopAgent
from coordination.human_checkpoint import HumanCheckpoint
from communication.message_bus import MessageBus
from tools.tool_registry import ToolRegistry
from tools.example_tools import CalculatorTool, FileReaderTool

class CompleteMultiAgentSystem:
    """Complete multi-agent system with all patterns"""
    
    def __init__(self):
        # Initialize components
        self.message_bus = MessageBus()
        self.tool_registry = ToolRegistry()
        self.human_checkpoint = HumanCheckpoint(auto_approve=False)
        
        # Register tools
        self.tool_registry.register_tool(CalculatorTool())
        self.tool_registry.register_tool(FileReaderTool())
        
        # Create agents
        self.agents = [
            HumanInTheLoopAgent(
                f"agent-{i}",
                ["processing", "analysis"],
                human_checkpoint=self.human_checkpoint,
                message_bus=self.message_bus
            )
            for i in range(3)
        ]
    
    def execute_task(self, task: Dict[str, Any]) -> Dict[str, Any]:
        """Execute task using multi-agent system"""
        results = []
        
        for agent in self.agents:
            if agent.capabilities_match(task.get("required_capabilities", [])):
                result = agent.execute_with_approval({
                    "id": task["id"],
                    "description": task["description"],
                    "data": task.get("data")
                })
                results.append(result)
        
        return {
            "task_id": task["id"],
            "results": results,
            "status": "completed" if all(r.get("status") != "rejected" for r in results) else "partial"
        }
    
    def get_system_status(self) -> Dict[str, Any]:
        """Get system status"""
        return {
            "agents": len(self.agents),
            "tools": len(self.tool_registry.list_tools()),
            "pending_approvals": len(self.human_checkpoint.pending_approvals),
            "approval_stats": self.human_checkpoint.get_approval_stats()
        }
```

### Step 2: Test Complete System

Create `tests/test_complete_system.py`:

```python
from systems.complete_system import CompleteMultiAgentSystem

def test_complete_system():
    system = CompleteMultiAgentSystem()
    
    # Execute task
    task = {
        "id": "task-1",
        "description": "Process data file",
        "required_capabilities": ["processing"],
        "data": {"file": "test.txt"}
    }
    
    result = system.execute_task(task)
    print(f"Task result: {result}")
    
    # Get system status
    status = system.get_system_status()
    print(f"System status: {status}")
    
    print("✅ Complete system test passed")

if __name__ == "__main__":
    test_complete_system()
```

### Deliverable 4.1: Complete System

- [ ] Complete system integrated
- [ ] All components working together
- [ ] End-to-end test passing
- [ ] System status monitoring
- [ ] Documentation complete

**Commit your work:**
```bash
git add .
git commit -m "Complete multi-agent system with all patterns integrated"
```

## Reflection Questions

1. **Why is bounded autonomy (≤10 steps)** important for production systems?
2. **How does human-in-the-loop** improve system reliability?
3. **What benefits** does a tool registry provide?
4. **How do all patterns** work together in a complete system?
5. **What challenges** did you encounter integrating everything?
6. **How would you improve** your complete system?

## Troubleshooting

### Issue: Bounded autonomy not enforcing limits
- **Solution**: Check step counting logic
- **Check**: Escalation triggers correctly

### Issue: Human checkpoint blocking execution
- **Solution**: Implement async approval or auto-approve for testing
- **Check**: Approval flow is correct

### Issue: Tool registry not finding tools
- **Solution**: Verify tool registration
- **Check**: Capability matching logic

## Next Steps

- Review Day 4 lesson materials
- Complete all exercises
- Commit your work to Git
- Prepare for Day 5: Agent Lifecycle and State Management

## Deliverables Checklist

- [ ] Bounded autonomy implemented
- [ ] Human-in-the-loop checkpoints working
- [ ] Tool registry built
- [ ] Complete system integrated
- [ ] End-to-end tests passing
- [ ] Code committed to Git repository
- [ ] Reflection questions answered
- [ ] Ready for Day 5

---

**Excellent work!** You've built a complete multi-agent system. Tomorrow we'll learn about agent lifecycle and state management.

