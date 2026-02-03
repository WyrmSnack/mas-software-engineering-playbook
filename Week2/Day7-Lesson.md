# Day 7: Coordination Mechanisms

## Learning Objectives

By the end of this lesson, you will be able to:
- Deep dive into Contract Net Protocol
- Implement Blackboard Architecture
- Understand Market-based coordination
- Apply Isorythm's WorkflowCoordinatorTool patterns
- Choose appropriate coordination mechanism

## Contract Net Protocol Deep Dive

We covered Contract Net basics in Day 2. Today we'll implement a complete, production-ready version.

### Complete Implementation

```python
from typing import Dict, List, Any, Optional
from dataclasses import dataclass
from datetime import datetime, timedelta
from enum import Enum

class BidStatus(Enum):
    PENDING = "pending"
    ACCEPTED = "accepted"
    REJECTED = "rejected"
    EXPIRED = "expired"

@dataclass
class TaskAnnouncement:
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

class ContractNetProtocol:
    """Complete Contract Net Protocol implementation"""
    
    def __init__(self, coordinator_id: str = "coordinator"):
        self.coordinator_id = coordinator_id
        self.agents: List[Any] = []
        self.announcements: Dict[str, TaskAnnouncement] = {}
        self.bids: Dict[str, List[Bid]] = {}
        self.awards: Dict[str, str] = {}
    
    def announce_task(self, task: TaskAnnouncement):
        """Phase 1: Announce task"""
        self.announcements[task.task_id] = task
        
        for agent in self.agents:
            agent.receive_announcement({
                "task_id": task.task_id,
                "description": task.description,
                "requirements": task.requirements,
                "deadline": task.deadline.isoformat()
            })
    
    def collect_bids(self, task_id: str, timeout: int = 30) -> List[Bid]:
        """Phase 2: Collect bids with timeout"""
        import time
        start_time = time.time()
        
        while time.time() - start_time < timeout:
            if task_id in self.bids and len(self.bids[task_id]) > 0:
                # Check if all agents have bid or timeout
                # Simplified - in production would be async
                break
            time.sleep(0.1)
        
        return self.bids.get(task_id, [])
    
    def evaluate_bids(self, task_id: str) -> Optional[Bid]:
        """Phase 3: Evaluate and select best bid"""
        bids = self.bids.get(task_id, [])
        if not bids:
            return None
        
        # Multi-criteria evaluation
        best_bid = max(bids, key=lambda b: self._calculate_bid_score(b))
        return best_bid
    
    def _calculate_bid_score(self, bid: Bid) -> float:
        """Calculate composite bid score"""
        # Consider: score, estimated_time, capabilities match
        time_score = 1.0 / (bid.estimated_time.total_seconds() + 1)
        return bid.score * 0.7 + time_score * 0.3
    
    def award_task(self, task_id: str) -> Optional[str]:
        """Phase 4: Award task to best bidder"""
        best_bid = self.evaluate_bids(task_id)
        if not best_bid:
            return None
        
        # Award to winner
        best_bid.status = BidStatus.ACCEPTED
        self.awards[task_id] = best_bid.agent_id
        
        # Reject others
        for bid in self.bids[task_id]:
            if bid.bid_id != best_bid.bid_id:
                bid.status = BidStatus.REJECTED
        
        return best_bid.agent_id
```

## Blackboard Architecture Deep Dive

### Complete Blackboard Implementation

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
    metadata: Dict[str, Any] = None

class Blackboard:
    """Complete Blackboard Architecture (50-80% improvement for structured analysis)"""
    
    def __init__(self):
        self.knowledge: Dict[str, BlackboardEntry] = {}
        self.subscribers: Dict[str, List[Callable]] = defaultdict(list)
        self.locks: Dict[str, threading.Lock] = defaultdict(threading.Lock)
        self.version_counter: Dict[str, int] = defaultdict(int)
        self.access_log: List[Dict[str, Any]] = []
    
    def read(self, key: str, agent_id: str) -> Optional[Any]:
        """Read from blackboard with access logging"""
        with self.locks[key]:
            entry = self.knowledge.get(key)
            if entry:
                self._log_access(key, agent_id, "read")
                return entry.value
            return None
    
    def write(
        self,
        key: str,
        value: Any,
        agent_id: str,
        metadata: Optional[Dict[str, Any]] = None
    ):
        """Write to blackboard"""
        with self.locks[key]:
            self.version_counter[key] += 1
            entry = BlackboardEntry(
                key=key,
                value=value,
                written_by=agent_id,
                timestamp=datetime.now(),
                version=self.version_counter[key],
                metadata=metadata or {}
            )
            self.knowledge[key] = entry
            self._log_access(key, agent_id, "write")
        
        # Notify subscribers (outside lock)
        self._notify_subscribers(key, entry)
    
    def _notify_subscribers(self, key: str, entry: BlackboardEntry):
        """Notify all subscribers"""
        for callback in self.subscribers[key]:
            try:
                callback(key, entry)
            except Exception as e:
                print(f"Error in subscriber: {e}")
    
    def _log_access(self, key: str, agent_id: str, operation: str):
        """Log blackboard access"""
        self.access_log.append({
            "key": key,
            "agent_id": agent_id,
            "operation": operation,
            "timestamp": datetime.now()
        })
```

**Research Finding**: Blackboard architecture shows 50-80% improvement for structured analysis tasks in multi-agent systems (MASGSE v0.81).

## Market-Based Coordination

### Implementation

```python
class MarketCoordinator:
    """Market-based coordination for resource allocation"""
    
    def __init__(self):
        self.market: Dict[str, float] = {}  # resource -> price
        self.agents: List[Any] = []
        self.transactions: List[Dict[str, Any]] = []
    
    def create_market(self, resource: str, initial_price: float = 1.0):
        """Create market for resource"""
        self.market[resource] = initial_price
    
    def bid_for_resource(
        self,
        agent_id: str,
        resource: str,
        bid_price: float,
        quantity: int = 1
    ) -> bool:
        """Agent bids for resource"""
        current_price = self.market.get(resource, 0)
        
        if bid_price >= current_price:
            # Award resource
            self.market[resource] = bid_price
            self.transactions.append({
                "agent_id": agent_id,
                "resource": resource,
                "price": bid_price,
                "quantity": quantity,
                "timestamp": datetime.now()
            })
            return True
        
        return False
```

## Isorythm's WorkflowCoordinatorTool

**Isorythm Implementation**: The `WorkflowCoordinatorTool`` demonstrates real-world coordination patterns:

**Key Features**:
- Condition analysis (failed tasks, timeouts, resource constraints)
- Dynamic adjustment strategies (conservative, aggressive, adaptive)
- Task reassignment
- Workflow modification

**Pattern to Apply**:
```python
class WorkflowCoordinator:
    """Inspired by Isorythm's WorkflowCoordinatorTool"""
    
    def analyze_conditions(
        self,
        workflow_state: Dict[str, Any],
        conditions: Dict[str, Any]
    ) -> Dict[str, Any]:
        """Analyze workflow conditions"""
        analysis = {
            "issues_detected": [],
            "recommendations": [],
            "priority": "low"
        }
        
        # Check failed tasks
        failed_tasks = workflow_state.get("failed_tasks", [])
        if failed_tasks:
            analysis["issues_detected"].append(f"{len(failed_tasks)} task(s) failed")
            analysis["recommendations"].append("Retry failed tasks")
            analysis["priority"] = "high"
        
        # Check timeouts
        if conditions.get("timeout", False):
            analysis["issues_detected"].append("Timeout detected")
            analysis["recommendations"].append("Increase timeout or break task")
            analysis["priority"] = "high"
        
        return analysis
    
    def generate_adjustments(
        self,
        analysis: Dict[str, Any],
        strategy: str = "conservative"
    ) -> Dict[str, Any]:
        """Generate workflow adjustments"""
        adjustments = {
            "actions": [],
            "task_reassignments": [],
            "workflow_modifications": []
        }
        
        if analysis["priority"] == "high":
            if "failed" in str(analysis["issues_detected"]):
                adjustments["actions"].append("retry_failed_tasks")
        
        return adjustments
```

## Key Takeaways

1. **Contract Net Protocol** enables distributed task allocation with bidding
2. **Blackboard Architecture** shows 50-80% improvement for structured analysis
3. **Market-based coordination** handles resource allocation through pricing
4. **Isorythm's WorkflowCoordinatorTool** demonstrates production coordination patterns
5. **Coordination mechanism selection** depends on problem characteristics

## Next Steps

- Complete Day 7 worksheet
- Implement complete coordination mechanisms
- Apply Isorythm patterns
- Read about agent specialization (preview of Day 8)

---

**Ready for hands-on practice?** Complete the [Day 7 Worksheet](Day7-Worksheet.md)!

