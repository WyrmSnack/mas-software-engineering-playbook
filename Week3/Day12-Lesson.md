# Day 12: Adaptive and Learning Agents

## Learning Objectives

By the end of this lesson, you will be able to:
- Understand reinforcement learning in agents
- Implement adaptive behavior patterns
- Design learning from experience mechanisms
- Apply online learning strategies
- Build agents that improve over time

## Reinforcement Learning in Agents

**Pattern**: Agents learn through trial and error, receiving rewards/penalties.

### Basic RL Pattern

```python
class LearningAgent(BaseAgent):
    """Agent that learns from experience"""
    
    def __init__(self, agent_id: str, capabilities: List[str]):
        super().__init__(agent_id, capabilities)
        self.q_table: Dict[str, Dict[str, float]] = {}  # state -> action -> value
        self.learning_rate = 0.1
        self.discount_factor = 0.9
        self.exploration_rate = 0.1
    
    def choose_action(self, state: str) -> str:
        """Choose action using epsilon-greedy"""
        if random.random() < self.exploration_rate:
            return self._explore(state)
        else:
            return self._exploit(state)
    
    def learn(self, state: str, action: str, reward: float, next_state: str):
        """Update Q-table using Q-learning"""
        if state not in self.q_table:
            self.q_table[state] = {}
        if action not in self.q_table[state]:
            self.q_table[state][action] = 0.0
        
        current_q = self.q_table[state][action]
        max_next_q = max(self.q_table.get(next_state, {}).values(), default=0.0)
        
        # Q-learning update
        new_q = current_q + self.learning_rate * (
            reward + self.discount_factor * max_next_q - current_q
        )
        self.q_table[state][action] = new_q
```

## Adaptive Behavior Patterns

### Pattern: Performance-Based Adaptation

```python
class AdaptiveAgent(BaseAgent):
    """Agent that adapts behavior based on performance"""
    
    def __init__(self, agent_id: str, capabilities: List[str]):
        super().__init__(agent_id, capabilities)
        self.performance_history: List[float] = []
        self.strategy = "conservative"
    
    def adapt_strategy(self):
        """Adapt strategy based on recent performance"""
        if len(self.performance_history) < 5:
            return
        
        recent_performance = sum(self.performance_history[-5:]) / 5
        
        if recent_performance > 0.8:
            self.strategy = "aggressive"  # High performance, be more aggressive
        elif recent_performance < 0.5:
            self.strategy = "conservative"  # Low performance, be cautious
        else:
            self.strategy = "balanced"
```

## Learning from Experience

Agents can learn by:
1. **Trial and Error**: Try different approaches, learn what works
2. **Observation**: Learn from other agents' successes/failures
3. **Feedback**: Use human or system feedback to improve
4. **Pattern Recognition**: Identify patterns in successful behaviors

## Key Takeaways

1. **Reinforcement learning** enables agents to learn from experience
2. **Adaptive behavior** improves performance over time
3. **Learning mechanisms** can be simple or complex
4. **Online learning** allows continuous improvement
5. **Experience-based learning** is essential for dynamic environments

## Next Steps

- Complete Day 12 worksheet
- Implement learning agents
- Test adaptive behavior
- Read about distributed systems (preview of Day 13)

---

**Ready for hands-on practice?** Complete the [Day 12 Worksheet](Day12-Worksheet.md)!

