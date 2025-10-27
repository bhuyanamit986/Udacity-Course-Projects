# Load Balancing Fundamentals

## What is Load Balancing?

Load balancing is the process of distributing network or application traffic across multiple servers to ensure no single server becomes overwhelmed, improving responsiveness and availability.

## Types of Load Balancers

### Layer 4 (Transport Layer) Load Balancing
- Operates at TCP/UDP level
- Routes based on IP and port
- **Pros**: Fast, low latency, protocol agnostic
- **Cons**: Limited routing decisions, no application awareness

```
Client → Layer 4 LB → [Server1:80, Server2:80, Server3:80]
         (IP + Port)
```

### Layer 7 (Application Layer) Load Balancing
- Operates at HTTP level
- Routes based on content (URL, headers, cookies)
- **Pros**: Smart routing, SSL termination, caching
- **Cons**: Higher latency, more resource intensive

```
Client → Layer 7 LB → [API Server, Web Server, Mobile API]
         (HTTP content)
```

## Load Balancing Algorithms

### 1. Round Robin
```python
class RoundRobinBalancer:
    def __init__(self, servers):
        self.servers = servers
        self.current = 0
    
    def get_server(self):
        server = self.servers[self.current]
        self.current = (self.current + 1) % len(self.servers)
        return server
```

**Pros**: Simple, fair distribution
**Cons**: Ignores server capacity, doesn't handle failures

### 2. Weighted Round Robin
```python
class WeightedRoundRobinBalancer:
    def __init__(self, servers_with_weights):
        self.servers = []
        for server, weight in servers_with_weights:
            self.servers.extend([server] * weight)
        self.current = 0
    
    def get_server(self):
        server = self.servers[self.current]
        self.current = (self.current + 1) % len(self.servers)
        return server
```

**Example**: Server A (weight 3), Server B (weight 1)
Result: A, A, A, B, A, A, A, B, ...

### 3. Least Connections
```python
class LeastConnectionsBalancer:
    def __init__(self, servers):
        self.servers = servers
        self.connections = {server: 0 for server in servers}
    
    def get_server(self):
        return min(self.connections, key=self.connections.get)
    
    def add_connection(self, server):
        self.connections[server] += 1
    
    def remove_connection(self, server):
        self.connections[server] -= 1
```

### 4. Least Response Time
- Routes to server with lowest response time
- Considers both active connections and response time

### 5. IP Hash
```python
import hashlib

class IPHashBalancer:
    def __init__(self, servers):
        self.servers = servers
    
    def get_server(self, client_ip):
        hash_value = int(hashlib.md5(client_ip.encode()).hexdigest(), 16)
        return self.servers[hash_value % len(self.servers)]
```

**Pros**: Session affinity, consistent routing
**Cons**: Uneven distribution, difficult scaling

### 6. Random
```python
import random

class RandomBalancer:
    def __init__(self, servers):
        self.servers = servers
    
    def get_server(self):
        return random.choice(self.servers)
```

## Health Checks

### Active Health Checks
```python
class HealthChecker:
    def __init__(self, servers, check_interval=30):
        self.servers = servers
        self.healthy_servers = set(servers)
        self.check_interval = check_interval
    
    def check_health(self, server):
        try:
            response = requests.get(f"http://{server}/health", timeout=5)
            return response.status_code == 200
        except:
            return False
    
    def run_health_checks(self):
        while True:
            for server in self.servers:
                if self.check_health(server):
                    self.healthy_servers.add(server)
                else:
                    self.healthy_servers.discard(server)
            time.sleep(self.check_interval)
```

### Passive Health Checks
- Monitor actual request failures
- Remove servers after consecutive failures
- Re-add after successful responses

## Load Balancer Architectures

### 1. Single Load Balancer
```
Internet → Load Balancer → [Server1, Server2, Server3]
```
**Issues**: Single point of failure

### 2. Load Balancer Pair (Active-Passive)
```
Internet → [Primary LB, Backup LB] → [Server1, Server2, Server3]
```
**Benefits**: High availability

### 3. DNS Load Balancing
```
DNS Server → Multiple IP addresses for same domain
Client → Connects to one of the IPs
```

### 4. Global Load Balancing
```
User in US → US Load Balancer → US Servers
User in EU → EU Load Balancer → EU Servers
User in Asia → Asia Load Balancer → Asia Servers
```

## Session Affinity (Sticky Sessions)

### Problem
```
User Login → Server A (creates session)
Next Request → Server B (no session data) → Error
```

### Solutions

#### 1. Sticky Sessions
```python
class StickySessionBalancer:
    def __init__(self, servers):
        self.servers = servers
        self.session_map = {}
    
    def get_server(self, session_id):
        if session_id in self.session_map:
            return self.session_map[session_id]
        
        # New session - assign to server
        server = self.select_server()  # Use any algorithm
        self.session_map[session_id] = server
        return server
```

#### 2. Shared Session Store
```python
# All servers use shared session storage
class SessionManager:
    def __init__(self):
        self.redis_client = redis.Redis()
    
    def get_session(self, session_id):
        return self.redis_client.get(f"session:{session_id}")
    
    def set_session(self, session_id, data, ttl=3600):
        self.redis_client.setex(f"session:{session_id}", ttl, data)
```

#### 3. Stateless Design (Preferred)
```python
# Use JWT tokens instead of server sessions
class StatelessAuth:
    def create_token(self, user_data):
        return jwt.encode(user_data, secret_key, algorithm='HS256')
    
    def verify_token(self, token):
        return jwt.decode(token, secret_key, algorithms=['HS256'])
```

## Advanced Load Balancing

### 1. Consistent Hashing
```python
import hashlib

class ConsistentHashBalancer:
    def __init__(self, servers, virtual_nodes=150):
        self.servers = servers
        self.virtual_nodes = virtual_nodes
        self.ring = {}
        self.sorted_keys = []
        self._build_ring()
    
    def _hash(self, key):
        return int(hashlib.md5(key.encode()).hexdigest(), 16)
    
    def _build_ring(self):
        for server in self.servers:
            for i in range(self.virtual_nodes):
                virtual_key = f"{server}:{i}"
                hash_value = self._hash(virtual_key)
                self.ring[hash_value] = server
        
        self.sorted_keys = sorted(self.ring.keys())
    
    def get_server(self, key):
        hash_value = self._hash(key)
        
        # Find the first server clockwise
        for ring_key in self.sorted_keys:
            if hash_value <= ring_key:
                return self.ring[ring_key]
        
        # Wrap around to first server
        return self.ring[self.sorted_keys[0]]
    
    def add_server(self, server):
        self.servers.append(server)
        self._build_ring()
    
    def remove_server(self, server):
        self.servers.remove(server)
        self._build_ring()
```

### 2. Geographic Load Balancing
```python
class GeographicBalancer:
    def __init__(self):
        self.regions = {
            'us-east': ['server1', 'server2'],
            'us-west': ['server3', 'server4'],
            'eu': ['server5', 'server6'],
            'asia': ['server7', 'server8']
        }
    
    def get_server(self, client_ip):
        region = self.get_region(client_ip)
        return self.select_from_region(region)
    
    def get_region(self, client_ip):
        # Use GeoIP lookup
        return geoip.lookup(client_ip).region
```

### 3. Content-Based Routing
```python
class ContentBasedBalancer:
    def __init__(self):
        self.api_servers = ['api1', 'api2']
        self.web_servers = ['web1', 'web2']
        self.mobile_servers = ['mobile1', 'mobile2']
    
    def route_request(self, request):
        if request.path.startswith('/api/'):
            return random.choice(self.api_servers)
        elif 'mobile' in request.headers.get('User-Agent', ''):
            return random.choice(self.mobile_servers)
        else:
            return random.choice(self.web_servers)
```

## Load Balancer Technologies

### Hardware Load Balancers
- **F5 BIG-IP**: Enterprise-grade
- **Citrix NetScaler**: Application delivery
- **A10 Networks**: High performance

### Software Load Balancers
- **HAProxy**: Open source, high performance
- **NGINX**: Web server + load balancer
- **Envoy**: Modern proxy, service mesh

### Cloud Load Balancers
- **AWS ELB**: Application/Network/Classic
- **Google Cloud Load Balancing**: Global/Regional
- **Azure Load Balancer**: Basic/Standard

## Configuration Examples

### HAProxy Configuration
```
global
    daemon
    maxconn 4096

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend web_frontend
    bind *:80
    default_backend web_servers

backend web_servers
    balance roundrobin
    option httpchk GET /health
    server web1 192.168.1.10:8080 check
    server web2 192.168.1.11:8080 check
    server web3 192.168.1.12:8080 check
```

### NGINX Configuration
```nginx
upstream backend {
    least_conn;
    server 192.168.1.10:8080 weight=3;
    server 192.168.1.11:8080 weight=2;
    server 192.168.1.12:8080 weight=1;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## Monitoring Load Balancers

### Key Metrics
- **Request rate**: Requests per second
- **Error rate**: 4xx/5xx responses
- **Response time**: Latency percentiles
- **Connection count**: Active connections
- **Server health**: Healthy vs unhealthy servers

### Alerting
```python
class LoadBalancerMonitor:
    def __init__(self):
        self.metrics = {}
    
    def check_health(self):
        healthy_servers = self.count_healthy_servers()
        total_servers = len(self.servers)
        
        if healthy_servers / total_servers < 0.5:
            self.alert("CRITICAL: Less than 50% servers healthy")
        
        if self.get_error_rate() > 0.05:
            self.alert("WARNING: Error rate above 5%")
```

## Exercise Problems

1. Design a load balancing strategy for a video streaming service
2. How would you handle server failures in a load-balanced system?
3. Compare the trade-offs between different load balancing algorithms
4. Design a global load balancing solution for a social media platform

## Key Takeaways

- Load balancing is essential for scalability and availability
- Choose algorithm based on application characteristics
- Health checks are crucial for reliability
- Consider session affinity requirements
- Monitor load balancer performance
- Plan for load balancer failures
- Different layers serve different purposes
- Geographic distribution improves user experience

## Next Steps

Complete Module 1 exercises and move to: **Module 2: System Components**