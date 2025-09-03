# Performance Optimization

## Performance Fundamentals

### Key Performance Metrics

#### Latency
- **Definition**: Time to process a single request
- **Measurement**: Milliseconds (ms)
- **Types**:
  - Network latency
  - Processing latency
  - Database latency

#### Throughput
- **Definition**: Number of operations per unit time
- **Measurement**: Requests per second (RPS), Transactions per second (TPS)
- **Relationship**: `Throughput = Concurrency / Latency`

#### Response Time
- **Definition**: End-to-end time from request to response
- **Components**: Network time + Processing time + Queue time

### Performance Laws

#### Little's Law
```
L = λ × W
```
- L = Average number of requests in system
- λ = Average arrival rate
- W = Average time spent in system

#### Amdahl's Law
```
Speedup = 1 / ((1 - P) + P/N)
```
- P = Proportion of parallelizable work
- N = Number of processors

## Optimization Strategies

### 1. Caching

#### Cache Levels
```
Browser → CDN → Load Balancer → App Cache → Database Cache → Database
```

#### Cache Patterns

**Cache-Aside (Lazy Loading)**
```python
def get_user(user_id):
    # Check cache first
    user = cache.get(f"user:{user_id}")
    if user is None:
        # Cache miss - fetch from database
        user = database.get_user(user_id)
        # Store in cache for future requests
        cache.set(f"user:{user_id}", user, ttl=300)
    return user
```

**Write-Through**
```python
def update_user(user_id, user_data):
    # Update database
    database.update_user(user_id, user_data)
    # Update cache
    cache.set(f"user:{user_id}", user_data, ttl=300)
```

**Write-Behind (Write-Back)**
```python
def update_user(user_id, user_data):
    # Update cache immediately
    cache.set(f"user:{user_id}", user_data, ttl=300)
    # Queue database update for later
    queue.enqueue("update_user_db", user_id, user_data)
```

#### Cache Eviction Policies
- **LRU** (Least Recently Used)
- **LFU** (Least Frequently Used)
- **FIFO** (First In, First Out)
- **TTL** (Time To Live)

### 2. Database Optimization

#### Indexing Strategies
```sql
-- B-tree index for range queries
CREATE INDEX idx_user_created_at ON users(created_at);

-- Composite index for multi-column queries
CREATE INDEX idx_user_email_status ON users(email, status);

-- Partial index for filtered queries
CREATE INDEX idx_active_users ON users(id) WHERE status = 'active';
```

#### Query Optimization
- Use EXPLAIN plans
- Avoid N+1 queries
- Batch operations
- Pagination for large datasets

#### Connection Pooling
```python
# Connection pool configuration
pool = ConnectionPool(
    host='localhost',
    database='mydb',
    user='user',
    password='password',
    minconn=1,
    maxconn=20
)
```

### 3. Content Delivery Networks (CDN)

#### CDN Benefits
- Reduced latency through geographic distribution
- Decreased server load
- Improved availability
- DDoS protection

#### CDN Strategies
- **Static content**: Images, CSS, JS files
- **Dynamic content**: API responses with appropriate headers
- **Edge computing**: Processing at edge locations

### 4. Asynchronous Processing

#### Message Queues
```python
# Producer
def process_order(order_data):
    # Handle immediate operations
    order_id = create_order(order_data)
    
    # Queue background tasks
    queue.enqueue('send_confirmation_email', order_id)
    queue.enqueue('update_inventory', order_data['items'])
    queue.enqueue('process_payment', order_id)
    
    return order_id

# Consumer
def send_confirmation_email(order_id):
    order = get_order(order_id)
    email_service.send(order.user_email, generate_confirmation(order))
```

#### Event-Driven Architecture
- Publish-Subscribe pattern
- Loose coupling between services
- Scalable event processing

### 5. Compression

#### HTTP Compression
```
# Enable gzip compression
Content-Encoding: gzip
```

#### Database Compression
- Row-level compression
- Page-level compression
- Column store compression

## Performance Testing

### Load Testing Types

#### Load Testing
- Normal expected load
- Baseline performance measurement

#### Stress Testing
- Beyond normal capacity
- Find breaking point

#### Spike Testing
- Sudden load increases
- Traffic surge handling

#### Volume Testing
- Large amounts of data
- Database performance under load

### Performance Testing Tools
- **Apache JMeter**: GUI-based load testing
- **Apache Bench (ab)**: Simple command-line tool
- **Gatling**: High-performance testing framework
- **K6**: Modern load testing tool

## Optimization Techniques

### 1. Algorithm Optimization
```python
# Inefficient: O(n²)
def find_duplicates_slow(arr):
    duplicates = []
    for i in range(len(arr)):
        for j in range(i+1, len(arr)):
            if arr[i] == arr[j] and arr[i] not in duplicates:
                duplicates.append(arr[i])
    return duplicates

# Optimized: O(n)
def find_duplicates_fast(arr):
    seen = set()
    duplicates = set()
    for item in arr:
        if item in seen:
            duplicates.add(item)
        else:
            seen.add(item)
    return list(duplicates)
```

### 2. Memory Optimization
- Object pooling
- Memory-mapped files
- Garbage collection tuning

### 3. Network Optimization
- HTTP/2 multiplexing
- Connection keep-alive
- Request batching
- Payload compression

## Monitoring and Profiling

### Application Performance Monitoring (APM)
- **New Relic**: Full-stack monitoring
- **Datadog**: Infrastructure and application monitoring
- **AppDynamics**: End-to-end visibility

### Profiling Tools
- **CPU profiling**: Identify hot spots
- **Memory profiling**: Find memory leaks
- **I/O profiling**: Database and file system bottlenecks

### Key Performance Indicators (KPIs)
```
Response Time Targets:
- Web pages: < 2 seconds
- API calls: < 100ms
- Database queries: < 50ms
- Cache hits: < 1ms
```

## Performance Anti-Patterns

### 1. Premature Optimization
- Optimize based on measurements, not assumptions
- Profile first, then optimize

### 2. Over-Engineering
- Don't build for scale you don't need
- Start simple, evolve as needed

### 3. Ignoring the Database
- Database is often the bottleneck
- Optimize queries and schema design

## Real-World Case Studies

### Instagram's Performance Strategy
- **CDN**: Static content delivery
- **Memcached**: Aggressive caching
- **Database sharding**: User-based partitioning
- **Asynchronous processing**: Background jobs

### Twitter's Timeline Performance
- **Fan-out strategies**: Push vs Pull models
- **Caching**: Timeline caches per user
- **Load balancing**: Geographic distribution

## Exercise Problems

1. Design a caching strategy for a news website with 1M daily active users
2. How would you optimize a slow database query that joins 5 tables?
3. Calculate the theoretical maximum throughput for a system with 100ms latency and 1000 concurrent connections
4. Design a performance monitoring strategy for a microservices architecture

## Performance Checklist

### Frontend Optimization
- [ ] Minify CSS/JS
- [ ] Image optimization
- [ ] Browser caching
- [ ] CDN implementation

### Backend Optimization
- [ ] Database indexing
- [ ] Connection pooling
- [ ] Caching layer
- [ ] Asynchronous processing

### Infrastructure Optimization
- [ ] Load balancing
- [ ] Auto-scaling
- [ ] Geographic distribution
- [ ] Monitoring setup

## Key Takeaways

- Measure before optimizing
- Cache aggressively but intelligently
- Database optimization is crucial
- Asynchronous processing improves user experience
- Monitor performance continuously
- Consider the entire request path
- Trade-offs exist between performance and other qualities

## Next Steps

Move to: **04-consistency-models.md**