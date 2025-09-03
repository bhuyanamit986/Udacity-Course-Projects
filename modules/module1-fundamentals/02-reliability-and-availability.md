# Reliability and Availability

## Definitions

### Reliability
The probability that a system performs correctly for a specific duration under specific conditions.

### Availability
The percentage of time a system is operational and accessible when required for use.

## Key Metrics

### Availability Percentages
- **99%** = 87.6 hours downtime/year
- **99.9%** = 8.76 hours downtime/year
- **99.99%** = 52.56 minutes downtime/year
- **99.999%** = 5.26 minutes downtime/year

### Mean Time Between Failures (MTBF)
Average time between system failures

### Mean Time to Recovery (MTTR)
Average time to restore service after failure

### Availability Formula
```
Availability = MTBF / (MTBF + MTTR)
```

## Fault Tolerance Strategies

### 1. Redundancy
- **Active-Active**: Multiple systems running simultaneously
- **Active-Passive**: Backup system ready to take over
- **N+1 Redundancy**: N systems + 1 backup

### 2. Replication
- **Synchronous**: All replicas updated simultaneously
- **Asynchronous**: Replicas updated with delay
- **Multi-master**: Multiple writable replicas

### 3. Failover Mechanisms
- **Automatic failover**: System switches automatically
- **Manual failover**: Human intervention required
- **Graceful degradation**: Reduced functionality instead of complete failure

## High Availability Patterns

### 1. Load Balancer with Health Checks
```
Internet → Load Balancer → [Server1, Server2, Server3]
                ↓
            Health Checks
```

### 2. Database Master-Slave Setup
```
Application → Master DB (Write)
           → Slave DB (Read)
           → Slave DB (Read)
```

### 3. Multi-Region Deployment
```
Region 1: [App Servers] + [Database]
Region 2: [App Servers] + [Database Replica]
Region 3: [App Servers] + [Database Replica]
```

## Disaster Recovery

### Recovery Strategies
1. **Backup and Restore**: Regular backups, restore when needed
2. **Pilot Light**: Minimal version always running
3. **Warm Standby**: Scaled-down version running
4. **Hot Standby**: Full production environment ready

### Recovery Time Objective (RTO)
Maximum acceptable downtime

### Recovery Point Objective (RPO)
Maximum acceptable data loss

## Error Handling

### Circuit Breaker Pattern
```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
    
    def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "HALF_OPEN"
            else:
                raise Exception("Circuit breaker is OPEN")
        
        try:
            result = func(*args, **kwargs)
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise e
    
    def on_success(self):
        self.failure_count = 0
        self.state = "CLOSED"
    
    def on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = "OPEN"
```

### Retry Mechanisms
- **Fixed delay**: Wait same time between retries
- **Exponential backoff**: Increase wait time exponentially
- **Jittered backoff**: Add randomness to prevent thundering herd

### Bulkhead Pattern
Isolate critical resources to prevent cascade failures

## Monitoring and Alerting

### Key Metrics to Monitor
- **Response time**: 95th, 99th percentiles
- **Error rate**: 4xx, 5xx errors
- **Throughput**: Requests per second
- **Resource utilization**: CPU, memory, disk

### Alerting Best Practices
- Set meaningful thresholds
- Avoid alert fatigue
- Include runbooks
- Escalation procedures

## Real-World Examples

### Netflix Chaos Engineering
- Chaos Monkey: Randomly terminates instances
- Chaos Kong: Simulates entire region failures
- Builds confidence in system resilience

### Amazon's Approach
- Cell-based architecture
- Blast radius limitation
- Automated recovery systems

### Google's SRE Practices
- Error budgets
- SLI/SLO/SLA framework
- Blameless post-mortems

## Trade-offs

### Consistency vs Availability
- Strong consistency: All nodes see same data
- Eventual consistency: Nodes will converge
- Choose based on business requirements

### Cost vs Reliability
- Higher availability = higher costs
- Business impact analysis needed
- Cost-benefit optimization

## Exercise Problems

1. Design a system that guarantees 99.99% availability
2. How would you handle a database failure in a critical system?
3. Calculate the availability of a system with two components in series, each with 99.9% availability
4. Design a disaster recovery plan for an e-commerce platform

## Key Takeaways

- Plan for failures, don't just hope they won't happen
- Redundancy is key to high availability
- Monitor everything and alert intelligently
- Practice disaster recovery procedures
- Balance cost with reliability requirements
- Automate recovery processes where possible

## Next Steps

Move to: **03-performance-optimization.md**