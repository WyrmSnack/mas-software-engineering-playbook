# Day 2 Worksheet: Agent Communication and Coordination

## Objectives

- Implement message bus communication pattern
- Build Contract Net Protocol for task distribution
- Create Blackboard architecture for shared knowledge
- Understand coordination overhead and trade-offs
- Practice with Isorythm-inspired coordination patterns

## Prerequisites Checklist

Before starting, ensure you have:

- [ ] Completed Day 1 exercises
- [ ] Understanding of basic agent communication
- [ ] Python environment set up
- [ ] Git repository initialized

## Exercise 1: Implement Message Bus

### Step 1: Create Message Bus Class

Create `communication/message_bus.py`:

```python
from typing import Dict, List, Callable, Any, Optional
from dataclasses import dataclass
from datetime import datetime
from collections import defaultdict
import threading

@dataclass
class Message:
    topic: str
    payload: Dict[str, Any]
    sender: str
    timestamp: datetime = None
    
    def __post_init__(self):
        if self.timestamp is None:
            self.timestamp = datetime.now()

class MessageBus:
    """Message bus for agent communication (inspired by Isorythm patterns)"""
    
    def __init__(self):
        self.subscribers: Dict[str, List[Callable]] = defaultdict(list)
        self.message_history: List[Message] = []
        self.lock = threading.Lock()
    
    def subscribe(self, topic: str, callback: Callable[[Message], None]):
        """Subscribe to a topic"""
        with self.lock:
            self.subscribers[topic].append(callback)
    
    def unsubscribe(self, topic: str, callback: Callable[[Message], None]):
        """Unsubscribe from a topic"""
        with self.lock:
            if callback in self.subscribers[topic]:
                self.subscribers[topic].remove(callback)
    
    def publish(self, topic: str, payload: Dict[str, Any], sender: str = "system"):
        """Publish message to topic"""
        message = Message(topic=topic, payload=payload, sender=sender)
        
        with self.lock:
            self.message_history.append(message)
            callbacks = self.subscribers[topic].copy()
        
        # Notify subscribers (outside lock to avoid deadlock)
        for callback in callbacks:
            try:
                callback(message)
            except Exception as e:
                print(f"Error in subscriber callback: {e}")
    
    def get_history(self, topic: Optional[str] = None) -> List[Message]:
        """Get message history, optionally filtered by topic"""
        with self.lock:
            if topic:
                return [msg for msg in self.message_history if msg.topic == topic]
            return self.message_history.copy()
```

### Step 2: Integrate Message Bus with Agents

Update `agents/base_agent.py`:

```python
from communication.message_bus import MessageBus, Message

class BaseAgent(ABC):
    """Base agent with message bus integration"""
    
    def __init__(self, agent_id: str, capabilities: List[str], message_bus: Optional[MessageBus] = None):
        self.agent_id = agent_id
        self.capabilities = capabilities
        self.message_bus = message_bus
        self.message_queue: List[Message] = []
        self.state: Dict[str, Any] = {}
        
        # Subscribe to relevant topics
        if self.message_bus:
            self.message_bus.subscribe(f"agent.{self.agent_id}", self._handle_message)
    
    def _handle_message(self, message: Message):
        """Internal message handler"""
        if message.payload.get("recipient") == self.agent_id:
            self.message_queue.append(message)
            self.process_messages()
    
    def publish(self, topic: str, payload: Dict[str, Any]):
        """Publish message to bus"""
        if self.message_bus:
            self.message_bus.publish(topic, payload, sender=self.agent_id)
    
    # ... rest of BaseAgent implementation
```

### Step 3: Test Message Bus

Create `tests/test_message_bus.py`:

```python
from communication.message_bus import MessageBus

def test_message_bus():
    bus = MessageBus()
    received_messages = []
    
    def handler(message):
        received_messages.append(message)
    
    bus.subscribe("test.topic", handler)
    bus.publish("test.topic", {"data": "test"})
    
    assert len(received_messages) == 1
    assert received_messages[0].payload["data"] == "test"
    print("✅ Message bus test passed")

if __name__ == "__main__":
    test_message_bus()
```

### Deliverable 1.1: Message Bus Implementation

- [ ] Message bus class implemented
- [ ] Subscribe/publish functionality working
- [ ] Message history tracking
- [ ] Thread-safe implementation
- [ ] Integration with agents
- [ ] Tests passing

**Commit your work:**
```bash
git add .
git commit -m "Implement message bus communication pattern"
```

## Exercise 2: Contract Net Protocol

### Step 1: Create Contract Net Components

Create `coordination/contract_net.py`:

```python
from typing import Dict, List, Optional, Any
from dataclasses import dataclass
from datetime import datetime, timedelta
from enum import Enum

class BidStatus(Enum):
    PENDING = "pending"
    ACCEPTED = "accepted"
    REJECTED = "rejected"

@dataclass
class Task:
    task_id: str
    description: str
    requirements: Dict[str, Any]
    deadline: datetime
    priority: int = 1

@dataclass
class Bid:
    bid_id: str
    agent_id: str
    task_id: str
    score: float
    estimated_time: timedelta
    capabilities: List[str]
    status: BidStatus = BidStatus.PENDING
    timestamp: datetime = None
    
    def __post_init__(self):
        if self.timestamp is None:
            self.timestamp = datetime.now()

class ContractNetCoordinator:
    """Contract Net Protocol coordinator"""
    
    def __init__(self, coordinator_id: str = "coordinator"):
        self.coordinator_id = coordinator_id
        self.agents: List[Any] = []  # List of agents
        self.active_tasks: Dict[str, Task] = {}
        self.bids: Dict[str, List[Bid]] = {}  # task_id -> bids
        self.awards: Dict[str, str] = {}  # task_id -> agent_id
    
    def register_agent(self, agent):
        """Register an agent"""
        self.agents.append(agent)
        agent.set_coordinator(self)
    
    def announce_task(self, task: Task):
        """Announce task to all agents (Step 1)"""
        self.active_tasks[task.task_id] = task
        
        announcement = {
            "type": "task_announcement",
            "task_id": task.task_id,
            "description": task.description,
            "requirements": task.requirements,
            "deadline": task.deadline.isoformat(),
            "priority": task.priority
        }
        
        # Broadcast to all agents
        for agent in self.agents:
            agent.receive_announcement(announcement)
        
        print(f"[{self.coordinator_id}] Announced task {task.task_id}")
    
    def receive_bid(self, bid: Bid):
        """Receive bid from agent (Step 2)"""
        if bid.task_id not in self.bids:
            self.bids[bid.task_id] = []
        
        self.bids[bid.task_id].append(bid)
        print(f"[{self.coordinator_id}] Received bid {bid.bid_id} from {bid.agent_id} for task {bid.task_id}")
    
    def evaluate_bids(self, task_id: str) -> Optional[Bid]:
        """Evaluate bids and select best (Step 3)"""
        if task_id not in self.bids or not self.bids[task_id]:
            return None
        
        bids = self.bids[task_id]
        
        # Simple evaluation: highest score wins
        # In production, would consider multiple factors
        best_bid = max(bids, key=lambda b: b.score)
        
        return best_bid
    
    def award_task(self, task_id: str):
        """Award task to best bidder (Step 4)"""
        best_bid = self.evaluate_bids(task_id)
        
        if not best_bid:
            print(f"[{self.coordinator_id}] No bids for task {task_id}")
            return None
        
        # Award to best bidder
        best_bid.status = BidStatus.ACCEPTED
        self.awards[task_id] = best_bid.agent_id
        
        # Reject other bids
        for bid in self.bids[task_id]:
            if bid.bid_id != best_bid.bid_id:
                bid.status = BidStatus.REJECTED
                # Notify rejected agents
                for agent in self.agents:
                    if agent.agent_id == bid.agent_id:
                        agent.receive_rejection(bid.task_id)
        
        # Notify winner
        for agent in self.agents:
            if agent.agent_id == best_bid.agent_id:
                agent.receive_award(task_id)
        
        print(f"[{self.coordinator_id}] Awarded task {task_id} to {best_bid.agent_id}")
        return best_bid.agent_id
```

### Step 2: Create Bidding Agent

Create `agents/bidding_agent.py`:

```python
from agents.base_agent import BaseAgent
from coordination.contract_net import ContractNetCoordinator, Task, Bid, BidStatus
from typing import List, Dict, Any, Optional

class BiddingAgent(BaseAgent):
    """Agent that participates in Contract Net Protocol"""
    
    def __init__(self, agent_id: str, capabilities: List[str], message_bus=None):
        super().__init__(agent_id, capabilities, message_bus)
        self.coordinator: Optional[ContractNetCoordinator] = None
        self.pending_announcements: Dict[str, Task] = {}
        self.active_bids: Dict[str, Bid] = {}
    
    def set_coordinator(self, coordinator: ContractNetCoordinator):
        """Set coordinator reference"""
        self.coordinator = coordinator
    
    def receive_announcement(self, announcement: Dict[str, Any]):
        """Receive task announcement and decide whether to bid"""
        task = Task(
            task_id=announcement["task_id"],
            description=announcement["description"],
            requirements=announcement["requirements"],
            deadline=datetime.fromisoformat(announcement["deadline"]),
            priority=announcement.get("priority", 1)
        )
        
        self.pending_announcements[task.task_id] = task
        
        # Evaluate if we can handle this task
        if self._can_handle_task(task):
            self._submit_bid(task)
        else:
            print(f"[{self.agent_id}] Cannot handle task {task.task_id}")
    
    def _can_handle_task(self, task: Task) -> bool:
        """Check if agent can handle task based on capabilities"""
        required = task.requirements.get("capabilities", [])
        return all(cap in self.capabilities for cap in required)
    
    def _submit_bid(self, task: Task):
        """Submit bid for task"""
        # Calculate bid score based on capabilities match
        score = self._calculate_bid_score(task)
        
        bid = Bid(
            bid_id=f"bid-{self.agent_id}-{task.task_id}",
            agent_id=self.agent_id,
            task_id=task.task_id,
            score=score,
            estimated_time=timedelta(seconds=10),  # Simplified
            capabilities=self.capabilities
        )
        
        self.active_bids[task.task_id] = bid
        
        if self.coordinator:
            self.coordinator.receive_bid(bid)
    
    def _calculate_bid_score(self, task: Task) -> float:
        """Calculate bid score (higher is better)"""
        # Simple scoring: match capabilities
        required = task.requirements.get("capabilities", [])
        matches = sum(1 for cap in required if cap in self.capabilities)
        return matches / len(required) if required else 0.5
    
    def receive_award(self, task_id: str):
        """Receive task award"""
        print(f"[{self.agent_id}] Awarded task {task_id}")
        if task_id in self.pending_announcements:
            task = self.pending_announcements.pop(task_id)
            self._execute_task(task)
    
    def receive_rejection(self, task_id: str):
        """Receive bid rejection"""
        print(f"[{self.agent_id}] Bid rejected for task {task_id}")
        if task_id in self.active_bids:
            self.active_bids[task_id].status = BidStatus.REJECTED
    
    def _execute_task(self, task: Task):
        """Execute awarded task"""
        print(f"[{self.agent_id}] Executing task {task.task_id}: {task.description}")
        # Task execution logic here
```

### Step 3: Test Contract Net Protocol

Create `tests/test_contract_net.py`:

```python
from coordination.contract_net import ContractNetCoordinator, Task
from agents.bidding_agent import BiddingAgent
from datetime import datetime, timedelta

def test_contract_net():
    coordinator = ContractNetCoordinator()
    
    # Create agents with different capabilities
    agent1 = BiddingAgent("agent-1", ["processing", "analysis"])
    agent2 = BiddingAgent("agent-2", ["processing", "transformation"])
    agent3 = BiddingAgent("agent-3", ["analysis", "validation"])
    
    coordinator.register_agent(agent1)
    coordinator.register_agent(agent2)
    coordinator.register_agent(agent3)
    
    # Create and announce task
    task = Task(
        task_id="task-1",
        description="Process and analyze data",
        requirements={"capabilities": ["processing", "analysis"]},
        deadline=datetime.now() + timedelta(hours=1)
    )
    
    coordinator.announce_task(task)
    
    # Wait for bids (in real system, would be async)
    import time
    time.sleep(0.1)
    
    # Award task
    winner = coordinator.award_task("task-1")
    
    assert winner is not None
    print(f"✅ Contract Net test passed. Winner: {winner}")

if __name__ == "__main__":
    test_contract_net()
```

### Deliverable 2.1: Contract Net Protocol

- [ ] Contract Net coordinator implemented
- [ ] Bidding agent implemented
- [ ] Task announcement working
- [ ] Bid collection working
- [ ] Task award mechanism working
- [ ] Tests passing

**Commit your work:**
```bash
git add .
git commit -m "Implement Contract Net Protocol for task distribution"
```

## Exercise 3: Blackboard Architecture

### Step 1: Create Blackboard System

Create `coordination/blackboard.py`:

```python
from typing import Dict, Any, List, Callable, Optional
from dataclasses import dataclass
from datetime import datetime
import threading
from collections import defaultdict

@dataclass
class BlackboardEntry:
    key: str
    value: Any
    written_by: str
    timestamp: datetime
    version: int = 1

class Blackboard:
    """Blackboard architecture for shared knowledge"""
    
    def __init__(self):
        self.knowledge: Dict[str, BlackboardEntry] = {}
        self.subscribers: Dict[str, List[Callable]] = defaultdict(list)
        self.locks: Dict[str, threading.Lock] = defaultdict(threading.Lock)
        self.version_counter: Dict[str, int] = defaultdict(int)
    
    def read(self, key: str) -> Optional[Any]:
        """Read value from blackboard"""
        with self.locks[key]:
            entry = self.knowledge.get(key)
            return entry.value if entry else None
    
    def read_entry(self, key: str) -> Optional[BlackboardEntry]:
        """Read full entry from blackboard"""
        with self.locks[key]:
            return self.knowledge.get(key)
    
    def write(self, key: str, value: Any, agent_id: str):
        """Write value to blackboard"""
        with self.locks[key]:
            self.version_counter[key] += 1
            entry = BlackboardEntry(
                key=key,
                value=value,
                written_by=agent_id,
                timestamp=datetime.now(),
                version=self.version_counter[key]
            )
            self.knowledge[key] = entry
        
        # Notify subscribers (outside lock)
        self._notify_subscribers(key, entry)
    
    def subscribe(self, key: str, callback: Callable[[str, BlackboardEntry], None]):
        """Subscribe to updates for a key"""
        self.subscribers[key].append(callback)
    
    def unsubscribe(self, key: str, callback: Callable):
        """Unsubscribe from updates"""
        if callback in self.subscribers[key]:
            self.subscribers[key].remove(callback)
    
    def _notify_subscribers(self, key: str, entry: BlackboardEntry):
        """Notify all subscribers of update"""
        for callback in self.subscribers[key]:
            try:
                callback(key, entry)
            except Exception as e:
                print(f"Error in blackboard subscriber: {e}")
    
    def get_all_keys(self) -> List[str]:
        """Get all keys in blackboard"""
        with threading.Lock():
            return list(self.knowledge.keys())
    
    def clear(self, key: Optional[str] = None):
        """Clear blackboard entry or all entries"""
        if key:
            with self.locks[key]:
                if key in self.knowledge:
                    del self.knowledge[key]
        else:
            with threading.Lock():
                self.knowledge.clear()
```

### Step 2: Create Blackboard-Aware Agent

Create `agents/blackboard_agent.py`:

```python
from agents.base_agent import BaseAgent
from coordination.blackboard import Blackboard, BlackboardEntry
from typing import List, Optional

class BlackboardAgent(BaseAgent):
    """Agent that uses blackboard for collaboration"""
    
    def __init__(self, agent_id: str, capabilities: List[str], blackboard: Blackboard, message_bus=None):
        super().__init__(agent_id, capabilities, message_bus)
        self.blackboard = blackboard
        self.subscribed_keys: List[str] = []
    
    def read_blackboard(self, key: str) -> Optional[Any]:
        """Read from blackboard"""
        return self.blackboard.read(key)
    
    def write_blackboard(self, key: str, value: Any):
        """Write to blackboard"""
        self.blackboard.write(key, value, self.agent_id)
        print(f"[{self.agent_id}] Wrote to blackboard: {key} = {value}")
    
    def subscribe_to_key(self, key: str):
        """Subscribe to blackboard updates"""
        if key not in self.subscribed_keys:
            self.blackboard.subscribe(key, self._on_blackboard_update)
            self.subscribed_keys.append(key)
    
    def _on_blackboard_update(self, key: str, entry: BlackboardEntry):
        """Handle blackboard update"""
        print(f"[{self.agent_id}] Blackboard updated: {key} by {entry.written_by}")
        self.handle_blackboard_update(key, entry)
    
    def handle_blackboard_update(self, key: str, entry: BlackboardEntry):
        """Override this method to handle updates"""
        pass
```

### Step 3: Collaborative Problem-Solving Example

Create `examples/blackboard_collaboration.py`:

```python
from coordination.blackboard import Blackboard
from agents.blackboard_agent import BlackboardAgent

class ProblemSolverAgent(BlackboardAgent):
    """Agent that contributes to problem-solving"""
    
    def __init__(self, agent_id: str, expertise: str, blackboard: Blackboard):
        super().__init__(agent_id, [expertise], blackboard)
        self.expertise = expertise
    
    def contribute_solution(self, problem_id: str, solution: Any):
        """Contribute solution to blackboard"""
        key = f"problem.{problem_id}.solution.{self.expertise}"
        self.write_blackboard(key, solution)
    
    def handle_blackboard_update(self, key: str, entry: BlackboardEntry):
        """React to other agents' contributions"""
        if "problem" in key and entry.written_by != self.agent_id:
            # Read other solutions and potentially build upon them
            problem_id = key.split(".")[1]
            other_solution = entry.value
            print(f"[{self.agent_id}] Noted {entry.written_by}'s solution for {problem_id}")

# Example usage
def collaborative_solving_example():
    blackboard = Blackboard()
    
    # Create specialized agents
    analyst = ProblemSolverAgent("analyst", "analysis", blackboard)
    designer = ProblemSolverAgent("designer", "design", blackboard)
    implementer = ProblemSolverAgent("implementer", "implementation", blackboard)
    
    # Subscribe to problem updates
    analyst.subscribe_to_key("problem.design")
    designer.subscribe_to_key("problem.implementation")
    
    # Agents contribute solutions
    analyst.contribute_solution("task-1", {"analysis": "Requirements identified"})
    designer.contribute_solution("task-1", {"design": "Architecture proposed"})
    implementer.contribute_solution("task-1", {"code": "Implementation complete"})
    
    # Read final solution
    final_solution = {
        "analysis": blackboard.read("problem.task-1.solution.analysis"),
        "design": blackboard.read("problem.task-1.solution.design"),
        "implementation": blackboard.read("problem.task-1.solution.implementation")
    }
    
    print("Final collaborative solution:", final_solution)

if __name__ == "__main__":
    collaborative_solving_example()
```

### Deliverable 3.1: Blackboard Architecture

- [ ] Blackboard class implemented
- [ ] Thread-safe read/write operations
- [ ] Subscription mechanism working
- [ ] Blackboard-aware agents implemented
- [ ] Collaborative example working
- [ ] Tests passing

**Commit your work:**
```bash
git add .
git commit -m "Implement Blackboard architecture for shared knowledge"
```

## Exercise 4: Coordination Overhead Analysis

### Step 1: Measure Communication Overhead

Create `analysis/coordination_overhead.py`:

```python
import time
from communication.message_bus import MessageBus
from coordination.contract_net import ContractNetCoordinator, Task
from datetime import datetime, timedelta

def measure_message_bus_overhead(num_messages: int, num_agents: int):
    """Measure message bus communication overhead"""
    bus = MessageBus()
    agents = []
    message_count = [0]
    
    def handler(msg):
        message_count[0] += 1
    
    # Create agents and subscribe
    for i in range(num_agents):
        bus.subscribe(f"topic.{i}", handler)
        agents.append(f"agent-{i}")
    
    # Measure publish time
    start = time.time()
    for i in range(num_messages):
        bus.publish(f"topic.{i % num_agents}", {"data": f"message-{i}"})
    end = time.time()
    
    overhead_per_message = (end - start) / num_messages
    print(f"Message Bus Overhead: {overhead_per_message*1000:.2f}ms per message")
    print(f"Total time: {end - start:.2f}s for {num_messages} messages")
    return overhead_per_message

def measure_contract_net_overhead(num_tasks: int, num_agents: int):
    """Measure Contract Net Protocol overhead"""
    coordinator = ContractNetCoordinator()
    
    # Create agents (simplified)
    from agents.bidding_agent import BiddingAgent
    agents = []
    for i in range(num_agents):
        agent = BiddingAgent(f"agent-{i}", ["capability"], None)
        coordinator.register_agent(agent)
        agents.append(agent)
    
    # Measure task announcement and bidding
    start = time.time()
    for i in range(num_tasks):
        task = Task(
            task_id=f"task-{i}",
            description=f"Task {i}",
            requirements={"capabilities": ["capability"]},
            deadline=datetime.now() + timedelta(hours=1)
        )
        coordinator.announce_task(task)
        # Wait for bids (simplified)
        time.sleep(0.01)
        coordinator.award_task(f"task-{i}")
    end = time.time()
    
    overhead_per_task = (end - start) / num_tasks
    print(f"Contract Net Overhead: {overhead_per_task*1000:.2f}ms per task")
    print(f"Total time: {end - start:.2f}s for {num_tasks} tasks")
    return overhead_per_task

if __name__ == "__main__":
    print("=== Coordination Overhead Analysis ===")
    print("\n1. Message Bus:")
    measure_message_bus_overhead(100, 10)
    
    print("\n2. Contract Net Protocol:")
    measure_contract_net_overhead(10, 5)
```

### Step 2: Reflection on Overhead

Answer these questions:

1. **At what point** does coordination overhead exceed benefits?
2. **Which pattern** has lower overhead: message bus or Contract Net?
3. **How would you optimize** communication for a large system?
4. **When would you choose** each coordination mechanism?

### Deliverable 4.1: Coordination Overhead Analysis

- [ ] Overhead measurement implemented
- [ ] Message bus overhead measured
- [ ] Contract Net overhead measured
- [ ] Reflection questions answered
- [ ] Optimization strategies documented

## Reflection Questions

1. **What are the trade-offs** between direct communication and message bus?
2. **When would Contract Net Protocol** be better than simple task assignment?
3. **How does Blackboard architecture** enable collaborative problem-solving?
4. **What coordination overhead** did you observe in your implementations?
5. **How would you design** a system to minimize coordination overhead?
6. **What patterns from Isorythm** could you apply to your system?

## Troubleshooting

### Issue: Messages not being received
- **Solution**: Check topic subscriptions match publish topics
- **Check**: Message bus is properly initialized and shared

### Issue: Contract Net bids not collected
- **Solution**: Ensure agents are registered before task announcement
- **Check**: Bid submission timing (may need async handling)

### Issue: Blackboard race conditions
- **Solution**: Verify locks are properly used
- **Check**: Thread-safe operations

## Next Steps

- Review Day 2 lesson materials
- Complete all exercises
- Commit your work to Git
- Prepare for Day 3: Agent Architectures and Patterns

## Deliverables Checklist

- [ ] Message bus implemented and tested
- [ ] Contract Net Protocol implemented
- [ ] Blackboard architecture implemented
- [ ] Coordination overhead analyzed
- [ ] Code committed to Git repository
- [ ] Reflection questions answered
- [ ] Ready for Day 3

---

**Great work!** You've implemented three key coordination patterns. Tomorrow we'll explore agent architectures and design patterns.

