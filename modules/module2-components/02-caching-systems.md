# Caching Systems and Strategies

## What is Caching?

Caching is the practice of storing frequently accessed data in fast storage to reduce latency and improve system performance.

## Cache Hierarchy

```
Browser Cache (100ms)
    ↓
CDN Cache (50ms)
    ↓
Load Balancer Cache (10ms)
    ↓
Application Cache (1ms)
    ↓
Database Cache (5ms)
    ↓
Database (100ms)
```

## Cache Patterns

### 1. Cache-Aside (Lazy Loading)
```python
class CacheAside:
    def __init__(self, cache, database):
        self.cache = cache
        self.database = database
    
    def get(self, key):
        # Try cache first
        value = self.cache.get(key)
        if value is not None:
            return value
        
        # Cache miss - fetch from database
        value = self.database.get(key)
        if value is not None:
            # Store in cache for future requests
            self.cache.set(key, value, ttl=300)
        
        return value
    
    def set(self, key, value):
        # Update database
        self.database.set(key, value)
        # Invalidate cache
        self.cache.delete(key)
```

**Pros**: Cache only what's needed, handles cache failures gracefully
**Cons**: Cache miss penalty, potential for stale data

### 2. Write-Through Cache
```python
class WriteThroughCache:
    def __init__(self, cache, database):
        self.cache = cache
        self.database = database
    
    def set(self, key, value):
        # Write to database first
        self.database.set(key, value)
        # Then update cache
        self.cache.set(key, value)
    
    def get(self, key):
        # Always read from cache
        return self.cache.get(key)
```

**Pros**: Cache is always consistent, no cache miss penalty for reads
**Cons**: Higher write latency, unnecessary cache writes

### 3. Write-Behind (Write-Back) Cache
```python
class WriteBehindCache:
    def __init__(self, cache, database):
        self.cache = cache
        self.database = database
        self.write_queue = queue.Queue()
        self.start_background_writer()
    
    def set(self, key, value):
        # Write to cache immediately
        self.cache.set(key, value)
        # Queue for database write
        self.write_queue.put((key, value))
    
    def get(self, key):
        return self.cache.get(key)
    
    def background_writer(self):
        while True:
            key, value = self.write_queue.get()
            self.database.set(key, value)
```

**Pros**: Low write latency, batch database writes
**Cons**: Risk of data loss, complex error handling

### 4. Write-Around Cache
```python
class WriteAroundCache:
    def __init__(self, cache, database):
        self.cache = cache
        self.database = database
    
    def set(self, key, value):
        # Write only to database
        self.database.set(key, value)
        # Don't update cache
    
    def get(self, key):
        # Try cache first
        value = self.cache.get(key)
        if value is not None:
            return value
        
        # Cache miss - fetch from database
        value = self.database.get(key)
        if value is not None:
            self.cache.set(key, value, ttl=300)
        
        return value
```

**Use case**: Write-heavy workloads where data is not immediately read

## Cache Eviction Policies

### 1. Least Recently Used (LRU)
```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = OrderedDict()
    
    def get(self, key):
        if key in self.cache:
            # Move to end (most recently used)
            self.cache.move_to_end(key)
            return self.cache[key]
        return None
    
    def set(self, key, value):
        if key in self.cache:
            # Update existing key
            self.cache.move_to_end(key)
        elif len(self.cache) >= self.capacity:
            # Remove least recently used
            self.cache.popitem(last=False)
        
        self.cache[key] = value
```

### 2. Least Frequently Used (LFU)
```python
class LFUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}
        self.frequencies = {}
        self.min_frequency = 0
        self.freq_to_keys = defaultdict(OrderedDict)
    
    def get(self, key):
        if key not in self.cache:
            return None
        
        # Increment frequency
        self._increment_frequency(key)
        return self.cache[key]
    
    def set(self, key, value):
        if self.capacity <= 0:
            return
        
        if key in self.cache:
            self.cache[key] = value
            self._increment_frequency(key)
        else:
            if len(self.cache) >= self.capacity:
                self._evict_lfu()
            
            self.cache[key] = value
            self.frequencies[key] = 1
            self.freq_to_keys[1][key] = True
            self.min_frequency = 1
```

### 3. Time-To-Live (TTL)
```python
import time

class TTLCache:
    def __init__(self):
        self.cache = {}
        self.expiry_times = {}
    
    def set(self, key, value, ttl):
        self.cache[key] = value
        self.expiry_times[key] = time.time() + ttl
    
    def get(self, key):
        if key not in self.cache:
            return None
        
        if time.time() > self.expiry_times[key]:
            # Expired
            del self.cache[key]
            del self.expiry_times[key]
            return None
        
        return self.cache[key]
```

## Distributed Caching

### 1. Redis Cluster
```python
import redis

class RedisCluster:
    def __init__(self, nodes):
        self.cluster = redis.RedisCluster(
            startup_nodes=nodes,
            decode_responses=True,
            skip_full_coverage_check=True
        )
    
    def get(self, key):
        return self.cluster.get(key)
    
    def set(self, key, value, ttl=3600):
        return self.cluster.setex(key, ttl, value)
    
    def delete(self, key):
        return self.cluster.delete(key)
```

### 2. Memcached
```python
import memcache

class MemcachedCluster:
    def __init__(self, servers):
        self.client = memcache.Client(servers)
    
    def get(self, key):
        return self.client.get(key)
    
    def set(self, key, value, ttl=3600):
        return self.client.set(key, value, time=ttl)
    
    def get_multi(self, keys):
        return self.client.get_multi(keys)
```

## Cache Strategies by Use Case

### 1. Database Query Caching
```python
class QueryCache:
    def __init__(self, cache, database):
        self.cache = cache
        self.database = database
    
    def execute_query(self, sql, params):
        # Create cache key from query and params
        cache_key = hashlib.md5(f"{sql}:{params}".encode()).hexdigest()
        
        # Check cache
        result = self.cache.get(cache_key)
        if result is not None:
            return result
        
        # Execute query
        result = self.database.execute(sql, params)
        
        # Cache result
        self.cache.set(cache_key, result, ttl=300)
        return result
```

### 2. API Response Caching
```python
from functools import wraps

def cache_api_response(ttl=300):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Create cache key from function name and arguments
            cache_key = f"{func.__name__}:{hash(str(args) + str(kwargs))}"
            
            # Check cache
            cached_result = cache.get(cache_key)
            if cached_result is not None:
                return cached_result
            
            # Execute function
            result = func(*args, **kwargs)
            
            # Cache result
            cache.set(cache_key, result, ttl=ttl)
            return result
        
        return wrapper
    return decorator

@cache_api_response(ttl=600)
def get_user_profile(user_id):
    return database.get_user(user_id)
```

### 3. Session Caching
```python
class SessionCache:
    def __init__(self, redis_client):
        self.redis = redis_client
    
    def create_session(self, user_id, session_data):
        session_id = str(uuid.uuid4())
        session_key = f"session:{session_id}"
        
        self.redis.setex(
            session_key, 
            3600,  # 1 hour TTL
            json.dumps({
                'user_id': user_id,
                'created_at': time.time(),
                **session_data
            })
        )
        return session_id
    
    def get_session(self, session_id):
        session_key = f"session:{session_id}"
        session_data = self.redis.get(session_key)
        
        if session_data:
            # Extend TTL on access
            self.redis.expire(session_key, 3600)
            return json.loads(session_data)
        
        return None
```

## Cache Invalidation

### 1. Time-Based Invalidation (TTL)
```python
# Set with TTL
cache.setex("user:1001", 3600, user_data)  # Expires in 1 hour
```

### 2. Event-Based Invalidation
```python
class EventBasedInvalidation:
    def __init__(self, cache, event_bus):
        self.cache = cache
        self.event_bus = event_bus
        self.event_bus.subscribe("user_updated", self.invalidate_user_cache)
    
    def invalidate_user_cache(self, event):
        user_id = event['user_id']
        cache_keys = [
            f"user:{user_id}",
            f"user_profile:{user_id}",
            f"user_settings:{user_id}"
        ]
        
        for key in cache_keys:
            self.cache.delete(key)
```

### 3. Tag-Based Invalidation
```python
class TaggedCache:
    def __init__(self, cache):
        self.cache = cache
        self.tag_mapping = defaultdict(set)
    
    def set(self, key, value, ttl=3600, tags=None):
        self.cache.set(key, value, ttl)
        
        if tags:
            for tag in tags:
                self.tag_mapping[tag].add(key)
    
    def invalidate_by_tag(self, tag):
        keys = self.tag_mapping[tag]
        for key in keys:
            self.cache.delete(key)
        del self.tag_mapping[tag]

# Usage
cache.set("user:1001", user_data, tags=["user", "profile"])
cache.set("posts:user:1001", posts, tags=["user", "posts"])

# Invalidate all user-related caches
cache.invalidate_by_tag("user")
```

## Caching Anti-Patterns

### 1. Cache Stampede
```python
# Problem: Multiple requests fetch same data simultaneously
def get_expensive_data(key):
    value = cache.get(key)
    if value is None:
        # Multiple requests hit this simultaneously
        value = expensive_database_operation(key)
        cache.set(key, value)
    return value

# Solution: Lock-based approach
import threading

class StampedeProtection:
    def __init__(self, cache, database):
        self.cache = cache
        self.database = database
        self.locks = {}
        self.lock_mutex = threading.Lock()
    
    def get(self, key):
        value = self.cache.get(key)
        if value is not None:
            return value
        
        # Get or create lock for this key
        with self.lock_mutex:
            if key not in self.locks:
                self.locks[key] = threading.Lock()
            lock = self.locks[key]
        
        with lock:
            # Double-check cache after acquiring lock
            value = self.cache.get(key)
            if value is not None:
                return value
            
            # Only one thread executes this
            value = self.database.get(key)
            self.cache.set(key, value, ttl=300)
            return value
```

### 2. Hot Key Problem
```python
class HotKeyMitigation:
    def __init__(self, cache, num_replicas=3):
        self.cache = cache
        self.num_replicas = num_replicas
    
    def get(self, key):
        # Try multiple cache keys for hot data
        for i in range(self.num_replicas):
            replica_key = f"{key}:replica:{i}"
            value = self.cache.get(replica_key)
            if value is not None:
                return value
        
        # Fallback to original key
        return self.cache.get(key)
    
    def set(self, key, value, ttl=300):
        # Replicate hot data across multiple cache keys
        for i in range(self.num_replicas):
            replica_key = f"{key}:replica:{i}"
            self.cache.set(replica_key, value, ttl)
```

## Cache Technologies

### In-Memory Caches

#### Redis
```python
import redis

class RedisCache:
    def __init__(self, host='localhost', port=6379):
        self.client = redis.Redis(host=host, port=port, decode_responses=True)
    
    def get(self, key):
        return self.client.get(key)
    
    def set(self, key, value, ttl=3600):
        return self.client.setex(key, ttl, value)
    
    def get_hash(self, key, field):
        return self.client.hget(key, field)
    
    def set_hash(self, key, field, value):
        return self.client.hset(key, field, value)
    
    def increment(self, key, amount=1):
        return self.client.incr(key, amount)
```

#### Memcached
```python
import memcache

class MemcachedCache:
    def __init__(self, servers):
        self.client = memcache.Client(servers)
    
    def get(self, key):
        return self.client.get(key)
    
    def set(self, key, value, ttl=3600):
        return self.client.set(key, value, time=ttl)
    
    def delete(self, key):
        return self.client.delete(key)
```

### Application-Level Caches

#### Local Cache (In-Process)
```python
import threading
from functools import lru_cache

class LocalCache:
    def __init__(self, max_size=1000):
        self.cache = {}
        self.access_times = {}
        self.max_size = max_size
        self.lock = threading.RLock()
    
    def get(self, key):
        with self.lock:
            if key in self.cache:
                self.access_times[key] = time.time()
                return self.cache[key]
            return None
    
    def set(self, key, value):
        with self.lock:
            if len(self.cache) >= self.max_size:
                self._evict_lru()
            
            self.cache[key] = value
            self.access_times[key] = time.time()
    
    def _evict_lru(self):
        lru_key = min(self.access_times, key=self.access_times.get)
        del self.cache[lru_key]
        del self.access_times[lru_key]

# Using Python's built-in LRU cache
@lru_cache(maxsize=1000)
def expensive_computation(param):
    return complex_calculation(param)
```

## Multi-Level Caching

### L1 (Local) + L2 (Distributed) Cache
```python
class MultiLevelCache:
    def __init__(self, local_cache, distributed_cache, database):
        self.l1_cache = local_cache  # Fast, small
        self.l2_cache = distributed_cache  # Slower, larger
        self.database = database
    
    def get(self, key):
        # Try L1 cache first
        value = self.l1_cache.get(key)
        if value is not None:
            return value
        
        # Try L2 cache
        value = self.l2_cache.get(key)
        if value is not None:
            # Promote to L1
            self.l1_cache.set(key, value, ttl=300)
            return value
        
        # Fetch from database
        value = self.database.get(key)
        if value is not None:
            # Store in both caches
            self.l1_cache.set(key, value, ttl=300)
            self.l2_cache.set(key, value, ttl=3600)
        
        return value
    
    def set(self, key, value):
        # Update database
        self.database.set(key, value)
        
        # Invalidate caches
        self.l1_cache.delete(key)
        self.l2_cache.delete(key)
```

## Cache Warming

### Proactive Cache Loading
```python
class CacheWarmer:
    def __init__(self, cache, database):
        self.cache = cache
        self.database = database
    
    def warm_popular_data(self):
        """Preload popular data into cache"""
        popular_keys = self.database.get_popular_keys()
        
        for key in popular_keys:
            value = self.database.get(key)
            self.cache.set(key, value, ttl=3600)
    
    def warm_user_data(self, user_id):
        """Preload user-specific data"""
        user_data = self.database.get_user(user_id)
        self.cache.set(f"user:{user_id}", user_data, ttl=1800)
        
        # Preload related data
        user_posts = self.database.get_user_posts(user_id)
        self.cache.set(f"posts:{user_id}", user_posts, ttl=900)
```

## Cache Monitoring

### Key Metrics
```python
class CacheMetrics:
    def __init__(self):
        self.hits = 0
        self.misses = 0
        self.total_requests = 0
        self.response_times = []
    
    def record_hit(self, response_time):
        self.hits += 1
        self.total_requests += 1
        self.response_times.append(response_time)
    
    def record_miss(self, response_time):
        self.misses += 1
        self.total_requests += 1
        self.response_times.append(response_time)
    
    def get_hit_ratio(self):
        if self.total_requests == 0:
            return 0
        return self.hits / self.total_requests
    
    def get_average_response_time(self):
        if not self.response_times:
            return 0
        return sum(self.response_times) / len(self.response_times)
```

### Alerting Thresholds
```python
def check_cache_health(metrics):
    hit_ratio = metrics.get_hit_ratio()
    avg_response_time = metrics.get_average_response_time()
    
    if hit_ratio < 0.8:
        alert("LOW_CACHE_HIT_RATIO", f"Hit ratio: {hit_ratio:.2%}")
    
    if avg_response_time > 100:  # ms
        alert("HIGH_CACHE_LATENCY", f"Avg latency: {avg_response_time}ms")
```

## Caching Best Practices

### 1. Cache Key Design
```python
class CacheKeyGenerator:
    @staticmethod
    def user_profile(user_id, version=None):
        if version:
            return f"user:profile:{user_id}:v{version}"
        return f"user:profile:{user_id}"
    
    @staticmethod
    def user_posts(user_id, page=1, limit=10):
        return f"user:posts:{user_id}:p{page}:l{limit}"
    
    @staticmethod
    def search_results(query, filters=None):
        filter_hash = hashlib.md5(str(sorted(filters.items())).encode()).hexdigest()
        return f"search:{hashlib.md5(query.encode()).hexdigest()}:{filter_hash}"
```

### 2. Cache Size Management
```python
class CacheSizeManager:
    def __init__(self, cache, max_memory_mb=1000):
        self.cache = cache
        self.max_memory_bytes = max_memory_mb * 1024 * 1024
    
    def check_memory_usage(self):
        current_usage = self.cache.memory_usage()
        
        if current_usage > self.max_memory_bytes * 0.9:
            # Proactively evict to prevent memory pressure
            self.evict_least_important_data()
    
    def evict_least_important_data(self):
        # Remove cache entries by importance/access pattern
        low_priority_keys = self.cache.get_keys_by_pattern("temp:*")
        for key in low_priority_keys:
            self.cache.delete(key)
```

### 3. Cache Consistency
```python
class ConsistentCache:
    def __init__(self, cache, database):
        self.cache = cache
        self.database = database
        self.version_counter = 0
    
    def update_data(self, key, new_value):
        # Increment version
        self.version_counter += 1
        
        # Update database with version
        self.database.set(key, {
            'data': new_value,
            'version': self.version_counter
        })
        
        # Update cache with version
        self.cache.set(key, {
            'data': new_value,
            'version': self.version_counter
        }, ttl=300)
    
    def get_data(self, key):
        cached_item = self.cache.get(key)
        if cached_item:
            # Verify version with database periodically
            if random.random() < 0.1:  # 10% of requests
                db_item = self.database.get(key)
                if db_item['version'] > cached_item['version']:
                    # Cache is stale, update it
                    self.cache.set(key, db_item, ttl=300)
                    return db_item['data']
            
            return cached_item['data']
        
        # Cache miss
        db_item = self.database.get(key)
        if db_item:
            self.cache.set(key, db_item, ttl=300)
            return db_item['data']
        
        return None
```

## Exercise Problems

1. Design a caching strategy for a news website with 1M daily users
2. How would you handle cache invalidation for a social media platform?
3. Design a multi-level caching system for an e-commerce product catalog
4. Implement a cache warming strategy for a recommendation system

## Key Takeaways

- Caching is one of the most effective performance optimizations
- Choose cache pattern based on read/write characteristics
- Cache invalidation is one of the hardest problems in computer science
- Monitor cache hit ratios and adjust strategies accordingly
- Consider cache consistency requirements
- Plan for cache failures and degraded performance
- Multi-level caching can provide best of both worlds
- Cache key design is crucial for maintainability

## Next Steps

Move to: **03-message-queues.md**