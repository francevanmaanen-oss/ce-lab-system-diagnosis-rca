**Long-term Fix:**
```markdown
# Long-term Fix Proposal

## Problem
Fixed connection pool doesn't adapt to traffic

## Solution
Implement dynamic connection pool scaling

## Implementation
```python
# Auto-scaling connection pool
class DynamicConnectionPool:
    def __init__(self):
        self.min_connections = 20
        self.max_connections = 100
        self.current_connections = 20
    
    def scale_based_on_traffic(self, current_traffic):
        target = current_traffic / 20  # 20 requests per connection
        target = max(self.min_connections, min(target, self.max_connections))
        
        if target > self.current_connections:
            self.expand_pool(target)
        elif target < self.current_connections * 0.7:
            self.shrink_pool(target)
Benefits
Handles traffic spikes automatically
Reduces cost during low traffic
No manual intervention needed
Monitoring
Add CloudWatch metric: ConnectionPoolSize
Alert if auto-scaling fails
Testing Plan
Load test with varying traffic (100-2000 req/min)
Verify pool scales up/down correctly
Ensure no connection leaks