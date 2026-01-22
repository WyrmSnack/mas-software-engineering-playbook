# Day 3 Worksheet: Agent Architectures and Patterns

## Objectives

- Design architecture for a given problem
- Implement WorkflowIR-like framework-agnostic structure
- Build nested building blocks pattern
- Create architecture decision matrix
- Apply Isorythm patterns to your system

## Prerequisites Checklist

Before starting, ensure you have:

- [ ] Completed Day 1 and Day 2 exercises
- [ ] Understanding of communication patterns
- [ ] Python environment set up
- [ ] Git repository with previous work

## Exercise 1: Design Architecture for Problem

### Step 1: Problem Statement

Design an architecture for a **code review system** with the following requirements:
- Multiple specialized reviewers (security, performance, style, documentation)
- Coordinator that assigns code to appropriate reviewers
- Results aggregation and reporting
- Support for 10-50 reviewers
- Need for both parallel and sequential review steps

### Step 2: Architecture Analysis

Answer these questions:

1. **What is the natural structure** of this problem?
   - [ ] Hierarchical
   - [ ] Peer-to-Peer
   - [ ] Hybrid
   - [ ] Skill-Centric
   - [ ] Hierarchical Clusters

2. **What communication pattern** fits best?
   - [ ] Direct communication
   - [ ] Message bus
   - [ ] Publish-subscribe
   - [ ] Blackboard

3. **What coordination mechanism** is appropriate?
   - [ ] Simple assignment
   - [ ] Contract Net Protocol
   - [ ] Market-based
   - [ ] Centralized coordinator

4. **Estimated number of agents**: __________

5. **Scaling requirements**: __________

### Step 3: Architecture Design

Create `architecture/code_review_architecture.md`:

```markdown
# Code Review System Architecture

## Selected Architecture
[Your choice: Hierarchical/Skill-Centric/Hybrid/etc.]

## Rationale
[Why this architecture fits the problem]

## Agent Roles
1. **Coordinator Agent**
   - Responsibilities: [list]
   - Capabilities: [list]

2. **Security Reviewer Agent**
   - Responsibilities: [list]
   - Capabilities: [list]

[Continue for all agents]

## Communication Pattern
[Selected pattern and justification]

## Coordination Mechanism
[Selected mechanism and how it works]

## Scaling Analysis
- Expected agents: [number]
- Communication complexity: O([complexity])
- Expected performance: [assessment]
```

### Deliverable 1.1: Architecture Design

- [ ] Problem analyzed
- [ ] Architecture selected with rationale
- [ ] Agent roles defined
- [ ] Communication pattern chosen
- [ ] Coordination mechanism selected
- [ ] Scaling analysis completed

**Commit your work:**
```bash
git add .
git commit -m "Design code review system architecture"
```

## Exercise 2: Implement WorkflowIR-Like Structure

### Step 1: Create WorkflowIR Data Classes

Create `architecture/workflow_ir.py`:

```python
from dataclasses import dataclass, field
from typing import List, Dict, Any, Optional
from enum import Enum

class TaskStatus(Enum):
    PENDING = "pending"
    READY = "ready"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"

@dataclass
class ToolSpec:
    """Framework-agnostic tool specification"""
    tool_id: str
    name: str
    description: str
    input_schema: Dict[str, Any]
    output_schema: Dict[str, Any]

@dataclass
class AgentSpec:
    """Framework-agnostic agent specification"""
    agent_id: str
    role: str
    goal: str
    backstory: str
    tools: List[str] = field(default_factory=list)
    llm_model: str = "gpt-4o"
    temperature: float = 0.7
    max_iter: int = 10
    allow_delegation: bool = False

@dataclass
class TaskSpec:
    """Framework-agnostic task specification"""
    task_id: str
    name: str
    description: str
    expected_output: str
    assigned_agent_id: Optional[str] = None
    depends_on: List[str] = field(default_factory=list)
    tools: List[str] = field(default_factory=list)

@dataclass
class WorkflowSpec:
    """Framework-agnostic workflow specification"""
    workflow_id: str
    name: str
    description: str
    process_type: str = "sequential"  # sequential, hierarchical, etc.

@dataclass
class WorkflowIR:
    """Workflow Intermediate Representation (framework-agnostic)"""
    workflow_spec: WorkflowSpec
    agents: List[AgentSpec]
    tasks: List[TaskSpec]
    tools: List[ToolSpec] = field(default_factory=list)
    
    def get_agent(self, agent_id: str) -> Optional[AgentSpec]:
        """Get agent by ID"""
        for agent in self.agents:
            if agent.agent_id == agent_id:
                return agent
        return None
    
    def get_task(self, task_id: str) -> Optional[TaskSpec]:
        """Get task by ID"""
        for task in self.tasks:
            if task.task_id == task_id:
                return task
        return None
    
    def validate(self) -> List[str]:
        """Validate workflow IR structure"""
        errors = []
        
        # Check agent references in tasks
        for task in self.tasks:
            if task.assigned_agent_id:
                if not self.get_agent(task.assigned_agent_id):
                    errors.append(f"Task {task.task_id} references unknown agent {task.assigned_agent_id}")
        
        # Check task dependencies
        task_ids = {task.task_id for task in self.tasks}
        for task in self.tasks:
            for dep_id in task.depends_on:
                if dep_id not in task_ids:
                    errors.append(f"Task {task.task_id} depends on unknown task {dep_id}")
        
        # Check tool references
        tool_names = {tool.name for tool in self.tools}
        for agent in self.agents:
            for tool_name in agent.tools:
                if tool_name not in tool_names:
                    errors.append(f"Agent {agent.agent_id} references unknown tool {tool_name}")
        
        return errors
```

### Step 2: Create Adapter Interface

Create `architecture/adapter.py`:

```python
from abc import ABC, abstractmethod
from architecture.workflow_ir import WorkflowIR
from typing import Any

class OrchestrationAdapter(ABC):
    """Abstract adapter for framework-specific orchestration"""
    
    @abstractmethod
    def compile(self, workflow_ir: WorkflowIR) -> Any:
        """Compile WorkflowIR to framework-specific format"""
        pass
    
    @abstractmethod
    def execute(self, compiled_workflow: Any) -> Dict[str, Any]:
        """Execute compiled workflow"""
        pass
    
    @abstractmethod
    def get_framework_name(self) -> str:
        """Get framework name"""
        pass

class SimpleAdapter(OrchestrationAdapter):
    """Simple adapter for demonstration (not framework-specific)"""
    
    def compile(self, workflow_ir: WorkflowIR) -> Dict[str, Any]:
        """Compile to simple dictionary format"""
        return {
            "workflow_id": workflow_ir.workflow_spec.workflow_id,
            "agents": [
                {
                    "agent_id": agent.agent_id,
                    "role": agent.role,
                    "goal": agent.goal
                }
                for agent in workflow_ir.agents
            ],
            "tasks": [
                {
                    "task_id": task.task_id,
                    "name": task.name,
                    "description": task.description,
                    "assigned_agent": task.assigned_agent_id
                }
                for task in workflow_ir.tasks
            ]
        }
    
    def execute(self, compiled_workflow: Dict[str, Any]) -> Dict[str, Any]:
        """Execute compiled workflow (simplified)"""
        result = {
            "workflow_id": compiled_workflow["workflow_id"],
            "status": "completed",
            "tasks_completed": len(compiled_workflow["tasks"])
        }
        return result
    
    def get_framework_name(self) -> str:
        return "simple"
```

### Step 3: Test WorkflowIR

Create `tests/test_workflow_ir.py`:

```python
from architecture.workflow_ir import WorkflowIR, WorkflowSpec, AgentSpec, TaskSpec
from architecture.adapter import SimpleAdapter

def test_workflow_ir():
    # Create workflow IR
    workflow_ir = WorkflowIR(
        workflow_spec=WorkflowSpec(
            workflow_id="test-workflow",
            name="Test Workflow",
            description="A test workflow"
        ),
        agents=[
            AgentSpec(
                agent_id="agent-1",
                role="Coder",
                goal="Write code",
                backstory="Experienced developer"
            )
        ],
        tasks=[
            TaskSpec(
                task_id="task-1",
                name="Write Function",
                description="Write a function",
                expected_output="Function code",
                assigned_agent_id="agent-1"
            )
        ]
    )
    
    # Validate
    errors = workflow_ir.validate()
    assert len(errors) == 0, f"Validation errors: {errors}"
    
    # Compile with adapter
    adapter = SimpleAdapter()
    compiled = adapter.compile(workflow_ir)
    
    assert compiled["workflow_id"] == "test-workflow"
    assert len(compiled["agents"]) == 1
    assert len(compiled["tasks"]) == 1
    
    print("✅ WorkflowIR test passed")

if __name__ == "__main__":
    test_workflow_ir()
```

### Deliverable 2.1: WorkflowIR Implementation

- [ ] WorkflowIR data classes implemented
- [ ] Adapter interface created
- [ ] Simple adapter implemented
- [ ] Validation logic working
- [ ] Tests passing

**Commit your work:**
```bash
git add .
git commit -m "Implement WorkflowIR framework-agnostic structure"
```

## Exercise 3: Nested Building Blocks Pattern

### Step 1: Extend WorkflowIR for Nested References

Update `architecture/workflow_ir.py` to support nested references:

```python
@dataclass
class NestedReference:
    """Reference to nested building block"""
    ref_type: str  # "workflow", "agent", "task", "tool"
    ref_id: str
    execution_order: int = 0
    trigger: str = "always"  # "always", "on_success", "on_failure", "conditional"
    context: Dict[str, Any] = field(default_factory=dict)

@dataclass
class AgentSpec:
    """Extended with nested references"""
    agent_id: str
    role: str
    goal: str
    backstory: str
    tools: List[str] = field(default_factory=list)
    nested_agents: List[NestedReference] = field(default_factory=list)
    nested_workflows: List[NestedReference] = field(default_factory=list)
    nested_tasks: List[NestedReference] = field(default_factory=list)
    # ... other fields

@dataclass
class TaskSpec:
    """Extended with nested references"""
    task_id: str
    name: str
    description: str
    expected_output: str
    assigned_agent_id: Optional[str] = None
    depends_on: List[str] = field(default_factory=list)
    tools: List[str] = field(default_factory=list)
    sub_workflows: List[NestedReference] = field(default_factory=list)
    sub_tasks: List[NestedReference] = field(default_factory=list)
    nested_agents: List[NestedReference] = field(default_factory=list)
    # ... other fields

@dataclass
class WorkflowSpec:
    """Extended with nested references"""
    workflow_id: str
    name: str
    description: str
    process_type: str = "sequential"
    sub_workflows: List[NestedReference] = field(default_factory=list)
    nested_tools: List[NestedReference] = field(default_factory=list)
```

### Step 2: Create Nested Workflow Example

Create `examples/nested_workflow_example.py`:

```python
from architecture.workflow_ir import (
    WorkflowIR, WorkflowSpec, AgentSpec, TaskSpec, NestedReference
)

def create_nested_workflow_example():
    """Create example with nested building blocks"""
    
    # Main workflow
    main_workflow = WorkflowIR(
        workflow_spec=WorkflowSpec(
            workflow_id="main-workflow",
            name="Main Development Workflow",
            description="Complete development workflow with nested components"
        ),
        agents=[
            AgentSpec(
                agent_id="coordinator",
                role="Project Coordinator",
                goal="Coordinate development tasks",
                backstory="Experienced project manager",
                nested_agents=[
                    NestedReference(
                        ref_type="agent",
                        ref_id="reviewer",
                        execution_order=2,
                        trigger="on_success"
                    )
                ]
            ),
            AgentSpec(
                agent_id="developer",
                role="Developer",
                goal="Write code",
                backstory="Senior developer",
                nested_tasks=[
                    NestedReference(
                        ref_type="task",
                        ref_id="code-review",
                        execution_order=1,
                        trigger="on_success"
                    )
                ]
            )
        ],
        tasks=[
            TaskSpec(
                task_id="develop-feature",
                name="Develop Feature",
                description="Implement new feature",
                expected_output="Feature code",
                assigned_agent_id="developer",
                sub_workflows=[
                    NestedReference(
                        ref_type="workflow",
                        ref_id="testing-workflow",
                        execution_order=2,
                        trigger="on_success"
                    )
                ]
            ),
            TaskSpec(
                task_id="code-review",
                name="Code Review",
                description="Review code quality",
                expected_output="Review report",
                assigned_agent_id="reviewer"
            )
        ]
    )
    
    # Validate nested structure
    errors = main_workflow.validate()
    if errors:
        print("Validation errors:", errors)
    else:
        print("✅ Nested workflow structure is valid")
    
    return main_workflow

if __name__ == "__main__":
    workflow = create_nested_workflow_example()
    print(f"Workflow: {workflow.workflow_spec.name}")
    print(f"Agents: {len(workflow.agents)}")
    print(f"Tasks: {len(workflow.tasks)}")
```

### Deliverable 3.1: Nested Building Blocks

- [ ] Nested references added to WorkflowIR
- [ ] Nested workflow example created
- [ ] Validation handles nested references
- [ ] Example demonstrates nesting
- [ ] Tests passing

**Commit your work:**
```bash
git add .
git commit -m "Implement nested building blocks pattern"
```

## Exercise 4: Architecture Decision Matrix

### Step 1: Create Decision Matrix

Create `architecture/decision_matrix.md`:

```markdown
# Architecture Decision Matrix

## Problem: [Your problem description]

## Architecture Options

| Architecture | Pros | Cons | Scaling | Complexity | Score |
|-------------|------|------|---------|------------|-------|
| Hierarchical | [list] | [list] | O(log n) | Medium | [1-10] |
| Peer-to-Peer | [list] | [list] | O(n²) | Low | [1-10] |
| Hybrid | [list] | [list] | O(n log n) | High | [1-10] |
| Skill-Centric | [list] | [list] | O(n) | Medium | [1-10] |
| Hierarchical Clusters | [list] | [list] | O(log² n) | Very High | [1-10] |

## Selected Architecture
[Your choice]

## Rationale
[Detailed explanation of why this architecture was chosen]

## Trade-offs Accepted
[What trade-offs you're making and why they're acceptable]

## Alternative Considered
[What you considered but rejected and why]
```

### Step 2: Document Decision Process

Answer these questions:

1. **What factors** were most important in your decision?
2. **What trade-offs** did you make?
3. **How would you validate** your architecture choice?
4. **What would cause you** to reconsider this architecture?

### Deliverable 4.1: Architecture Decision Matrix

- [ ] Decision matrix completed
- [ ] All architectures evaluated
- [ ] Selection justified
- [ ] Trade-offs documented
- [ ] Decision process recorded

## Reflection Questions

1. **How do the five canonical architectures** differ in their scaling properties?
2. **When would you choose** framework-agnostic design over framework-specific?
3. **What benefits** does nested building blocks provide?
4. **How does Isorythm's architecture** compare to the canonical patterns?
5. **What factors** are most important when selecting an architecture?
6. **How would you measure** if your architecture choice was correct?

## Troubleshooting

### Issue: WorkflowIR validation fails
- **Solution**: Check all references exist
- **Check**: Agent IDs, task IDs, tool names match

### Issue: Nested references circular
- **Solution**: Implement cycle detection
- **Check**: Reference chains don't create loops

### Issue: Adapter compilation fails
- **Solution**: Verify all required fields present
- **Check**: Framework-specific requirements met

## Next Steps

- Review Day 3 lesson materials
- Complete all exercises
- Commit your work to Git
- Prepare for Day 4: Building Your First Multi-Agent System

## Deliverables Checklist

- [ ] Architecture designed for code review system
- [ ] WorkflowIR structure implemented
- [ ] Adapter pattern implemented
- [ ] Nested building blocks working
- [ ] Architecture decision matrix completed
- [ ] Code committed to Git repository
- [ ] Reflection questions answered
- [ ] Ready for Day 4

---

**Excellent progress!** You've learned about architectures and patterns. Tomorrow we'll build a complete multi-agent system using these patterns.

