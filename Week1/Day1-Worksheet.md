# Day 1 Worksheet: Introduction to Multi-Agent Systems

## Objectives

- Set up development environment for multi-agent systems
- Build your first simple multi-agent system
- Understand agent types and characteristics
- Practice basic agent communication

## Prerequisites Checklist

Before starting, ensure you have:

- [ ] Python 3.11+ installed
- [ ] Git installed and configured
- [ ] GitHub account set up
- [ ] Code editor (VS Code, PyCharm, etc.)
- [ ] Terminal/command line access

## Exercise 1: Environment Setup

### Step 1: Create Project Repository

```bash
# Create project directory
mkdir multi-agent-playbook
cd multi-agent-playbook

# Initialize Git repository
git init
git branch -M main
```

### Step 2: Set Up Virtual Environment

```bash
# Create virtual environment
python -m venv venv

# Activate virtual environment
# Windows:
venv\Scripts\activate
# Linux/Mac:
source venv/bin/activate

# Verify activation (should show venv path)
which python  # Linux/Mac
where python  # Windows
```

### Step 3: Install Dependencies

Create `requirements.txt`:

```txt
langchain>=0.1.0
crewai>=0.1.0
python-dotenv>=1.0.0
pytest>=7.4.0
```

Install dependencies:

```bash
pip install -r requirements.txt
```

### Step 4: Create Project Structure

```bash
mkdir -p agents communication coordination config tests
touch agents/__init__.py
touch communication/__init__.py
touch coordination/__init__.py
touch config/__init__.py
touch README.md
```

### Deliverable 1.1: Environment Setup

- [ ] Virtual environment created and activated
- [ ] Dependencies installed successfully
- [ ] Project structure created
- [ ] Git repository initialized
- [ ] Initial commit made

**Commit your setup:**
```bash
git add .
git commit -m "Initial project setup with environment configuration"
```

## Exercise 2: Build Your First Multi-Agent System

### Step 1: Create Base Agent Class

Create `agents/base_agent.py`:

```python
from abc import ABC, abstractmethod
from typing import Dict, Any, List
from dataclasses import dataclass
from datetime import datetime

@dataclass
class Message:
    sender: str
    recipient: str
    content: Dict[str, Any]
    timestamp: datetime = None
    
    def __post_init__(self):
        if self.timestamp is None:
            self.timestamp = datetime.now()

class BaseAgent(ABC):
    """Base class for all agents"""
    
    def __init__(self, agent_id: str, capabilities: List[str]):
        self.agent_id = agent_id
        self.capabilities = capabilities
        self.message_queue: List[Message] = []
        self.state: Dict[str, Any] = {}
    
    def receive_message(self, message: Message):
        """Receive a message from another agent"""
        if message.recipient == self.agent_id:
            self.message_queue.append(message)
            self.process_messages()
    
    def process_messages(self):
        """Process all messages in queue"""
        while self.message_queue:
            message = self.message_queue.pop(0)
            self.handle_message(message)
    
    @abstractmethod
    def handle_message(self, message: Message):
        """Handle a specific message - must be implemented by subclasses"""
        pass
    
    def send_message(self, recipient: 'BaseAgent', content: Dict[str, Any]):
        """Send a message to another agent"""
        message = Message(
            sender=self.agent_id,
            recipient=recipient.agent_id,
            content=content
        )
        recipient.receive_message(message)
    
    def get_state(self) -> Dict[str, Any]:
        """Get current agent state"""
        return self.state.copy()
    
    def update_state(self, key: str, value: Any):
        """Update agent state"""
        self.state[key] = value
```

### Step 2: Create Coordinator Agent

Create `agents/coordinator.py`:

```python
from agents.base_agent import BaseAgent, Message
from typing import List, Dict, Any

class CoordinatorAgent(BaseAgent):
    """Coordinates tasks among worker agents"""
    
    def __init__(self, agent_id: str = "coordinator"):
        super().__init__(agent_id, ["coordination", "task_distribution"])
        self.workers: List[BaseAgent] = []
        self.tasks: Dict[str, Dict[str, Any]] = {}
        self.task_counter = 0
    
    def register_worker(self, worker: BaseAgent):
        """Register a worker agent"""
        self.workers.append(worker)
        print(f"[{self.agent_id}] Registered worker: {worker.agent_id}")
    
    def submit_task(self, task_description: str) -> str:
        """Submit a task and return task ID"""
        self.task_counter += 1
        task_id = f"task-{self.task_counter}"
        
        self.tasks[task_id] = {
            "id": task_id,
            "description": task_description,
            "status": "pending",
            "assigned_to": None
        }
        
        self.distribute_task(task_id)
        return task_id
    
    def distribute_task(self, task_id: str):
        """Distribute task to available worker"""
        if not self.workers:
            print(f"[{self.agent_id}] No workers available for task {task_id}")
            return
        
        # Simple round-robin assignment
        worker = self.workers[len(self.tasks) % len(self.workers)]
        task = self.tasks[task_id]
        task["assigned_to"] = worker.agent_id
        task["status"] = "in_progress"
        
        self.send_message(worker, {
            "type": "task_assignment",
            "task_id": task_id,
            "task_description": task["description"]
        })
        
        print(f"[{self.agent_id}] Assigned {task_id} to {worker.agent_id}")
    
    def handle_message(self, message: Message):
        """Handle messages from workers"""
        content = message.content
        
        if content["type"] == "task_completed":
            task_id = content["task_id"]
            if task_id in self.tasks:
                self.tasks[task_id]["status"] = "completed"
                self.tasks[task_id]["result"] = content.get("result")
                print(f"[{self.agent_id}] Task {task_id} completed by {message.sender}")
        
        elif content["type"] == "task_failed":
            task_id = content["task_id"]
            if task_id in self.tasks:
                self.tasks[task_id]["status"] = "failed"
                print(f"[{self.agent_id}] Task {task_id} failed: {content.get('error')}")
```

### Step 3: Create Worker Agent

Create `agents/worker.py`:

```python
from agents.base_agent import BaseAgent, Message
from typing import List
import time
import random

class WorkerAgent(BaseAgent):
    """Worker agent that processes tasks"""
    
    def __init__(self, agent_id: str, capabilities: List[str], processing_time: float = 1.0):
        super().__init__(agent_id, capabilities)
        self.processing_time = processing_time
        self.current_task = None
        self.tasks_completed = 0
    
    def handle_message(self, message: Message):
        """Handle messages from coordinator"""
        content = message.content
        
        if content["type"] == "task_assignment":
            self.process_task(
                content["task_id"],
                content["task_description"],
                message.sender
            )
    
    def process_task(self, task_id: str, task_description: str, coordinator_id: str):
        """Process an assigned task"""
        self.current_task = task_id
        print(f"[{self.agent_id}] Starting task {task_id}: {task_description}")
        
        # Simulate processing time
        time.sleep(self.processing_time)
        
        # Simulate success/failure (90% success rate)
        success = random.random() > 0.1
        
        if success:
            result = f"Successfully processed: {task_description}"
            self.tasks_completed += 1
            
            # Find coordinator agent (in real system, would use message bus)
            # For now, we'll need to pass coordinator reference
            print(f"[{self.agent_id}] Completed task {task_id}")
        else:
            error = f"Failed to process: {task_description}"
            print(f"[{self.agent_id}] Task {task_id} failed")
    
    def get_stats(self) -> dict:
        """Get worker statistics"""
        return {
            "agent_id": self.agent_id,
            "tasks_completed": self.tasks_completed,
            "current_task": self.current_task
        }
```

### Step 4: Create Main Application

Create `main.py`:

```python
from agents.coordinator import CoordinatorAgent
from agents.worker import WorkerAgent

def main():
    """Main application demonstrating multi-agent system"""
    
    # Create coordinator
    coordinator = CoordinatorAgent("coordinator-1")
    
    # Create workers
    worker1 = WorkerAgent("worker-1", ["processing", "analysis"], processing_time=0.5)
    worker2 = WorkerAgent("worker-2", ["processing", "transformation"], processing_time=0.7)
    worker3 = WorkerAgent("worker-3", ["processing", "validation"], processing_time=0.6)
    
    # Register workers
    coordinator.register_worker(worker1)
    coordinator.register_worker(worker2)
    coordinator.register_worker(worker3)
    
    # Submit tasks
    tasks = [
        "Process customer data",
        "Analyze sales trends",
        "Generate report",
        "Validate data integrity",
        "Transform data format"
    ]
    
    task_ids = []
    for task in tasks:
        task_id = coordinator.submit_task(task)
        task_ids.append(task_id)
    
    # Wait for tasks to complete
    import time
    time.sleep(5)
    
    # Print results
    print("\n=== Task Status ===")
    for task_id in task_ids:
        task = coordinator.tasks.get(task_id, {})
        print(f"{task_id}: {task.get('status', 'unknown')}")
    
    # Print worker stats
    print("\n=== Worker Statistics ===")
    for worker in coordinator.workers:
        stats = worker.get_stats()
        print(f"{stats['agent_id']}: {stats['tasks_completed']} tasks completed")

if __name__ == "__main__":
    main()
```

### Step 5: Run Your System

```bash
python main.py
```

### Deliverable 2.1: First Multi-Agent System

- [ ] Base agent class implemented
- [ ] Coordinator agent implemented
- [ ] Worker agent implemented
- [ ] Main application runs successfully
- [ ] Agents communicate and coordinate tasks
- [ ] System handles multiple tasks

**Commit your implementation:**
```bash
git add .
git commit -m "Implement first multi-agent system with coordinator and workers"
```

## Exercise 3: Experiment with Agent Types

### Step 1: Create Reactive Agent

Create `agents/reactive_agent.py`:

```python
from agents.base_agent import BaseAgent, Message

class ReactiveAgent(BaseAgent):
    """Simple reactive agent that responds immediately to stimuli"""
    
    def __init__(self, agent_id: str, response_pattern: dict):
        super().__init__(agent_id, ["reactive"])
        self.response_pattern = response_pattern  # stimulus -> response mapping
    
    def handle_message(self, message: Message):
        """React immediately based on message content"""
        stimulus = message.content.get("type", "unknown")
        response = self.response_pattern.get(stimulus, "no_response")
        
        print(f"[{self.agent_id}] Reacted to {stimulus} with: {response}")
        
        # Send response back
        if message.sender:
            self.send_message(message.sender, {
                "type": "response",
                "stimulus": stimulus,
                "response": response
            })
```

### Step 2: Test Different Agent Behaviors

Modify `main.py` to include reactive agent:

```python
from agents.reactive_agent import ReactiveAgent

# Add to main function
reactive = ReactiveAgent("reactive-1", {
    "alert": "investigate",
    "warning": "monitor",
    "error": "escalate"
})

coordinator.send_message(reactive, {"type": "alert", "message": "System alert"})
```

### Deliverable 3.1: Agent Type Experimentation

- [ ] Reactive agent implemented
- [ ] Tested different agent behaviors
- [ ] Documented differences between agent types
- [ ] Reflected on when to use each type

## Exercise 4: Implement Bounded Autonomy Pattern

### Step 1: Add Step Counter to Worker Agent

Modify `agents/worker.py` to implement bounded autonomy (≤10 steps):

```python
class WorkerAgent(BaseAgent):
    """Worker agent with bounded autonomy (≤10 steps)"""
    
    MAX_STEPS = 10  # Research finding: 68% of successful deployments use ≤10 steps
    
    def __init__(self, agent_id: str, capabilities: List[str], processing_time: float = 1.0):
        super().__init__(agent_id, capabilities)
        self.processing_time = processing_time
        self.current_task = None
        self.tasks_completed = 0
        self.step_count = 0  # Track steps for bounded autonomy
    
    def process_task(self, task_id: str, task_description: str, coordinator_id: str):
        """Process task with bounded autonomy"""
        self.current_task = task_id
        self.step_count = 0
        
        print(f"[{self.agent_id}] Starting task {task_id}: {task_description}")
        
        # Simulate multi-step processing with bounded autonomy
        while self.step_count < self.MAX_STEPS:
            self.step_count += 1
            print(f"[{self.agent_id}] Step {self.step_count}/{self.MAX_STEPS}")
            
            # Simulate work
            time.sleep(self.processing_time / self.MAX_STEPS)
            
            # Check if task is complete (simulate completion condition)
            if self.step_count >= 5:  # Task completes after 5 steps
                break
        
        if self.step_count >= self.MAX_STEPS:
            print(f"[{self.agent_id}] WARNING: Reached max steps ({self.MAX_STEPS})")
            # In production, would escalate to human or coordinator
        
        result = f"Processed: {task_description} (completed in {self.step_count} steps)"
        self.tasks_completed += 1
        print(f"[{self.agent_id}] Completed task {task_id} in {self.step_count} steps")
```

### Step 2: Add Human-in-the-Loop Checkpoint

Create `coordination/human_checkpoint.py`:

```python
from typing import Dict, Any, Callable, Optional

class HumanCheckpoint:
    """Human-in-the-loop checkpoint (74% of successful systems use this)"""
    
    def __init__(self, approval_callback: Optional[Callable] = None):
        self.approval_callback = approval_callback
        self.pending_approvals: Dict[str, Dict[str, Any]] = {}
    
    def request_approval(self, request_id: str, context: Dict[str, Any]) -> bool:
        """Request human approval for critical decision"""
        self.pending_approvals[request_id] = {
            "context": context,
            "status": "pending",
            "timestamp": datetime.now()
        }
        
        print(f"\n[Human Checkpoint] Approval required for: {request_id}")
        print(f"Context: {context.get('description', 'No description')}")
        
        # In production, would integrate with UI/notification system
        # For now, simulate approval
        if self.approval_callback:
            return self.approval_callback(request_id, context)
        
        # Simulate: auto-approve for demo (in production, require actual human input)
        response = input("Approve? (y/n): ").lower() == 'y'
        
        self.pending_approvals[request_id]["status"] = "approved" if response else "rejected"
        return response
    
    def is_approved(self, request_id: str) -> bool:
        """Check if request is approved"""
        approval = self.pending_approvals.get(request_id, {})
        return approval.get("status") == "approved"

# Example usage in coordinator
class CoordinatorAgent(BaseAgent):
    def __init__(self, agent_id: str = "coordinator", human_checkpoint: Optional[HumanCheckpoint] = None):
        super().__init__(agent_id, ["coordination", "task_distribution"])
        self.workers: List[BaseAgent] = []
        self.tasks: Dict[str, Dict[str, Any]] = {}
        self.task_counter = 0
        self.human_checkpoint = human_checkpoint or HumanCheckpoint()
    
    def submit_task(self, task_description: str) -> str:
        """Submit task with human approval for high-risk tasks"""
        self.task_counter += 1
        task_id = f"task-{self.task_counter}"
        
        # Check if task requires human approval (e.g., high-risk operations)
        requires_approval = self._requires_approval(task_description)
        
        if requires_approval:
            approved = self.human_checkpoint.request_approval(
                f"task-{task_id}",
                {"description": task_description, "type": "task_submission"}
            )
            if not approved:
                print(f"[{self.agent_id}] Task {task_id} rejected by human checkpoint")
                return None
        
        self.tasks[task_id] = {
            "id": task_id,
            "description": task_description,
            "status": "pending",
            "assigned_to": None
        }
        
        self.distribute_task(task_id)
        return task_id
    
    def _requires_approval(self, task_description: str) -> bool:
        """Determine if task requires human approval"""
        high_risk_keywords = ["delete", "modify", "critical", "production", "security"]
        return any(keyword in task_description.lower() for keyword in high_risk_keywords)
```

### Deliverable 4.1: Bounded Autonomy and Human-in-the-Loop

- [ ] Bounded autonomy implemented (≤10 steps)
- [ ] Human checkpoint system implemented
- [ ] High-risk tasks require approval
- [ ] Step counting and limits enforced
- [ ] Tested with various task types

**Commit your implementation:**
```bash
git add .
git commit -m "Add bounded autonomy and human-in-the-loop patterns"
```

## Reflection Questions

1. **What are the key differences** between reactive and deliberative agents?
2. **When would you choose** a multi-agent system over a monolithic system?
3. **Based on research findings**, when should you avoid using multi-agent systems?
4. **Why is bounded autonomy (≤10 steps)** important for production systems?
5. **How does human-in-the-loop** improve system reliability?
6. **What challenges** did you encounter while building your first system?
7. **How would you improve** the simple task distribution system?
8. **What questions** do you have about multi-agent systems?

## Troubleshooting

### Issue: Agents not receiving messages
- **Solution**: Ensure message recipient matches agent ID exactly
- **Check**: Message queue processing is called

### Issue: Tasks not completing
- **Solution**: Verify worker agents are processing messages
- **Check**: Coordinator is sending messages correctly

### Issue: Import errors
- **Solution**: Ensure all `__init__.py` files exist
- **Check**: Python path includes project directory

## Next Steps

- Review Day 1 lesson materials
- Complete all exercises
- Commit your work to Git
- Prepare for Day 2: Agent Communication and Coordination

## Deliverables Checklist

- [ ] Environment setup complete
- [ ] First multi-agent system implemented
- [ ] Different agent types tested
- [ ] Bounded autonomy pattern implemented (≤10 steps)
- [ ] Human-in-the-loop checkpoint implemented
- [ ] Code committed to Git repository
- [ ] Reflection questions answered
- [ ] Ready for Day 2

---

**Great work!** You've built your first multi-agent system. Tomorrow we'll dive deeper into agent communication and coordination patterns.

