# Microservices Architecture

## What are Microservices?

Microservices is an architectural approach where applications are built as a suite of small, independent services that communicate over well-defined APIs.

## Monolith vs Microservices

### Monolithic Architecture
```
Single Deployable Unit:
[UI + Business Logic + Data Access + Database]
```

**Pros**: Simple deployment, easy testing, good performance
**Cons**: Technology lock-in, scaling challenges, single point of failure

### Microservices Architecture
```
Service A ←→ API Gateway ←→ Service B
    ↓                           ↓
Database A                 Database B
```

**Pros**: Technology diversity, independent scaling, fault isolation
**Cons**: Complexity, network latency, data consistency challenges

## Microservices Design Principles

### 1. Single Responsibility
```python
# Good: Each service has one responsibility
class UserService:
    def create_user(self, user_data): pass
    def get_user(self, user_id): pass
    def update_user(self, user_id, data): pass
    def delete_user(self, user_id): pass

class OrderService:
    def create_order(self, order_data): pass
    def get_order(self, order_id): pass
    def update_order_status(self, order_id, status): pass

class PaymentService:
    def process_payment(self, payment_data): pass
    def refund_payment(self, payment_id): pass
```

### 2. Decentralized Data Management
```python
# Each service owns its data
class EcommerceSystem:
    def __init__(self):
        self.user_service = UserService(user_database)
        self.product_service = ProductService(product_database)
        self.order_service = OrderService(order_database)
        self.payment_service = PaymentService(payment_database)
```

### 3. Design for Failure
```python
class ResilientMicroservice:
    def __init__(self, external_service):
        self.external_service = external_service
        self.circuit_breaker = CircuitBreaker()
        self.cache = Cache()
    
    def call_external_service(self, request):
        try:
            # Try circuit breaker protected call
            return self.circuit_breaker.call(
                self.external_service.make_request, 
                request
            )
        except Exception as e:
            # Fallback to cached data
            cached_response = self.cache.get(f"fallback:{request.id}")
            if cached_response:
                return cached_response
            
            # Final fallback
            return self.default_response(request)
```

## Service Communication Patterns

### 1. Synchronous Communication (HTTP/REST)
```python
import requests

class HTTPServiceClient:
    def __init__(self, base_url, timeout=30):
        self.base_url = base_url
        self.timeout = timeout
        self.session = requests.Session()
    
    def get_user(self, user_id):
        try:
            response = self.session.get(
                f"{self.base_url}/users/{user_id}",
                timeout=self.timeout
            )
            response.raise_for_status()
            return response.json()
        except requests.exceptions.Timeout:
            raise ServiceTimeoutError("User service timeout")
        except requests.exceptions.RequestException as e:
            raise ServiceError(f"User service error: {e}")
    
    def create_order(self, order_data):
        response = self.session.post(
            f"{self.base_url}/orders",
            json=order_data,
            timeout=self.timeout
        )
        response.raise_for_status()
        return response.json()
```

### 2. Asynchronous Communication (Message Queues)
```python
class AsyncServiceCommunication:
    def __init__(self, message_broker):
        self.broker = message_broker
    
    def publish_event(self, event_type, data):
        """Publish domain event"""
        event = {
            'event_type': event_type,
            'data': data,
            'timestamp': time.time(),
            'service': self.service_name
        }
        
        self.broker.publish(f'events.{event_type}', event)
    
    def subscribe_to_events(self, event_types, handler):
        """Subscribe to events from other services"""
        for event_type in event_types:
            self.broker.subscribe(f'events.{event_type}', handler)

# Example usage
class OrderService:
    def create_order(self, order_data):
        order = self.save_order(order_data)
        
        # Publish event for other services
        self.event_bus.publish_event('order_created', {
            'order_id': order.id,
            'user_id': order.user_id,
            'total_amount': order.total
        })
        
        return order

class InventoryService:
    def handle_order_created(self, event):
        """React to order creation"""
        order_id = event['data']['order_id']
        # Update inventory
        self.update_inventory_for_order(order_id)
```

### 3. Service Mesh
```python
class ServiceMeshProxy:
    def __init__(self, service_name):
        self.service_name = service_name
        self.metrics = ServiceMetrics()
        self.circuit_breaker = CircuitBreaker()
    
    def make_request(self, target_service, endpoint, data=None):
        """Proxy request through service mesh"""
        
        start_time = time.time()
        
        try:
            # Service discovery
            target_url = self.discover_service(target_service)
            
            # Load balancing
            server = self.select_server(target_url)
            
            # Make request with retries
            response = self.circuit_breaker.call(
                self.http_request,
                f"{server}{endpoint}",
                data
            )
            
            # Record metrics
            self.metrics.record_success(
                target_service, 
                time.time() - start_time
            )
            
            return response
            
        except Exception as e:
            self.metrics.record_failure(target_service, str(e))
            raise e
```

## Service Discovery

### 1. Client-Side Discovery
```python
class ClientSideDiscovery:
    def __init__(self, registry_client):
        self.registry = registry_client
        self.service_cache = {}
        self.cache_ttl = 60  # 1 minute
    
    def discover_service(self, service_name):
        """Discover service instances"""
        
        # Check cache first
        cached_entry = self.service_cache.get(service_name)
        if cached_entry and time.time() - cached_entry['timestamp'] < self.cache_ttl:
            return cached_entry['instances']
        
        # Fetch from registry
        instances = self.registry.get_service_instances(service_name)
        
        # Update cache
        self.service_cache[service_name] = {
            'instances': instances,
            'timestamp': time.time()
        }
        
        return instances
    
    def select_instance(self, instances):
        """Select instance using load balancing"""
        healthy_instances = [
            instance for instance in instances
            if self.health_check(instance)
        ]
        
        if not healthy_instances:
            raise ServiceUnavailableError(f"No healthy instances available")
        
        # Simple round-robin
        return random.choice(healthy_instances)
```

### 2. Server-Side Discovery (Service Mesh)
```python
class ServerSideDiscovery:
    def __init__(self):
        self.service_registry = {}
        self.health_checks = {}
    
    def register_service(self, service_name, instance_info):
        """Register service instance"""
        if service_name not in self.service_registry:
            self.service_registry[service_name] = []
        
        instance_info['registered_at'] = time.time()
        instance_info['last_heartbeat'] = time.time()
        
        self.service_registry[service_name].append(instance_info)
    
    def heartbeat(self, service_name, instance_id):
        """Update instance heartbeat"""
        instances = self.service_registry.get(service_name, [])
        
        for instance in instances:
            if instance['id'] == instance_id:
                instance['last_heartbeat'] = time.time()
                break
    
    def get_healthy_instances(self, service_name):
        """Get healthy instances for service"""
        instances = self.service_registry.get(service_name, [])
        current_time = time.time()
        
        healthy_instances = [
            instance for instance in instances
            if current_time - instance['last_heartbeat'] < 30  # 30 second timeout
        ]
        
        return healthy_instances
```

## API Gateway

### Gateway Responsibilities
```python
class APIGateway:
    def __init__(self):
        self.rate_limiter = RateLimiter()
        self.auth_service = AuthenticationService()
        self.service_discovery = ServiceDiscovery()
        self.metrics = GatewayMetrics()
    
    def handle_request(self, request):
        """Central request handling"""
        
        start_time = time.time()
        
        try:
            # 1. Authentication
            user = self.auth_service.authenticate(request.headers.get('Authorization'))
            
            # 2. Rate limiting
            if not self.rate_limiter.allow_request(user.id):
                return self.error_response(429, "Rate limit exceeded")
            
            # 3. Request routing
            target_service = self.determine_target_service(request.path)
            service_instances = self.service_discovery.get_instances(target_service)
            
            # 4. Load balancing
            selected_instance = self.load_balance(service_instances)
            
            # 5. Request transformation
            transformed_request = self.transform_request(request, user)
            
            # 6. Forward request
            response = self.forward_request(selected_instance, transformed_request)
            
            # 7. Response transformation
            final_response = self.transform_response(response, user)
            
            # 8. Metrics collection
            self.metrics.record_request(
                target_service,
                time.time() - start_time,
                response.status_code
            )
            
            return final_response
            
        except Exception as e:
            self.metrics.record_error(str(e))
            return self.error_response(500, "Internal server error")
```

### Gateway Patterns

#### Backend for Frontend (BFF)
```python
class MobileAPIGateway:
    """Specialized gateway for mobile clients"""
    
    def get_dashboard_data(self, user_id):
        """Aggregate data from multiple services"""
        
        # Fetch data from multiple services in parallel
        with ThreadPoolExecutor(max_workers=3) as executor:
            user_future = executor.submit(self.user_service.get_user, user_id)
            orders_future = executor.submit(self.order_service.get_recent_orders, user_id)
            recommendations_future = executor.submit(self.recommendation_service.get_recommendations, user_id)
        
        # Wait for all responses
        user_data = user_future.result()
        recent_orders = orders_future.result()
        recommendations = recommendations_future.result()
        
        # Compose mobile-optimized response
        return {
            'user': {
                'name': user_data['name'],
                'avatar_url': user_data['avatar_url']
            },
            'recent_orders': recent_orders[:5],  # Limit for mobile
            'recommendations': recommendations[:10]
        }

class WebAPIGateway:
    """Specialized gateway for web clients"""
    
    def get_dashboard_data(self, user_id):
        """More detailed data for web interface"""
        
        # Fetch more comprehensive data
        user_data = self.user_service.get_full_profile(user_id)
        order_history = self.order_service.get_order_history(user_id)
        detailed_recommendations = self.recommendation_service.get_detailed_recommendations(user_id)
        
        return {
            'user': user_data,
            'order_history': order_history,
            'recommendations': detailed_recommendations,
            'analytics': self.analytics_service.get_user_analytics(user_id)
        }
```

## Data Management in Microservices

### 1. Database per Service
```python
class ServiceDataIsolation:
    """Each service has its own database"""
    
    def __init__(self):
        # Each service has dedicated database
        self.user_service = UserService(PostgreSQL("user_db"))
        self.product_service = ProductService(MongoDB("product_db"))
        self.order_service = OrderService(PostgreSQL("order_db"))
        self.analytics_service = AnalyticsService(ClickHouse("analytics_db"))
```

### 2. Saga Pattern for Distributed Transactions
```python
class OrderSaga:
    """Manage distributed transaction across services"""
    
    def __init__(self, user_service, inventory_service, payment_service):
        self.user_service = user_service
        self.inventory_service = inventory_service
        self.payment_service = payment_service
    
    def process_order(self, order_data):
        """Choreography-based saga"""
        
        saga_id = str(uuid.uuid4())
        
        try:
            # Step 1: Validate user
            user = self.user_service.validate_user(order_data['user_id'])
            
            # Step 2: Reserve inventory
            reservation = self.inventory_service.reserve_items(
                order_data['items'], saga_id
            )
            
            # Step 3: Process payment
            payment = self.payment_service.charge(
                user['payment_method'],
                order_data['total_amount'],
                saga_id
            )
            
            # Step 4: Confirm order
            order = self.create_order(order_data, saga_id)
            
            return order
            
        except Exception as e:
            # Compensating actions
            self.compensate_saga(saga_id, e)
            raise e
    
    def compensate_saga(self, saga_id, error):
        """Execute compensating transactions"""
        
        # Release inventory reservation
        try:
            self.inventory_service.release_reservation(saga_id)
        except Exception as e:
            log.error(f"Failed to release inventory: {e}")
        
        # Refund payment if charged
        try:
            self.payment_service.refund_by_saga(saga_id)
        except Exception as e:
            log.error(f"Failed to refund payment: {e}")
```

### 3. Event Sourcing for Data Synchronization
```python
class EventDrivenDataSync:
    def __init__(self, event_bus):
        self.event_bus = event_bus
        
        # Subscribe to relevant events
        self.event_bus.subscribe('user.created', self.handle_user_created)
        self.event_bus.subscribe('user.updated', self.handle_user_updated)
    
    def handle_user_created(self, event):
        """Handle user creation event"""
        user_data = event['data']
        
        # Create read model for this service
        self.create_user_read_model(user_data)
        
        # Publish derived event if needed
        self.event_bus.publish('user.profile.created', {
            'user_id': user_data['id'],
            'profile_id': self.generate_profile_id()
        })
```

## Microservices Patterns

### 1. API Composition
```python
class APICompositionService:
    """Compose responses from multiple services"""
    
    def __init__(self, user_service, order_service, recommendation_service):
        self.user_service = user_service
        self.order_service = order_service
        self.recommendation_service = recommendation_service
    
    def get_user_dashboard(self, user_id):
        """Compose dashboard data from multiple services"""
        
        # Make parallel calls to services
        with ThreadPoolExecutor() as executor:
            user_future = executor.submit(
                self.user_service.get_user, user_id
            )
            orders_future = executor.submit(
                self.order_service.get_user_orders, user_id
            )
            recommendations_future = executor.submit(
                self.recommendation_service.get_recommendations, user_id
            )
        
        # Combine results
        return {
            'user': user_future.result(),
            'recent_orders': orders_future.result()[:5],
            'recommendations': recommendations_future.result()
        }
```

### 2. Command Query Responsibility Segregation (CQRS)
```python
class UserCommandService:
    """Handle user write operations"""
    
    def __init__(self, event_store):
        self.event_store = event_store
    
    def create_user(self, user_data):
        # Validate command
        self.validate_user_data(user_data)
        
        # Generate events
        events = [
            {'type': 'user_created', 'data': user_data},
            {'type': 'profile_initialized', 'data': {'user_id': user_data['id']}}
        ]
        
        # Store events
        for event in events:
            self.event_store.append(user_data['id'], event)
        
        return user_data['id']

class UserQueryService:
    """Handle user read operations"""
    
    def __init__(self, read_database):
        self.read_db = read_database
    
    def get_user_profile(self, user_id):
        """Get optimized read model"""
        return self.read_db.get_user_profile(user_id)
    
    def search_users(self, query):
        """Optimized search functionality"""
        return self.read_db.search_users(query)
```

### 3. Bulkhead Pattern
```python
class BulkheadIsolation:
    """Isolate resources to prevent cascade failures"""
    
    def __init__(self):
        # Separate thread pools for different operations
        self.critical_pool = ThreadPoolExecutor(max_workers=10)
        self.non_critical_pool = ThreadPoolExecutor(max_workers=5)
        
        # Separate connection pools
        self.critical_db_pool = ConnectionPool(max_connections=20)
        self.analytics_db_pool = ConnectionPool(max_connections=5)
    
    def handle_critical_request(self, request):
        """Use dedicated resources for critical operations"""
        future = self.critical_pool.submit(
            self.process_critical_request, 
            request
        )
        return future.result(timeout=30)
    
    def handle_analytics_request(self, request):
        """Use separate resources for analytics"""
        future = self.non_critical_pool.submit(
            self.process_analytics_request,
            request
        )
        return future.result(timeout=60)
```

## Microservices Deployment

### 1. Containerization with Docker
```dockerfile
# Dockerfile for microservice
FROM python:3.9-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

# Run application
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 2. Kubernetes Deployment
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
      - name: user-service
        image: user-service:v1.0.0
        ports:
        - containerPort: 8000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
  - port: 80
    targetPort: 8000
  type: ClusterIP
```

## Microservices Monitoring

### 1. Distributed Tracing
```python
from opentelemetry import trace
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

class DistributedTracing:
    def __init__(self, service_name):
        # Configure tracing
        trace.set_tracer_provider(TracerProvider())
        tracer = trace.get_tracer(__name__)
        
        # Configure Jaeger exporter
        jaeger_exporter = JaegerExporter(
            agent_host_name="localhost",
            agent_port=6831,
        )
        
        span_processor = BatchSpanProcessor(jaeger_exporter)
        trace.get_tracer_provider().add_span_processor(span_processor)
        
        self.tracer = tracer
        self.service_name = service_name
    
    def trace_operation(self, operation_name):
        """Decorator for tracing operations"""
        def decorator(func):
            def wrapper(*args, **kwargs):
                with self.tracer.start_as_current_span(operation_name) as span:
                    span.set_attribute("service.name", self.service_name)
                    span.set_attribute("operation.name", operation_name)
                    
                    try:
                        result = func(*args, **kwargs)
                        span.set_attribute("operation.success", True)
                        return result
                    except Exception as e:
                        span.set_attribute("operation.success", False)
                        span.set_attribute("error.message", str(e))
                        raise e
            return wrapper
        return decorator
```

### 2. Centralized Logging
```python
import structlog

class MicroserviceLogger:
    def __init__(self, service_name, correlation_id_header='X-Correlation-ID'):
        self.service_name = service_name
        self.correlation_id_header = correlation_id_header
        
        # Configure structured logging
        structlog.configure(
            processors=[
                structlog.stdlib.filter_by_level,
                structlog.stdlib.add_logger_name,
                structlog.stdlib.add_log_level,
                structlog.stdlib.PositionalArgumentsFormatter(),
                structlog.processors.TimeStamper(fmt="iso"),
                structlog.processors.StackInfoRenderer(),
                structlog.processors.format_exc_info,
                structlog.processors.UnicodeDecoder(),
                structlog.processors.JSONRenderer()
            ],
            context_class=dict,
            logger_factory=structlog.stdlib.LoggerFactory(),
            cache_logger_on_first_use=True,
        )
        
        self.logger = structlog.get_logger()
    
    def log_request(self, request, response, duration):
        """Log request with correlation ID"""
        
        correlation_id = request.headers.get(self.correlation_id_header, 'unknown')
        
        self.logger.info(
            "request_processed",
            service=self.service_name,
            correlation_id=correlation_id,
            method=request.method,
            path=request.path,
            status_code=response.status_code,
            duration_ms=duration * 1000,
            user_id=getattr(request, 'user_id', None)
        )
```

### 3. Service Metrics
```python
class ServiceMetrics:
    def __init__(self, service_name):
        self.service_name = service_name
        self.request_count = Counter()
        self.request_duration = Histogram()
        self.error_count = Counter()
    
    def record_request(self, endpoint, method, status_code, duration):
        """Record request metrics"""
        
        labels = {
            'service': self.service_name,
            'endpoint': endpoint,
            'method': method,
            'status_code': status_code
        }
        
        self.request_count.labels(**labels).inc()
        self.request_duration.labels(**labels).observe(duration)
        
        if status_code >= 400:
            self.error_count.labels(**labels).inc()
    
    def get_health_metrics(self):
        """Get service health indicators"""
        
        total_requests = sum(self.request_count._value.values())
        total_errors = sum(self.error_count._value.values())
        
        error_rate = total_errors / total_requests if total_requests > 0 else 0
        avg_response_time = self.calculate_avg_response_time()
        
        return {
            'error_rate': error_rate,
            'avg_response_time': avg_response_time,
            'total_requests': total_requests,
            'health_status': 'healthy' if error_rate < 0.05 else 'unhealthy'
        }
```

## Microservices Challenges and Solutions

### 1. Network Latency
```python
class LatencyOptimization:
    def __init__(self):
        self.cache = Cache()
        self.batch_processor = BatchProcessor()
    
    def optimize_service_calls(self, user_id):
        """Reduce network calls through batching and caching"""
        
        # Batch multiple operations
        operations = [
            ('get_user', user_id),
            ('get_preferences', user_id),
            ('get_recent_activity', user_id)
        ]
        
        # Execute in single batch call
        results = self.batch_processor.execute_batch(operations)
        
        # Cache results for future requests
        for operation, result in results.items():
            cache_key = f"{operation}:{user_id}"
            self.cache.set(cache_key, result, ttl=300)
        
        return results
```

### 2. Service Dependencies
```python
class DependencyManagement:
    def __init__(self):
        self.service_dependencies = {
            'order_service': ['user_service', 'inventory_service'],
            'recommendation_service': ['user_service', 'product_service'],
            'notification_service': ['user_service']
        }
    
    def check_circular_dependencies(self):
        """Detect circular dependencies between services"""
        
        def has_cycle(node, visited, rec_stack):
            visited[node] = True
            rec_stack[node] = True
            
            for neighbor in self.service_dependencies.get(node, []):
                if not visited.get(neighbor, False):
                    if has_cycle(neighbor, visited, rec_stack):
                        return True
                elif rec_stack.get(neighbor, False):
                    return True
            
            rec_stack[node] = False
            return False
        
        visited = {}
        rec_stack = {}
        
        for service in self.service_dependencies:
            if not visited.get(service, False):
                if has_cycle(service, visited, rec_stack):
                    return True
        
        return False
```

### 3. Data Consistency Across Services
```python
class EventualConsistencyManager:
    def __init__(self, event_bus):
        self.event_bus = event_bus
        self.consistency_checkers = []
    
    def ensure_eventual_consistency(self):
        """Background process to check and fix inconsistencies"""
        
        for checker in self.consistency_checkers:
            inconsistencies = checker.find_inconsistencies()
            
            for inconsistency in inconsistencies:
                self.fix_inconsistency(inconsistency)
    
    def fix_inconsistency(self, inconsistency):
        """Fix detected inconsistency"""
        
        if inconsistency['type'] == 'user_profile_mismatch':
            # Re-sync user profile data
            user_id = inconsistency['user_id']
            canonical_data = self.user_service.get_user(user_id)
            
            # Update all services with canonical data
            self.event_bus.publish('user.sync_required', {
                'user_id': user_id,
                'canonical_data': canonical_data
            })
```

## Testing Microservices

### 1. Contract Testing
```python
class ServiceContract:
    """Define and test service contracts"""
    
    def __init__(self, service_name, version):
        self.service_name = service_name
        self.version = version
        self.contracts = {}
    
    def define_contract(self, endpoint, request_schema, response_schema):
        """Define API contract"""
        self.contracts[endpoint] = {
            'request_schema': request_schema,
            'response_schema': response_schema
        }
    
    def test_contract_compliance(self, endpoint, request_data, response_data):
        """Test if actual request/response matches contract"""
        
        contract = self.contracts.get(endpoint)
        if not contract:
            return False, f"No contract defined for {endpoint}"
        
        # Validate request
        if not self.validate_schema(request_data, contract['request_schema']):
            return False, "Request doesn't match contract"
        
        # Validate response
        if not self.validate_schema(response_data, contract['response_schema']):
            return False, "Response doesn't match contract"
        
        return True, "Contract compliant"
```

### 2. Integration Testing
```python
class IntegrationTestSuite:
    def __init__(self, test_environment):
        self.test_env = test_environment
        self.services = {}
    
    def test_order_flow(self):
        """Test complete order processing flow"""
        
        # Setup test data
        test_user = self.create_test_user()
        test_product = self.create_test_product()
        
        # Test the flow
        order_data = {
            'user_id': test_user['id'],
            'items': [{'product_id': test_product['id'], 'quantity': 1}]
        }
        
        # Create order
        order = self.order_service.create_order(order_data)
        assert order['status'] == 'pending'
        
        # Verify inventory updated
        inventory = self.inventory_service.get_inventory(test_product['id'])
        assert inventory['reserved'] == 1
        
        # Process payment
        payment_result = self.payment_service.process_payment(order['id'])
        assert payment_result['status'] == 'success'
        
        # Verify order status updated
        updated_order = self.order_service.get_order(order['id'])
        assert updated_order['status'] == 'confirmed'
```

## Exercise Problems

1. Design a microservices architecture for an e-commerce platform
2. How would you handle distributed transactions in a microservices system?
3. Design service discovery and communication patterns for a social media platform
4. Implement a circuit breaker pattern for service-to-service communication

## Key Takeaways

- Microservices enable independent scaling and technology choices
- Service boundaries should align with business domains
- Network communication introduces latency and failure points
- Data consistency becomes more complex in distributed systems
- Service discovery and API gateways are essential infrastructure
- Monitoring and observability are crucial for debugging
- Start with a monolith, evolve to microservices when needed
- Consider the organizational impact (Conway's Law)

## Next Steps

Move to: **06-api-gateway-patterns.md** and then to **Module 3: Design Patterns**