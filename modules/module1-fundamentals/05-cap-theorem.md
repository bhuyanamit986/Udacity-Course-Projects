# CAP Theorem

## What is CAP Theorem?

The CAP theorem, also known as Brewer's theorem, states that in a distributed system, you can only guarantee **two out of three** properties simultaneously:

- **C**onsistency
- **A**vailability  
- **P**artition Tolerance

## The Three Properties

### Consistency (C)
- All nodes see the same data simultaneously
- Every read receives the most recent write
- Linear consistency across the system

### Availability (A)
- System remains operational
- Every request receives a response
- No downtime or service interruption

### Partition Tolerance (P)
- System continues to operate despite network failures
- Network can lose or delay messages
- Nodes can become isolated

## CAP Combinations

### CA (Consistency + Availability)
- **When**: No network partitions
- **Example**: Traditional RDBMS in single data center
- **Trade-off**: Cannot handle network partitions
- **Real-world**: MySQL, PostgreSQL (single node)

### CP (Consistency + Partition Tolerance)
- **When**: Network partitions may occur
- **Example**: Distributed databases prioritizing consistency
- **Trade-off**: May become unavailable during partitions
- **Real-world**: MongoDB, Redis Cluster, HBase

### AP (Availability + Partition Tolerance)
- **When**: Network partitions may occur
- **Example**: Systems prioritizing availability
- **Trade-off**: May serve stale data during partitions
- **Real-world**: Cassandra, DynamoDB, CouchDB

## CAP in Practice

### Network Partitions are Reality
```
In distributed systems, network partitions WILL happen:
- Hardware failures
- Network congestion
- Configuration errors
- Geographic issues
```

Therefore, the choice is really between **CP** or **AP**.

## Real-World Examples

### Google's Approach (CP)
```
Spanner Database:
✓ Consistency: Strong consistency globally
✓ Partition Tolerance: Handles network issues
✗ Availability: May become unavailable during partitions
```

### Amazon's Approach (AP)
```
DynamoDB:
✗ Consistency: Eventually consistent
✓ Availability: Always available for reads/writes
✓ Partition Tolerance: Handles network partitions
```

### Facebook's Approach (Mixed)
```
Different systems for different needs:
- User data: CP (consistency important)
- News feed: AP (availability important)
- Analytics: AP (eventual consistency OK)
```

## CAP Theorem Limitations

### 1. Not Binary
- Consistency and availability exist on spectrums
- Systems can provide different guarantees for different operations

### 2. PACELC Extension
**PACELC**: In case of **P**artition, choose between **A**vailability and **C**onsistency, **E**lse choose between **L**atency and **C**onsistency

### 3. Time Dimension
- CAP doesn't consider time
- Systems may sacrifice one property temporarily

## Design Decisions Framework

### 1. Analyze Requirements
```
Questions to ask:
- How critical is data consistency?
- Can the system tolerate downtime?
- What are the latency requirements?
- How often do network partitions occur?
```

### 2. Business Impact Analysis
```
Consistency Impact:
- Financial: Critical (money transfers)
- Social Media: Moderate (likes/comments)
- Analytics: Low (reporting delays OK)

Availability Impact:
- E-commerce: High (lost sales)
- Internal tools: Moderate
- Batch processing: Low
```

### 3. Hybrid Approaches

#### Multi-Model Systems
```python
class HybridDataStore:
    def __init__(self):
        self.consistent_store = PostgreSQL()  # For critical data
        self.available_store = Cassandra()    # For high-volume data
    
    def write_critical(self, key, value):
        return self.consistent_store.write(key, value)
    
    def write_analytics(self, key, value):
        return self.available_store.write(key, value)
```

## Consistency Patterns

### 1. Read-Your-Writes Consistency
```python
class ReadYourWritesCache:
    def __init__(self):
        self.write_cache = {}
        self.read_cache = {}
    
    def write(self, user_id, key, value):
        # Store in user's write cache
        if user_id not in self.write_cache:
            self.write_cache[user_id] = {}
        self.write_cache[user_id][key] = value
        
        # Async write to database
        self.async_write_to_db(key, value)
    
    def read(self, user_id, key):
        # Check user's write cache first
        if user_id in self.write_cache and key in self.write_cache[user_id]:
            return self.write_cache[user_id][key]
        
        # Fallback to regular cache/database
        return self.read_from_db(key)
```

### 2. Monotonic Read Consistency
```python
class MonotonicReadCache:
    def __init__(self):
        self.user_versions = {}
        self.replicas = []
    
    def read(self, user_id, key):
        user_version = self.user_versions.get(user_id, 0)
        
        # Find replica with version >= user_version
        for replica in self.replicas:
            if replica.version >= user_version:
                result = replica.read(key)
                self.user_versions[user_id] = replica.version
                return result
        
        # Fallback to most up-to-date replica
        latest_replica = max(self.replicas, key=lambda r: r.version)
        return latest_replica.read(key)
```

### 3. Session Consistency
- Consistency within a user session
- Different users may see different states

## CAP in Different Architectures

### Microservices and CAP
```
Service A (CP) ←→ Service B (AP) ←→ Service C (CP)
```

Each service can make its own CAP choice based on requirements.

### Event Sourcing
```python
class EventStore:
    def __init__(self):
        self.events = []
        self.snapshots = {}
    
    def append_event(self, event):
        # Strongly consistent event append
        self.events.append(event)
        self.replicate_to_followers(event)
    
    def get_current_state(self, entity_id):
        # Eventually consistent projections
        return self.build_projection(entity_id)
```

## Measuring Consistency

### Consistency Metrics
```python
def measure_consistency_lag():
    """Measure time between write and read consistency"""
    write_time = time.time()
    write_to_master(key, value)
    
    while True:
        read_value = read_from_replica(key)
        if read_value == value:
            return time.time() - write_time
        time.sleep(0.01)
```

### Consistency Testing
```python
def test_linearizability():
    """Test if system provides linearizable consistency"""
    operations = []
    
    # Concurrent operations from multiple clients
    for client in clients:
        operation = client.execute_operation()
        operations.append(operation)
    
    # Check if there exists a valid sequential ordering
    return is_linearizable(operations)
```

## CAP Theorem in System Design Interviews

### Common Questions
1. "How does CAP theorem influence your design?"
2. "What happens during a network partition?"
3. "How would you ensure consistency in this system?"

### Answer Framework
```
1. Identify the CAP requirements
   - What consistency level is needed?
   - How critical is availability?
   - Will network partitions occur?

2. Make explicit trade-offs
   - "We choose AP because..."
   - "We choose CP because..."

3. Explain mitigation strategies
   - How to handle the sacrificed property
   - Monitoring and alerting
   - Recovery procedures
```

## Example System Designs

### Banking System (CP)
```
Design Choice: CP
Reasoning: Money transfers require strong consistency
Implementation:
- Synchronous replication
- Two-phase commit for transactions
- Accept downtime during network issues
```

### Social Media Feed (AP)
```
Design Choice: AP
Reasoning: Users expect always-available feeds
Implementation:
- Asynchronous replication
- Eventual consistency for posts
- Graceful degradation during partitions
```

### E-commerce Inventory (Hybrid)
```
Critical Path (CP): Payment processing
Non-critical Path (AP): Product recommendations
Implementation:
- Strong consistency for payments
- Eventual consistency for recommendations
```

## Advanced Concepts

### Jepsen Testing
- Distributed systems testing framework
- Simulates network partitions
- Verifies consistency claims

### Linearizability vs Serializability
- **Linearizability**: Real-time ordering
- **Serializability**: Some sequential ordering exists

### Byzantine Fault Tolerance
- Handles malicious or corrupted nodes
- Requires 3f+1 nodes to tolerate f failures
- Used in blockchain systems

## Exercise Problems

1. Design a distributed counter that must be strongly consistent
2. How would you implement eventual consistency for a global user profile system?
3. Analyze the CAP trade-offs for a real-time collaborative editor
4. Design a system that provides different consistency guarantees for different data types

## Common Misconceptions

### "NoSQL = Eventual Consistency"
- Many NoSQL systems offer tunable consistency
- Can configure for strong consistency when needed

### "ACID = Strong Consistency"
- ACID is about single-node guarantees
- Distributed ACID requires additional protocols

### "Partition Tolerance is Optional"
- In distributed systems, partitions will happen
- Must choose between C and A during partitions

## Key Takeaways

- CAP theorem is fundamental to distributed system design
- Network partitions are inevitable in distributed systems
- Choose CP or AP based on business requirements
- Consider PACELC for a more complete picture
- Different parts of system can make different CAP choices
- Consistency is a spectrum, not binary
- Test your consistency assumptions
- Monitor consistency metrics in production

## Next Steps

Move to: **06-load-balancing-fundamentals.md**