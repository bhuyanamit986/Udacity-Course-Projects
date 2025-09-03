# Databases: SQL vs NoSQL

## Database Fundamentals

### ACID Properties (SQL Databases)
- **Atomicity**: All or nothing transactions
- **Consistency**: Data integrity maintained
- **Isolation**: Concurrent operations don't interfere
- **Durability**: Committed data survives failures

### BASE Properties (NoSQL Databases)
- **Basically Available**: System remains operational
- **Soft state**: Data may change over time
- **Eventually consistent**: Consistency achieved over time

## SQL Databases (RDBMS)

### Characteristics
- Structured data with predefined schema
- ACID compliance
- Complex queries with JOINs
- Mature ecosystem and tooling

### Popular SQL Databases
- **PostgreSQL**: Advanced features, JSON support
- **MySQL**: Fast, widely adopted
- **Oracle**: Enterprise features
- **SQL Server**: Microsoft ecosystem

### When to Use SQL Databases
```
✓ Complex relationships between data
✓ ACID transactions required
✓ Complex queries and reporting
✓ Mature tooling needed
✓ Strong consistency requirements
```

### SQL Database Patterns

#### Master-Slave Replication
```sql
-- Master (Write operations)
INSERT INTO users (name, email) VALUES ('John', 'john@example.com');
UPDATE users SET status = 'active' WHERE id = 1;

-- Slave (Read operations)
SELECT * FROM users WHERE status = 'active';
SELECT COUNT(*) FROM orders WHERE created_at > '2023-01-01';
```

#### Sharding
```python
class DatabaseSharding:
    def __init__(self, shards):
        self.shards = shards
    
    def get_shard(self, user_id):
        return self.shards[user_id % len(self.shards)]
    
    def insert_user(self, user_data):
        shard = self.get_shard(user_data['id'])
        return shard.execute(
            "INSERT INTO users (id, name, email) VALUES (?, ?, ?)",
            user_data['id'], user_data['name'], user_data['email']
        )
```

#### Federation
```python
class DatabaseFederation:
    def __init__(self):
        self.user_db = UserDatabase()
        self.product_db = ProductDatabase()
        self.order_db = OrderDatabase()
    
    def get_user(self, user_id):
        return self.user_db.get_user(user_id)
    
    def get_product(self, product_id):
        return self.product_db.get_product(product_id)
    
    def create_order(self, order_data):
        return self.order_db.create_order(order_data)
```

## NoSQL Databases

### Document Databases
- **MongoDB**: JSON-like documents, flexible schema
- **CouchDB**: Multi-master replication
- **Amazon DocumentDB**: MongoDB-compatible

```javascript
// MongoDB Example
{
  "_id": ObjectId("..."),
  "name": "John Doe",
  "email": "john@example.com",
  "addresses": [
    {
      "type": "home",
      "street": "123 Main St",
      "city": "New York"
    },
    {
      "type": "work",
      "street": "456 Office Blvd",
      "city": "San Francisco"
    }
  ]
}
```

### Key-Value Stores
- **Redis**: In-memory, caching
- **Amazon DynamoDB**: Managed, serverless
- **Riak**: Distributed, fault-tolerant

```python
# Redis Example
redis_client.set("user:1001", json.dumps({
    "name": "John Doe",
    "email": "john@example.com"
}))

user_data = json.loads(redis_client.get("user:1001"))
```

### Column-Family
- **Cassandra**: High write throughput
- **HBase**: Hadoop ecosystem
- **Amazon SimpleDB**: Managed service

```cql
-- Cassandra Example
CREATE TABLE user_timeline (
    user_id UUID,
    tweet_id TIMEUUID,
    content TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, tweet_id)
) WITH CLUSTERING ORDER BY (tweet_id DESC);
```

### Graph Databases
- **Neo4j**: Property graph model
- **Amazon Neptune**: Managed graph database
- **ArangoDB**: Multi-model database

```cypher
// Neo4j Example - Social network
CREATE (john:Person {name: 'John', age: 30})
CREATE (jane:Person {name: 'Jane', age: 25})
CREATE (john)-[:FOLLOWS]->(jane)

// Find friends of friends
MATCH (user:Person)-[:FOLLOWS]->(friend)-[:FOLLOWS]->(fof)
WHERE user.name = 'John'
RETURN fof.name
```

## Database Selection Guide

### Decision Matrix

| Requirement | SQL | Document | Key-Value | Column | Graph |
|-------------|-----|----------|-----------|---------|-------|
| Complex queries | ✅ | ⚠️ | ❌ | ⚠️ | ✅ |
| Transactions | ✅ | ⚠️ | ❌ | ❌ | ⚠️ |
| Scalability | ⚠️ | ✅ | ✅ | ✅ | ⚠️ |
| Flexibility | ❌ | ✅ | ✅ | ⚠️ | ✅ |
| Consistency | ✅ | ⚠️ | ⚠️ | ⚠️ | ✅ |

### Use Case Examples

#### E-commerce Platform
```python
class EcommerceDataLayer:
    def __init__(self):
        # SQL for transactions and inventory
        self.transaction_db = PostgreSQL()
        
        # Document DB for product catalog
        self.catalog_db = MongoDB()
        
        # Key-value for session data
        self.session_db = Redis()
        
        # Graph DB for recommendations
        self.recommendation_db = Neo4j()
```

#### Social Media Platform
```python
class SocialMediaDataLayer:
    def __init__(self):
        # SQL for user profiles and relationships
        self.user_db = MySQL()
        
        # Column-family for timeline data
        self.timeline_db = Cassandra()
        
        # Document DB for posts and media
        self.content_db = MongoDB()
        
        # Key-value for real-time features
        self.realtime_db = Redis()
```

## Database Scaling Patterns

### 1. Read Replicas
```python
class ReadReplicaManager:
    def __init__(self, master, replicas):
        self.master = master
        self.replicas = replicas
        self.replica_index = 0
    
    def write(self, query, params):
        return self.master.execute(query, params)
    
    def read(self, query, params):
        replica = self.replicas[self.replica_index]
        self.replica_index = (self.replica_index + 1) % len(self.replicas)
        return replica.execute(query, params)
```

### 2. Database Sharding Strategies

#### Range-Based Sharding
```python
def get_shard_by_range(user_id):
    if user_id < 1000000:
        return "shard_1"
    elif user_id < 2000000:
        return "shard_2"
    else:
        return "shard_3"
```

#### Hash-Based Sharding
```python
def get_shard_by_hash(user_id):
    return f"shard_{hash(user_id) % num_shards}"
```

#### Directory-Based Sharding
```python
class ShardDirectory:
    def __init__(self):
        self.directory = {
            "users_1-1000": "shard_1",
            "users_1001-2000": "shard_2",
            "users_2001-3000": "shard_3"
        }
    
    def get_shard(self, user_id):
        for range_key, shard in self.directory.items():
            start, end = self.parse_range(range_key)
            if start <= user_id <= end:
                return shard
```

### 3. Database Federation
```python
class DatabaseFederation:
    """Split databases by function"""
    def __init__(self):
        self.user_service_db = UserDB()
        self.product_service_db = ProductDB()
        self.order_service_db = OrderDB()
        self.analytics_db = AnalyticsDB()
    
    def route_query(self, table_name, query):
        if table_name.startswith('user_'):
            return self.user_service_db.execute(query)
        elif table_name.startswith('product_'):
            return self.product_service_db.execute(query)
        # ... etc
```

## Database Performance Optimization

### Indexing Best Practices
```sql
-- Good: Selective index
CREATE INDEX idx_active_users ON users(status) WHERE status = 'active';

-- Good: Composite index for common query
CREATE INDEX idx_order_user_date ON orders(user_id, created_at);

-- Bad: Over-indexing
CREATE INDEX idx_user_name ON users(name);
CREATE INDEX idx_user_email ON users(email);
CREATE INDEX idx_user_phone ON users(phone);
-- Too many indexes slow down writes
```

### Query Optimization
```sql
-- Bad: N+1 query
SELECT * FROM users;
-- For each user:
SELECT * FROM orders WHERE user_id = ?;

-- Good: JOIN query
SELECT u.*, o.* 
FROM users u 
LEFT JOIN orders o ON u.id = o.user_id;

-- Good: Pagination
SELECT * FROM posts 
ORDER BY created_at DESC 
LIMIT 20 OFFSET 100;
```

### Connection Management
```python
class DatabaseConnectionPool:
    def __init__(self, max_connections=20):
        self.pool = queue.Queue(maxsize=max_connections)
        self.max_connections = max_connections
        self.current_connections = 0
    
    def get_connection(self):
        if not self.pool.empty():
            return self.pool.get()
        elif self.current_connections < self.max_connections:
            conn = self.create_connection()
            self.current_connections += 1
            return conn
        else:
            # Wait for available connection
            return self.pool.get(block=True, timeout=30)
    
    def return_connection(self, conn):
        if conn.is_healthy():
            self.pool.put(conn)
        else:
            self.current_connections -= 1
```

## Polyglot Persistence

### Multi-Database Architecture
```python
class PolyglotPersistence:
    """Use different databases for different needs"""
    
    def __init__(self):
        # User profiles and relationships
        self.user_db = PostgreSQL()
        
        # Product catalog with flexible schema
        self.catalog_db = MongoDB()
        
        # High-frequency trading data
        self.trading_db = Cassandra()
        
        # Real-time analytics
        self.analytics_db = InfluxDB()
        
        # Search functionality
        self.search_db = Elasticsearch()
        
        # Cache layer
        self.cache = Redis()
    
    def create_user(self, user_data):
        user_id = self.user_db.create_user(user_data)
        # Cache user data
        self.cache.set(f"user:{user_id}", user_data, ttl=3600)
        return user_id
    
    def search_products(self, query):
        return self.search_db.search("products", query)
    
    def log_event(self, event_data):
        self.analytics_db.write_point(event_data)
```

## Database Migration Strategies

### Zero-Downtime Migration
```python
class DatabaseMigration:
    def migrate_to_new_system(self):
        """Dual-write pattern for migration"""
        
        # Phase 1: Dual write
        def write_data(data):
            old_db.write(data)  # Primary
            try:
                new_db.write(data)  # Secondary
            except Exception as e:
                log_error(f"New DB write failed: {e}")
        
        # Phase 2: Backfill historical data
        self.backfill_historical_data()
        
        # Phase 3: Dual read with verification
        def read_data(key):
            old_result = old_db.read(key)
            new_result = new_db.read(key)
            if old_result != new_result:
                log_inconsistency(key, old_result, new_result)
            return old_result  # Still primary
        
        # Phase 4: Switch to new system
        def read_data(key):
            return new_db.read(key)  # New system is primary
```

## Exercise Problems

1. Design a database architecture for a ride-sharing application
2. How would you migrate from a monolithic database to microservices?
3. Choose appropriate databases for different parts of a social media platform
4. Design a sharding strategy for a global messaging application

## Key Takeaways

- Different databases serve different purposes
- SQL databases excel at complex queries and transactions
- NoSQL databases provide scalability and flexibility
- Polyglot persistence is often the best approach
- Database choice significantly impacts system architecture
- Plan for data migration from the beginning
- Monitor database performance and scaling metrics

## Next Steps

Move to: **02-caching-systems.md**