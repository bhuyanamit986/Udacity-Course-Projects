# Consistency Models

## What is Consistency?

Consistency refers to the agreement of data across different nodes in a distributed system. It ensures that all nodes see the same data at the same time or follow specific rules about data visibility.

## Types of Consistency

### 1. Strong Consistency
- All nodes see the same data simultaneously
- Reads always return the most recent write
- **Example**: Traditional RDBMS with ACID properties

```
Time: T1    T2    T3    T4
Node A: W(x=1) ---- R(x=1) ----
Node B: ---- ---- R(x=1) ----
Node C: ---- ---- R(x=1) ----
```

**Pros**: Data accuracy, simple reasoning
**Cons**: Higher latency, reduced availability

### 2. Eventual Consistency
- System will become consistent over time
- Temporary inconsistencies allowed
- **Example**: DNS, Amazon S3

```
Time: T1    T2    T3    T4
Node A: W(x=1) ---- R(x=1) ----
Node B: ---- R(x=0) R(x=1) ----
Node C: ---- R(x=0) ---- R(x=1)
```

**Pros**: High availability, better performance
**Cons**: Temporary inconsistencies, complex application logic

### 3. Weak Consistency
- No guarantees about when all nodes will be consistent
- **Example**: Live video streaming, gaming

### 4. Causal Consistency
- Causally related operations are seen in the same order
- Concurrent operations may be seen in different orders

## ACID Properties

### Atomicity
- Transactions are all-or-nothing
- Either all operations succeed or all fail

```sql
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT; -- Both operations succeed or both fail
```

### Consistency
- Database remains in valid state
- All constraints are satisfied

### Isolation
- Concurrent transactions don't interfere
- **Isolation Levels**:
  - Read Uncommitted
  - Read Committed
  - Repeatable Read
  - Serializable

### Durability
- Committed transactions persist
- Survive system failures

## BASE Properties (NoSQL Alternative)

### Basically Available
- System remains operational
- May have reduced functionality

### Soft State
- System state may change over time
- Even without new input

### Eventually Consistent
- System will become consistent
- Given enough time

## Consensus Algorithms

### 1. Raft Consensus
```
Leader Election → Log Replication → Safety
```

**Phases**:
1. **Leader Election**: One node becomes leader
2. **Log Replication**: Leader replicates entries
3. **Commitment**: Entries applied to state machine

### 2. Paxos
- More complex than Raft
- Proven correctness
- Used in Google's Chubby

### 3. PBFT (Practical Byzantine Fault Tolerance)
- Handles malicious nodes
- Used in blockchain systems

## Distributed System Challenges

### Split-Brain Problem
```
Network Partition:
[Node A] ---- X ---- [Node B, Node C]
```
- Multiple leaders elected
- Conflicting decisions made
- **Solution**: Quorum-based decisions

### Quorum Systems
```
N = Total replicas
W = Write quorum
R = Read quorum

For strong consistency: W + R > N
```

**Example**: N=5, W=3, R=3
- Need 3 nodes for writes
- Need 3 nodes for reads
- Guarantees overlap

## Consistency Patterns in Practice

### 1. Master-Slave Replication
```
Application → Master (Write)
           → Slave 1 (Read)
           → Slave 2 (Read)
```

**Consistency Issues**:
- Read-after-write problems
- Replication lag

**Solutions**:
- Read from master for recent writes
- Session consistency
- Monotonic read consistency

### 2. Multi-Master Replication
```
App Region 1 → Master 1 ←→ Master 2 ← App Region 2
```

**Conflict Resolution**:
- Last-write-wins
- Vector clocks
- Application-level resolution

### 3. Sharding with Consistency
```python
def get_shard(key):
    return hash(key) % num_shards

def write_data(key, value):
    shard = get_shard(key)
    return shards[shard].write(key, value)

def read_data(key):
    shard = get_shard(key)
    return shards[shard].read(key)
```

## Consistency in Different Systems

### Relational Databases
- ACID compliance
- Strong consistency by default
- Configurable isolation levels

### NoSQL Databases

#### MongoDB
- Strong consistency within replica set
- Configurable read/write concerns
```javascript
// Strong consistency read
db.users.find().readConcern("majority")

// Eventual consistency read
db.users.find().readConcern("local")
```

#### Cassandra
- Tunable consistency
- Consistency level per operation
```cql
-- Strong consistency
SELECT * FROM users WHERE id = ? USING CONSISTENCY QUORUM;

-- Eventual consistency
SELECT * FROM users WHERE id = ? USING CONSISTENCY ONE;
```

#### DynamoDB
- Eventually consistent reads by default
- Strongly consistent reads available
```python
# Eventually consistent
response = table.get_item(Key={'id': '123'})

# Strongly consistent
response = table.get_item(
    Key={'id': '123'},
    ConsistentRead=True
)
```

## Handling Inconsistency

### 1. Conflict-Free Replicated Data Types (CRDTs)
```python
class GCounter:
    """Grow-only counter CRDT"""
    def __init__(self, node_id):
        self.node_id = node_id
        self.counts = {}
    
    def increment(self):
        if self.node_id not in self.counts:
            self.counts[self.node_id] = 0
        self.counts[self.node_id] += 1
    
    def value(self):
        return sum(self.counts.values())
    
    def merge(self, other):
        for node_id, count in other.counts.items():
            self.counts[node_id] = max(
                self.counts.get(node_id, 0), 
                count
            )
```

### 2. Vector Clocks
```python
class VectorClock:
    def __init__(self, node_id):
        self.node_id = node_id
        self.clock = {}
    
    def tick(self):
        self.clock[self.node_id] = self.clock.get(self.node_id, 0) + 1
    
    def update(self, other_clock):
        for node, timestamp in other_clock.items():
            self.clock[node] = max(
                self.clock.get(node, 0), 
                timestamp
            )
        self.tick()
```

### 3. Operational Transformation
- Used in collaborative editing
- Google Docs, real-time collaboration
- Transform operations based on concurrent changes

## Consistency Trade-offs

### Consistency vs Performance
```
Strong Consistency:
+ Data accuracy
+ Simple application logic
- Higher latency
- Reduced availability

Eventual Consistency:
+ Better performance
+ Higher availability
- Complex application logic
- Temporary inconsistencies
```

### Choosing Consistency Models

#### Financial Systems
- **Requirement**: Strong consistency
- **Reason**: Money transfers must be accurate
- **Solution**: ACID databases, two-phase commit

#### Social Media
- **Requirement**: Eventual consistency acceptable
- **Reason**: Slight delays in likes/comments OK
- **Solution**: NoSQL with eventual consistency

#### Gaming Leaderboards
- **Requirement**: Strong consistency for rankings
- **Reason**: Fair competition
- **Solution**: Centralized scoring with replication

## Implementation Strategies

### 1. Two-Phase Commit (2PC)
```
Phase 1: Prepare
Coordinator → "Can you commit?" → All participants
All participants → "Yes/No" → Coordinator

Phase 2: Commit
If all yes: Coordinator → "Commit" → All participants
If any no: Coordinator → "Abort" → All participants
```

### 2. Saga Pattern
```python
class OrderSaga:
    def execute(self, order_data):
        try:
            # Step 1: Reserve inventory
            reservation_id = inventory_service.reserve(order_data.items)
            
            # Step 2: Process payment
            payment_id = payment_service.charge(order_data.payment)
            
            # Step 3: Create order
            order_id = order_service.create(order_data)
            
            return order_id
        except Exception as e:
            # Compensating transactions
            if 'payment_id' in locals():
                payment_service.refund(payment_id)
            if 'reservation_id' in locals():
                inventory_service.release(reservation_id)
            raise e
```

## Exercise Problems

1. Design a consistency model for a distributed chat application
2. How would you handle consistency in a multi-region e-commerce system?
3. Compare the trade-offs between strong and eventual consistency for a social media feed
4. Design a conflict resolution strategy for a collaborative document editor

## Key Takeaways

- Consistency is a spectrum, not binary
- Choose consistency model based on business requirements
- Strong consistency comes with performance costs
- Eventual consistency requires careful application design
- Monitoring and alerting are crucial for inconsistency detection
- Consider conflict resolution strategies upfront
- Test consistency behavior under network partitions

## Next Steps

Move to: **05-cap-theorem.md**