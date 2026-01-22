# Day 6 Worksheet: Orchestration Patterns

## Objectives

- Implement adapter pattern for framework orchestration
- Design workflow patterns (sequential, parallel, hierarchical)
- Build load balancing mechanisms
- Apply Isorythm's orchestration patterns

## Exercise 1: Implement Adapter Pattern

### Step 1: Create Adapter Interface

Create `orchestration/adapter.py` following Isorythm's pattern:

```python
from abc import ABC, abstractmethod
from architecture.workflow_ir import WorkflowIR
from typing import Any, Dict

class OrchestrationAdapter(ABC):
    """Abstract adapter (Isorythm pattern)"""
    
    @abstractmethod
    def compile(self, workflow_ir: WorkflowIR) -> Any:
        pass
    
    @abstractmethod
    def execute(self, compiled: Any) -> Dict[str, Any]:
        pass
    
    @abstractmethod
    def get_framework_name(self) -> str:
        pass

# Implement simple adapter
class SimpleAdapter(OrchestrationAdapter):
    def compile(self, workflow_ir: WorkflowIR) -> Dict[str, Any]:
        return {
            "workflow_id": workflow_ir.workflow_spec.workflow_id,
            "agents": [a.agent_id for a in workflow_ir.agents],
            "tasks": [t.task_id for t in workflow_ir.tasks]
        }
    
    def execute(self, compiled: Dict[str, Any]) -> Dict[str, Any]:
        return {"status": "completed", "result": compiled}
    
    def get_framework_name(self) -> str:
        return "simple"
```

### Deliverable 1.1: Adapter Pattern

- [ ] Adapter interface created
- [ ] Simple adapter implemented
- [ ] Compile/execute working
- [ ] Tests passing

**Commit your work:**
```bash
git add .
git commit -m "Implement orchestration adapter pattern"
```

## Exercise 2: Workflow Patterns

### Step 1: Implement Sequential Workflow

Create `orchestration/sequential_workflow.py`:

```python
from architecture.workflow_ir import WorkflowIR, TaskSpec
from typing import List, Dict, Any

class SequentialWorkflowExecutor:
    """Execute workflow sequentially"""
    
    def execute(self, workflow_ir: WorkflowIR) -> Dict[str, Any]:
        """Execute tasks in order"""
        results = []
        
        # Sort tasks by dependencies
        sorted_tasks = self._topological_sort(workflow_ir.tasks)
        
        for task in sorted_tasks:
            result = self._execute_task(task, workflow_ir)
            results.append(result)
            
            if result["status"] == "failed":
                return {
                    "status": "failed",
                    "failed_task": task.task_id,
                    "results": results
                }
        
        return {
            "status": "completed",
            "results": results
        }
    
    def _topological_sort(self, tasks: List[TaskSpec]) -> List[TaskSpec]:
        """Sort tasks by dependencies"""
        # Simplified - in production would use proper topological sort
        return sorted(tasks, key=lambda t: len(t.depends_on))
    
    def _execute_task(self, task: TaskSpec, workflow_ir: WorkflowIR) -> Dict[str, Any]:
        """Execute single task"""
        # Simplified execution
        return {
            "task_id": task.task_id,
            "status": "completed",
            "result": f"Executed {task.name}"
        }
```

### Step 2: Implement Parallel Workflow

Create `orchestration/parallel_workflow.py`:

```python
import concurrent.futures
from typing import List, Dict, Any

class ParallelWorkflowExecutor:
    """Execute independent tasks in parallel"""
    
    def execute(self, workflow_ir: WorkflowIR) -> Dict[str, Any]:
        """Execute tasks in parallel"""
        # Find independent tasks
        independent_tasks = [
            t for t in workflow_ir.tasks
            if len(t.depends_on) == 0
        ]
        
        # Execute in parallel
        with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
            futures = {
                executor.submit(self._execute_task, task, workflow_ir): task
                for task in independent_tasks
            }
            
            results = []
            for future in concurrent.futures.as_completed(futures):
                task = futures[future]
                try:
                    result = future.result()
                    results.append(result)
                except Exception as e:
                    results.append({
                        "task_id": task.task_id,
                        "status": "failed",
                        "error": str(e)
                    })
        
        return {
            "status": "completed",
            "results": results
        }
    
    def _execute_task(self, task: TaskSpec, workflow_ir: WorkflowIR) -> Dict[str, Any]:
        """Execute single task"""
        return {
            "task_id": task.task_id,
            "status": "completed",
            "result": f"Executed {task.name} in parallel"
        }
```

### Deliverable 2.1: Workflow Patterns

- [ ] Sequential workflow implemented
- [ ] Parallel workflow implemented
- [ ] Task dependency handling
- [ ] Tests passing

**Commit your work:**
```bash
git add .
git commit -m "Implement sequential and parallel workflow patterns"
```

## Exercise 3: Load Balancing

### Step 1: Implement Load Balancer

Create `orchestration/load_balancer.py`:

```python
from typing import List, Dict, Any
from agents.base_agent import BaseAgent

class LoadBalancer:
    """Load balancing for task distribution"""
    
    def __init__(self, strategy: str = "round_robin"):
        self.strategy = strategy
        self.agent_loads: Dict[str, int] = {}
        self.round_robin_index = 0
    
    def select_agent(
        self,
        agents: List[BaseAgent],
        task: Dict[str, Any]
    ) -> BaseAgent:
        """Select agent based on strategy"""
        if self.strategy == "round_robin":
            return self._round_robin(agents)
        elif self.strategy == "least_loaded":
            return self._least_loaded(agents)
        elif self.strategy == "capability_based":
            return self._capability_based(agents, task)
        else:
            return agents[0]
    
    def _round_robin(self, agents: List[BaseAgent]) -> BaseAgent:
        """Round-robin selection"""
        agent = agents[self.round_robin_index % len(agents)]
        self.round_robin_index += 1
        return agent
    
    def _least_loaded(self, agents: List[BaseAgent]) -> BaseAgent:
        """Select least loaded agent"""
        # Initialize loads if needed
        for agent in agents:
            if agent.agent_id not in self.agent_loads:
                self.agent_loads[agent.agent_id] = 0
        
        # Find least loaded
        least_loaded_id = min(
            self.agent_loads.keys(),
            key=lambda k: self.agent_loads[k]
        )
        return next(a for a in agents if a.agent_id == least_loaded_id)
    
    def _capability_based(
        self,
        agents: List[BaseAgent],
        task: Dict[str, Any]
    ) -> BaseAgent:
        """Select agent based on capabilities"""
        required = task.get("required_capabilities", [])
        
        # Find agents with matching capabilities
        matching = [
            a for a in agents
            if all(cap in a.capabilities for cap in required)
        ]
        
        if matching:
            # Return least loaded from matching
            return self._least_loaded(matching)
        
        return agents[0]  # Fallback
    
    def record_task_assignment(self, agent_id: str):
        """Record task assignment"""
        self.agent_loads[agent_id] = self.agent_loads.get(agent_id, 0) + 1
    
    def record_task_completion(self, agent_id: str):
        """Record task completion"""
        if agent_id in self.agent_loads:
            self.agent_loads[agent_id] = max(0, self.agent_loads[agent_id] - 1)
```

### Deliverable 3.1: Load Balancing

- [ ] Load balancer implemented
- [ ] Multiple strategies supported
- [ ] Load tracking working
- [ ] Tests passing

**Commit your work:**
```bash
git add .
git commit -m "Implement load balancing mechanisms"
```

## Reflection Questions

1. **What are the trade-offs** between centralized and decentralized orchestration?
2. **How does Isorythm's adapter pattern** enable framework flexibility?
3. **When would you choose** sequential vs. parallel workflows?
4. **What load balancing strategy** works best for your use case?

## Next Steps

- Review Day 6 lesson
- Complete exercises
- Commit work
- Prepare for Day 7: Coordination Mechanisms

---

**Great progress!** Tomorrow we'll explore coordination mechanisms in depth.

