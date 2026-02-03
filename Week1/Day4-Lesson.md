# Day 4: Building Your First Multi-Agent System

## Learning Objectives

By the end of this lesson, you will be able to:
- Implement bounded autonomy pattern (≤10 steps)
- Add human-in-the-loop checkpoints (74% of successful systems)
- Build a tool registry system
- Design error handling and fault tolerance
- Integrate all patterns into a complete system
- Apply Isorythm's implementation patterns

## Bounded Autonomy Pattern

**Research Finding**: 68% of successful production deployments use bounded autonomy with ≤10 steps per agent.

### What is Bounded Autonomy?

Bounded autonomy limits the number of steps an agent can take autonomously before requiring human oversight or escalation.

**Key Principles**:
1. **Step Limits**: Agents have maximum step counts (typically ≤10)
2. **Checkpoints**: Regular checkpoints for validation
3. **Escalation**: Automatic escalation when limits reached
4. **Human Oversight**: Critical decisions require human approval

### Implementation Pattern

```python
class BoundedAutonomyAgent(BaseAgent):
    MAX_STEPS = 10  # Research finding: 68% use ≤10 steps
    
    def __init__(self, agent_id: str, capabilities: List[str]):
        super().__init__(agent_id, capabilities)
        self.step_count = 0
        self.max_steps = self.MAX_STEPS
    
    def execute_step(self, action: Dict[str, Any]) -> Dict[str, Any]:
        """Execute a single step with bounded autonomy"""
        if self.step_count >= self.max_steps:
            return {
                "status": "escalated",
                "reason": "Max steps reached",
                "step_count": self.step_count
            }
        
        self.step_count += 1
        result = self._perform_action(action)
        
        # Checkpoint every 5 steps
        if self.step_count % 5 == 0:
            if not self._checkpoint_validation():
                return {
                    "status": "halted",
                    "reason": "Checkpoint validation failed",
                    "step_count": self.step_count
                }
        
        return result
    
    def _checkpoint_validation(self) -> bool:
        """Validate at checkpoint"""
        # In production, would check with human or validation system
        return True
    
    def reset_step_count(self):
        """Reset step count (e.g., after human approval)"""
        self.step_count = 0
```

**Isorythm Pattern**: Isorythm's agents operate within workflow constraints that naturally limit autonomy, with execution tracking through WorkflowRun and TaskRun models.

## Human-in-the-Loop Pattern

**Research Finding**: 74% of successful agent systems rely on human-in-the-loop evaluation.

### Why Human-in-the-Loop?

1. **Critical Decisions**: Humans make final decisions on important matters
2. **Quality Control**: Human validation ensures quality
3. **Error Prevention**: Catch errors before they propagate
4. **Trust Building**: Human oversight builds confidence

### Implementation Pattern

```python
class HumanCheckpoint:
    """Human-in-the-loop checkpoint system"""
    
    def __init__(self):
        self.pending_approvals: Dict[str, ApprovalRequest] = {}
        self.approval_callbacks: Dict[str, Callable] = {}
    
    def request_approval(
        self,
        request_id: str,
        context: Dict[str, Any],
        approval_type: str = "standard"
    ) -> bool:
        """Request human approval"""
        request = ApprovalRequest(
            request_id=request_id,
            context=context,
            approval_type=approval_type,
            timestamp=datetime.now()
        )
        
        self.pending_approvals[request_id] = request
        
        # In production, would integrate with UI/notification system
        # For now, simulate approval
        return self._simulate_approval(request)
    
    def _simulate_approval(self, request: ApprovalRequest) -> bool:
        """Simulate human approval (replace with actual UI integration)"""
        print(f"\n[Human Checkpoint] Approval required:")
        print(f"  Request ID: {request.request_id}")
        print(f"  Type: {request.approval_type}")
        print(f"  Context: {request.context.get('description', 'N/A')}")
        
        # In production: return await ui_approval_dialog(request)
        response = input("Approve? (y/n): ").lower() == 'y'
        
        request.status = "approved" if response else "rejected"
        return response

class HumanInTheLoopAgent(BaseAgent):
    """Agent with human-in-the-loop checkpoints"""
    
    def __init__(self, agent_id: str, capabilities: List[str], checkpoint: HumanCheckpoint):
        super().__init__(agent_id, capabilities)
        self.checkpoint = checkpoint
    
    def execute_with_approval(self, action: Dict[str, Any]) -> Dict[str, Any]:
        """Execute action requiring human approval"""
        if self._requires_approval(action):
            approved = self.checkpoint.request_approval(
                request_id=f"{self.agent_id}-{action['id']}",
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
                    "reason": "Human approval denied"
                }
        
        return self._perform_action(action)
    
    def _requires_approval(self, action: Dict[str, Any]) -> bool:
        """Determine if action requires approval"""
        high_risk_keywords = ["delete", "modify", "critical", "production", "security"]
        action_desc = action.get("description", "").lower()
        return any(keyword in action_desc for keyword in high_risk_keywords)
    
    def _get_approval_type(self, action: Dict[str, Any]) -> str:
        """Get approval type based on action"""
        if "delete" in action.get("description", "").lower():
            return "destructive"
        elif "security" in action.get("description", "").lower():
            return "security"
        else:
            return "standard"
```

**Isorythm Pattern**: Isorythm supports human-in-the-loop through workflow status tracking and WebSocket notifications, allowing UIs to display workflow progress and request approvals.

## Tool Registry System

**Isorythm Pattern**: Isorythm implements a centralized tool registry that manages tools across the system.

### Benefits

1. **Centralized Management**: Single source of truth for tools
2. **Reusability**: Tools can be shared across agents
3. **Discovery**: Easy to find available tools
4. **Versioning**: Track tool versions and updates

### Implementation Pattern

```python
class ToolRegistry:
    """Centralized tool registry (inspired by Isorythm)"""
    
    def __init__(self):
        self.tools: Dict[str, ToolSpec] = {}
        self.tool_counter = 0
    
    def register_tool(self, tool: ToolSpec) -> str:
        """Register a tool"""
        if tool.tool_id:
            tool_id = tool.tool_id
        else:
            self.tool_counter += 1
            tool_id = f"tool-{self.tool_counter}"
            tool.tool_id = tool_id
        
        self.tools[tool_id] = tool
        return tool_id
    
    def get_tool(self, tool_id: str) -> Optional[ToolSpec]:
        """Get tool by ID"""
        return self.tools.get(tool_id)
    
    def get_tool_by_name(self, name: str) -> Optional[ToolSpec]:
        """Get tool by name"""
        for tool in self.tools.values():
            if tool.name == name:
                return tool
        return None
    
    def list_tools(self) -> List[ToolSpec]:
        """List all registered tools"""
        return list(self.tools.values())
    
    def get_tools_for_agent(self, agent_capabilities: List[str]) -> List[ToolSpec]:
        """Get tools matching agent capabilities"""
        matching_tools = []
        for tool in self.tools.values():
            if any(cap in tool.required_capabilities for cap in agent_capabilities):
                matching_tools.append(tool)
        return matching_tools

# Example tool implementations
class WebSearchTool(ToolSpec):
    def __init__(self):
        super().__init__(
            tool_id="web-search",
            name="Web Search",
            description="Search the web for information",
            input_schema={"query": "string"},
            output_schema={"results": "array"}
        )
        self.required_capabilities = ["research", "information_gathering"]
    
    def execute(self, query: str) -> Dict[str, Any]:
        """Execute web search"""
        # Implementation here
        return {"results": []}

class CodeAnalysisTool(ToolSpec):
    def __init__(self):
        super().__init__(
            tool_id="code-analysis",
            name="Code Analysis",
            description="Analyze code for issues",
            input_schema={"code": "string", "language": "string"},
            output_schema={"issues": "array", "metrics": "object"}
        )
        self.required_capabilities = ["code_review", "analysis"]
    
    def execute(self, code: str, language: str) -> Dict[str, Any]:
        """Execute code analysis"""
        # Implementation here
        return {"issues": [], "metrics": {}}
```

**Isorythm Implementation**: Isorythm's tool registry supports dual framework compatibility (CrewAI and LangChain), automatic framework conversion, and centralized tool management.

## Error Handling and Fault Tolerance

### Error Handling Patterns

1. **Graceful Degradation**: System continues with reduced functionality
2. **Retry Logic**: Automatic retry with exponential backoff
3. **Circuit Breaker**: Stop calling failing services
4. **Fallback Mechanisms**: Alternative approaches when primary fails

### Implementation Pattern

```python
class FaultTolerantAgent(BaseAgent):
    """Agent with fault tolerance"""
    
    def __init__(self, agent_id: str, capabilities: List[str], max_retries: int = 3):
        super().__init__(agent_id, capabilities)
        self.max_retries = max_retries
        self.retry_delays = [1, 2, 5]  # Exponential backoff
    
    def execute_with_retry(self, action: Dict[str, Any]) -> Dict[str, Any]:
        """Execute action with retry logic"""
        last_error = None
        
        for attempt in range(self.max_retries):
            try:
                result = self._perform_action(action)
                if result.get("status") == "success":
                    return result
                else:
                    # Retry on failure
                    if attempt < self.max_retries - 1:
                        time.sleep(self.retry_delays[attempt])
                        continue
            except Exception as e:
                last_error = e
                if attempt < self.max_retries - 1:
                    time.sleep(self.retry_delays[attempt])
                    continue
        
        # All retries failed
        return {
            "status": "failed",
            "error": str(last_error) if last_error else "Unknown error",
            "attempts": self.max_retries
        }
    
    def _perform_action(self, action: Dict[str, Any]) -> Dict[str, Any]:
        """Perform action (override in subclasses)"""
        raise NotImplementedError
```

## Complete System Integration

### System Architecture

A complete multi-agent system integrates:

1. **Agent Framework**: Base agent classes with bounded autonomy
2. **Communication**: Message bus or coordination mechanism
3. **Coordination**: Contract Net, Blackboard, or centralized
4. **Tool Registry**: Centralized tool management
5. **Human Checkpoints**: Approval and oversight mechanisms
6. **Error Handling**: Fault tolerance and recovery
7. **Monitoring**: Observability and logging

### Example: Code Review System

```python
class CodeReviewSystem:
    """Complete code review multi-agent system"""
    
    def __init__(self):
        # Components
        self.message_bus = MessageBus()
        self.tool_registry = ToolRegistry()
        self.human_checkpoint = HumanCheckpoint()
        
        # Register tools
        self.tool_registry.register_tool(CodeAnalysisTool())
        self.tool_registry.register_tool(SecurityAnalysisTool())
        
        # Create agents
        self.coordinator = CoordinatorAgent(
            "coordinator",
            message_bus=self.message_bus
        )
        
        self.reviewers = [
            BoundedAutonomyAgent(
                f"reviewer-{i}",
                ["code_review", "analysis"],
                message_bus=self.message_bus,
                tool_registry=self.tool_registry,
                human_checkpoint=self.human_checkpoint
            )
            for i in range(5)
        ]
        
        # Register agents
        for reviewer in self.reviewers:
            self.coordinator.register_agent(reviewer)
    
    def review_code(self, code: str, language: str) -> Dict[str, Any]:
        """Review code using multi-agent system"""
        # Submit review task
        task_id = self.coordinator.submit_task({
            "type": "code_review",
            "code": code,
            "language": language
        })
        
        # Wait for completion (in production, would be async)
        # ...
        
        return {"task_id": task_id, "status": "completed"}
```

## Key Takeaways

1. **Bounded autonomy (≤10 steps)** is used by 68% of successful deployments
2. **Human-in-the-loop (74% usage)** is critical for production systems
3. **Tool registry** enables centralized tool management
4. **Error handling** and fault tolerance are essential for reliability
5. **Complete systems** integrate all patterns together
6. **Isorythm patterns** provide real-world implementation examples
7. **Reliability—not capability—is the central challenge** in production

## Next Steps

- Complete the Day 4 worksheet
- Build complete multi-agent system
- Integrate all patterns
- Test system end-to-end
- Read about agent lifecycle (preview of Day 5)

## Additional Resources

- [Production Patterns Guide](Resources/Production-Patterns.md)
- [Isorythm Patterns Guide](Resources/Isorythm-Patterns-Guide.md)
- [MASGSE Research Findings](Resources/MASGSE-Research-Findings.md)

---

**Ready for hands-on practice?** Complete the [Day 4 Worksheet](Day4-Worksheet.md)!

