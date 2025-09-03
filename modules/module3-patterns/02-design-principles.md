# Design Principles

## SOLID Principles for System Design

### Single Responsibility Principle (SRP)
Each service should have only one reason to change.

```python
# Bad: Service doing too many things
class UserService:
    def create_user(self, user_data): pass
    def send_email(self, email, message): pass
    def process_payment(self, payment_data): pass
    def generate_report(self, user_id): pass

# Good: Separate services with single responsibilities
class UserService:
    def create_user(self, user_data): pass
    def get_user(self, user_id): pass
    def update_user(self, user_id, data): pass

class EmailService:
    def send_welcome_email(self, user): pass
    def send_notification(self, email, message): pass

class PaymentService:
    def process_payment(self, payment_data): pass
    def refund_payment(self, payment_id): pass

class ReportingService:
    def generate_user_report(self, user_id): pass
    def generate_sales_report(self, period): pass
```

### Open-Closed Principle (OCP)
Systems should be open for extension but closed for modification.

```python
# Plugin-based extensibility
class NotificationService:
    def __init__(self):
        self.providers = []
    
    def add_provider(self, provider):
        """Extend functionality without modifying existing code"""
        self.providers.append(provider)
    
    def send_notification(self, message, user):
        """Send through all registered providers"""
        for provider in self.providers:
            if provider.supports_user(user):
                provider.send(message, user)

# Extensions
class EmailNotificationProvider:
    def supports_user(self, user):
        return user.email is not None
    
    def send(self, message, user):
        email_service.send(user.email, message)

class SMSNotificationProvider:
    def supports_user(self, user):
        return user.phone is not None
    
    def send(self, message, user):
        sms_service.send(user.phone, message)

class PushNotificationProvider:
    def supports_user(self, user):
        return user.device_token is not None
    
    def send(self, message, user):
        push_service.send(user.device_token, message)
```

### Liskov Substitution Principle (LSP)
Derived classes must be substitutable for their base classes.

```python
# Base cache interface
class Cache(ABC):
    @abstractmethod
    def get(self, key): pass
    
    @abstractmethod
    def set(self, key, value, ttl=None): pass
    
    @abstractmethod
    def delete(self, key): pass

# Implementations must behave consistently
class RedisCache(Cache):
    def __init__(self, redis_client):
        self.redis = redis_client
    
    def get(self, key):
        return self.redis.get(key)
    
    def set(self, key, value, ttl=None):
        if ttl:
            return self.redis.setex(key, ttl, value)
        return self.redis.set(key, value)
    
    def delete(self, key):
        return self.redis.delete(key)

class MemoryCache(Cache):
    def __init__(self):
        self.cache = {}
        self.expiry = {}
    
    def get(self, key):
        if key in self.expiry and time.time() > self.expiry[key]:
            self.delete(key)
            return None
        return self.cache.get(key)
    
    def set(self, key, value, ttl=None):
        self.cache[key] = value
        if ttl:
            self.expiry[key] = time.time() + ttl
    
    def delete(self, key):
        self.cache.pop(key, None)
        self.expiry.pop(key, None)

# Both implementations can be used interchangeably
def use_cache(cache: Cache):
    cache.set("key", "value", ttl=300)
    value = cache.get("key")
    cache.delete("key")
```

### Interface Segregation Principle (ISP)
Clients should not depend on interfaces they don't use.

```python
# Bad: Fat interface
class DatabaseService:
    def create_user(self, user_data): pass
    def get_user(self, user_id): pass
    def create_order(self, order_data): pass
    def get_order(self, order_id): pass
    def generate_analytics(self): pass
    def backup_database(self): pass

# Good: Segregated interfaces
class UserRepository:
    def create_user(self, user_data): pass
    def get_user(self, user_id): pass

class OrderRepository:
    def create_order(self, order_data): pass
    def get_order(self, order_id): pass

class AnalyticsService:
    def generate_user_analytics(self): pass
    def generate_sales_analytics(self): pass

class BackupService:
    def backup_user_data(self): pass
    def backup_order_data(self): pass
```

### Dependency Inversion Principle (DIP)
High-level modules should not depend on low-level modules. Both should depend on abstractions.

```python
# Abstraction
class PaymentProcessor(ABC):
    @abstractmethod
    def process_payment(self, amount, payment_method): pass

# High-level module depends on abstraction
class OrderService:
    def __init__(self, payment_processor: PaymentProcessor):
        self.payment_processor = payment_processor
    
    def complete_order(self, order):
        # Business logic doesn't depend on specific payment implementation
        payment_result = self.payment_processor.process_payment(
            order.total_amount,
            order.payment_method
        )
        
        if payment_result.success:
            order.status = 'completed'
        else:
            order.status = 'payment_failed'

# Low-level modules implement abstraction
class StripePaymentProcessor(PaymentProcessor):
    def process_payment(self, amount, payment_method):
        return stripe.charge(amount, payment_method)

class PayPalPaymentProcessor(PaymentProcessor):
    def process_payment(self, amount, payment_method):
        return paypal.process(amount, payment_method)
```

## Distributed System Principles

### 1. Design for Failure
```python
class ResilientService:
    def __init__(self, external_service):
        self.external_service = external_service
        self.circuit_breaker = CircuitBreaker()
        self.cache = Cache()
        self.fallback_data = FallbackDataProvider()
    
    def get_data(self, key):
        """Get data with multiple failure handling strategies"""
        
        try:
            # Try primary service with circuit breaker
            return self.circuit_breaker.call(
                self.external_service.get_data, key
            )
        except CircuitBreakerOpenError:
            # Circuit breaker is open, try cache
            cached_data = self.cache.get(key)
            if cached_data:
                return cached_data
            
            # Fallback to default data
            return self.fallback_data.get_default(key)
        except Exception as e:
            # Service error, try cache first
            cached_data = self.cache.get(key)
            if cached_data:
                log.warning(f"Service failed, serving from cache: {e}")
                return cached_data
            
            # Final fallback
            log.error(f"All data sources failed: {e}")
            return self.fallback_data.get_default(key)
```

### 2. Idempotency
```python
class IdempotentOperationHandler:
    def __init__(self, operation_store):
        self.operation_store = operation_store
    
    def execute_operation(self, operation_id, operation_func, *args, **kwargs):
        """Execute operation idempotently"""
        
        # Check if operation already executed
        existing_result = self.operation_store.get_result(operation_id)
        if existing_result:
            return existing_result
        
        # Check if operation is in progress
        if self.operation_store.is_in_progress(operation_id):
            # Wait for completion or timeout
            return self.wait_for_completion(operation_id)
        
        try:
            # Mark operation as in progress
            self.operation_store.mark_in_progress(operation_id)
            
            # Execute operation
            result = operation_func(*args, **kwargs)
            
            # Store result
            self.operation_store.store_result(operation_id, result)
            
            return result
            
        except Exception as e:
            # Mark operation as failed
            self.operation_store.mark_failed(operation_id, str(e))
            raise e

# Example usage for payment processing
class PaymentService:
    def __init__(self, idempotent_handler, payment_gateway):
        self.idempotent_handler = idempotent_handler
        self.payment_gateway = payment_gateway
    
    def process_payment(self, payment_request):
        """Process payment idempotently"""
        
        operation_id = payment_request.get('idempotency_key') or str(uuid.uuid4())
        
        return self.idempotent_handler.execute_operation(
            operation_id,
            self._process_payment_internal,
            payment_request
        )
    
    def _process_payment_internal(self, payment_request):
        """Internal payment processing logic"""
        return self.payment_gateway.charge(
            payment_request['amount'],
            payment_request['payment_method']
        )
```

### 3. Graceful Degradation
```python
class GracefulDegradationService:
    def __init__(self):
        self.feature_flags = FeatureFlags()
        self.service_health = ServiceHealthMonitor()
    
    def get_user_recommendations(self, user_id):
        """Get recommendations with graceful degradation"""
        
        # Check if ML recommendation service is healthy
        if self.service_health.is_healthy('recommendation_ml_service'):
            try:
                return self.get_ml_recommendations(user_id)
            except Exception as e:
                log.warning(f"ML recommendations failed: {e}")
        
        # Fallback to rule-based recommendations
        if self.service_health.is_healthy('recommendation_rules_service'):
            try:
                return self.get_rules_based_recommendations(user_id)
            except Exception as e:
                log.warning(f"Rules-based recommendations failed: {e}")
        
        # Final fallback to popular items
        return self.get_popular_items()
    
    def get_search_results(self, query):
        """Search with graceful degradation"""
        
        # Try advanced search with personalization
        if self.feature_flags.is_enabled('advanced_search'):
            try:
                return self.advanced_search_service.search(query)
            except Exception as e:
                log.warning(f"Advanced search failed: {e}")
        
        # Fallback to basic search
        try:
            return self.basic_search_service.search(query)
        except Exception as e:
            log.error(f"Basic search failed: {e}")
            return {'results': [], 'message': 'Search temporarily unavailable'}
```

## Scalability Principles

### 1. Horizontal Scaling
```python
class HorizontallyScalableService:
    def __init__(self):
        self.instances = []
        self.load_balancer = LoadBalancer()
        self.auto_scaler = AutoScaler()
    
    def add_instance(self, instance):
        """Add new service instance"""
        self.instances.append(instance)
        self.load_balancer.add_backend(instance)
    
    def remove_instance(self, instance):
        """Remove service instance"""
        self.instances.remove(instance)
        self.load_balancer.remove_backend(instance)
    
    def handle_request(self, request):
        """Route request to available instance"""
        
        # Check if scaling is needed
        if self.auto_scaler.should_scale_up():
            self.scale_up()
        elif self.auto_scaler.should_scale_down():
            self.scale_down()
        
        # Route request
        return self.load_balancer.route_request(request)
    
    def scale_up(self):
        """Add new instances"""
        new_instance = self.create_new_instance()
        self.add_instance(new_instance)
    
    def scale_down(self):
        """Remove instances gracefully"""
        if len(self.instances) > 1:
            instance_to_remove = self.select_instance_for_removal()
            self.gracefully_shutdown_instance(instance_to_remove)
            self.remove_instance(instance_to_remove)
```

### 2. Stateless Design
```python
class StatelessService:
    """Service that doesn't maintain client state"""
    
    def __init__(self, session_store, cache):
        self.session_store = session_store
        self.cache = cache
    
    def process_request(self, request):
        """Process request without maintaining state"""
        
        # Extract session from request
        session_token = request.headers.get('Authorization')
        session_data = self.session_store.get_session(session_token)
        
        if not session_data:
            return self.unauthorized_response()
        
        # Process request using session data
        result = self.business_logic(request, session_data)
        
        # Update session if needed
        if result.get('update_session'):
            self.session_store.update_session(session_token, result['session_updates'])
        
        return result
    
    def business_logic(self, request, session_data):
        """Business logic that doesn't depend on instance state"""
        
        user_id = session_data['user_id']
        
        # Get user data (could be cached)
        user_data = self.get_user_data(user_id)
        
        # Process based on request and user data
        return self.process_user_request(request, user_data)

# Stateless session management
class ExternalSessionStore:
    def __init__(self, redis_client):
        self.redis = redis_client
    
    def create_session(self, user_id, session_data):
        session_token = str(uuid.uuid4())
        session_key = f"session:{session_token}"
        
        self.redis.setex(
            session_key,
            3600,  # 1 hour TTL
            json.dumps({
                'user_id': user_id,
                'created_at': time.time(),
                **session_data
            })
        )
        
        return session_token
    
    def get_session(self, session_token):
        session_key = f"session:{session_token}"
        session_data = self.redis.get(session_key)
        
        if session_data:
            # Extend session TTL on access
            self.redis.expire(session_key, 3600)
            return json.loads(session_data)
        
        return None
```

### 3. Loose Coupling
```python
# Event-driven loose coupling
class EventBus:
    def __init__(self):
        self.subscribers = defaultdict(list)
    
    def subscribe(self, event_type, handler):
        self.subscribers[event_type].append(handler)
    
    def publish(self, event_type, event_data):
        for handler in self.subscribers[event_type]:
            try:
                # Async execution to avoid blocking
                asyncio.create_task(handler(event_data))
            except Exception as e:
                log.error(f"Event handler failed: {e}")

# Services communicate through events, not direct calls
class UserService:
    def __init__(self, event_bus):
        self.event_bus = event_bus
    
    def create_user(self, user_data):
        user = self.save_user(user_data)
        
        # Publish event instead of calling other services directly
        self.event_bus.publish('user_created', {
            'user_id': user.id,
            'email': user.email,
            'name': user.name
        })
        
        return user

class EmailService:
    def __init__(self, event_bus):
        self.event_bus = event_bus
        # Subscribe to relevant events
        self.event_bus.subscribe('user_created', self.send_welcome_email)
    
    async def send_welcome_email(self, event_data):
        user_id = event_data['user_id']
        email = event_data['email']
        name = event_data['name']
        
        await self.email_client.send_welcome_email(email, name)

class AnalyticsService:
    def __init__(self, event_bus):
        self.event_bus = event_bus
        # Subscribe to the same event for different purpose
        self.event_bus.subscribe('user_created', self.track_user_registration)
    
    async def track_user_registration(self, event_data):
        await self.analytics_client.track_event('user_registered', event_data)
```

## Reliability Principles

### 1. Bulkhead Pattern
```python
class BulkheadIsolation:
    """Isolate resources to prevent cascade failures"""
    
    def __init__(self):
        # Separate thread pools for different operations
        self.critical_operations_pool = ThreadPoolExecutor(max_workers=10)
        self.background_operations_pool = ThreadPoolExecutor(max_workers=5)
        self.analytics_operations_pool = ThreadPoolExecutor(max_workers=3)
        
        # Separate connection pools
        self.critical_db_pool = ConnectionPool(max_connections=20)
        self.analytics_db_pool = ConnectionPool(max_connections=5)
    
    def execute_critical_operation(self, operation, *args, **kwargs):
        """Execute critical operation with dedicated resources"""
        
        future = self.critical_operations_pool.submit(operation, *args, **kwargs)
        return future.result(timeout=30)  # Fast timeout for critical ops
    
    def execute_analytics_operation(self, operation, *args, **kwargs):
        """Execute analytics operation with separate resources"""
        
        future = self.analytics_operations_pool.submit(operation, *args, **kwargs)
        # Longer timeout, failures won't affect critical operations
        return future.result(timeout=300)
```

### 2. Circuit Breaker Pattern
```python
import enum
import time
import threading

class CircuitBreakerState(enum.Enum):
    CLOSED = "CLOSED"
    OPEN = "OPEN"
    HALF_OPEN = "HALF_OPEN"

class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=60, expected_exception=Exception):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.expected_exception = expected_exception
        
        self.failure_count = 0
        self.last_failure_time = None
        self.state = CircuitBreakerState.CLOSED
        self.lock = threading.Lock()
    
    def call(self, func, *args, **kwargs):
        """Execute function through circuit breaker"""
        
        with self.lock:
            if self.state == CircuitBreakerState.OPEN:
                if time.time() - self.last_failure_time > self.recovery_timeout:
                    self.state = CircuitBreakerState.HALF_OPEN
                else:
                    raise CircuitBreakerOpenError("Circuit breaker is OPEN")
            
            if self.state == CircuitBreakerState.HALF_OPEN:
                # Allow one request to test if service recovered
                try:
                    result = func(*args, **kwargs)
                    self.on_success()
                    return result
                except self.expected_exception as e:
                    self.on_failure()
                    raise e
        
        # CLOSED state
        try:
            result = func(*args, **kwargs)
            self.on_success()
            return result
        except self.expected_exception as e:
            self.on_failure()
            raise e
    
    def on_success(self):
        """Reset circuit breaker on successful call"""
        self.failure_count = 0
        self.state = CircuitBreakerState.CLOSED
    
    def on_failure(self):
        """Handle failed call"""
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitBreakerState.OPEN
```

### 3. Retry with Exponential Backoff
```python
import random

class RetryPolicy:
    def __init__(self, max_retries=3, base_delay=1, max_delay=60, jitter=True):
        self.max_retries = max_retries
        self.base_delay = base_delay
        self.max_delay = max_delay
        self.jitter = jitter
    
    def execute_with_retry(self, func, *args, **kwargs):
        """Execute function with exponential backoff retry"""
        
        last_exception = None
        
        for attempt in range(self.max_retries + 1):
            try:
                return func(*args, **kwargs)
            except Exception as e:
                last_exception = e
                
                if attempt == self.max_retries:
                    # Final attempt failed
                    raise e
                
                # Calculate delay for next attempt
                delay = min(
                    self.base_delay * (2 ** attempt),
                    self.max_delay
                )
                
                # Add jitter to prevent thundering herd
                if self.jitter:
                    delay = delay * (0.5 + random.random() * 0.5)
                
                log.warning(f"Attempt {attempt + 1} failed, retrying in {delay:.2f}s: {e}")
                time.sleep(delay)
        
        raise last_exception

# Usage with different retry policies
class ExternalServiceClient:
    def __init__(self):
        # Different retry policies for different operations
        self.critical_retry = RetryPolicy(max_retries=5, base_delay=0.5)
        self.normal_retry = RetryPolicy(max_retries=3, base_delay=1)
        self.background_retry = RetryPolicy(max_retries=10, base_delay=5)
    
    def get_critical_data(self, key):
        return self.critical_retry.execute_with_retry(
            self._make_request, f"/critical/{key}"
        )
    
    def get_normal_data(self, key):
        return self.normal_retry.execute_with_retry(
            self._make_request, f"/data/{key}"
        )
    
    def send_analytics(self, data):
        return self.background_retry.execute_with_retry(
            self._make_request, "/analytics", method="POST", data=data
        )
```

## Performance Principles

### 1. Caching at Multiple Levels
```python
class MultiLevelCachingStrategy:
    def __init__(self):
        # L1: Application memory cache (fastest, smallest)
        self.l1_cache = LRUCache(maxsize=1000)
        
        # L2: Redis cache (fast, larger)
        self.l2_cache = RedisCache()
        
        # L3: Database query cache
        self.l3_cache = DatabaseQueryCache()
        
        # Database
        self.database = Database()
    
    def get_data(self, key):
        """Get data with multi-level caching"""
        
        # Try L1 cache (in-memory)
        data = self.l1_cache.get(key)
        if data is not None:
            return data
        
        # Try L2 cache (Redis)
        data = self.l2_cache.get(key)
        if data is not None:
            # Promote to L1
            self.l1_cache.set(key, data)
            return data
        
        # Try L3 cache (Database cache)
        data = self.l3_cache.get(key)
        if data is not None:
            # Promote to L2 and L1
            self.l2_cache.set(key, data, ttl=3600)
            self.l1_cache.set(key, data)
            return data
        
        # Fetch from database
        data = self.database.get(key)
        if data is not None:
            # Store in all cache levels
            self.l3_cache.set(key, data, ttl=7200)
            self.l2_cache.set(key, data, ttl=3600)
            self.l1_cache.set(key, data)
        
        return data
```

### 2. Asynchronous Processing
```python
class AsyncProcessingPattern:
    def __init__(self, task_queue, result_store):
        self.task_queue = task_queue
        self.result_store = result_store
    
    def submit_long_running_task(self, task_data):
        """Submit task for async processing"""
        
        task_id = str(uuid.uuid4())
        
        # Store initial task status
        self.result_store.set_task_status(task_id, 'pending')
        
        # Queue task for processing
        self.task_queue.enqueue('process_task', {
            'task_id': task_id,
            'task_data': task_data
        })
        
        return {'task_id': task_id, 'status': 'pending'}
    
    def get_task_result(self, task_id):
        """Get task result or status"""
        
        status = self.result_store.get_task_status(task_id)
        
        if status == 'completed':
            result = self.result_store.get_task_result(task_id)
            return {'task_id': task_id, 'status': 'completed', 'result': result}
        elif status == 'failed':
            error = self.result_store.get_task_error(task_id)
            return {'task_id': task_id, 'status': 'failed', 'error': error}
        else:
            return {'task_id': task_id, 'status': status}
    
    def process_task(self, task_data):
        """Background task processor"""
        
        task_id = task_data['task_id']
        
        try:
            # Update status to processing
            self.result_store.set_task_status(task_id, 'processing')
            
            # Execute long-running operation
            result = self.execute_task(task_data['task_data'])
            
            # Store result
            self.result_store.set_task_result(task_id, result)
            self.result_store.set_task_status(task_id, 'completed')
            
        except Exception as e:
            # Store error
            self.result_store.set_task_error(task_id, str(e))
            self.result_store.set_task_status(task_id, 'failed')
```

### 3. Database Connection Pooling
```python
class DatabaseConnectionPool:
    def __init__(self, db_config, min_connections=5, max_connections=20):
        self.db_config = db_config
        self.min_connections = min_connections
        self.max_connections = max_connections
        
        self.available_connections = queue.Queue()
        self.active_connections = set()
        self.total_connections = 0
        self.lock = threading.Lock()
        
        # Initialize minimum connections
        self.initialize_pool()
    
    def initialize_pool(self):
        """Create initial pool of connections"""
        for _ in range(self.min_connections):
            conn = self.create_connection()
            self.available_connections.put(conn)
            self.total_connections += 1
    
    def get_connection(self, timeout=30):
        """Get connection from pool"""
        
        try:
            # Try to get available connection
            conn = self.available_connections.get(timeout=1)
            
            if self.is_connection_healthy(conn):
                with self.lock:
                    self.active_connections.add(conn)
                return conn
            else:
                # Connection is stale, create new one
                self.total_connections -= 1
                return self.get_connection(timeout - 1)
                
        except queue.Empty:
            # No available connections
            with self.lock:
                if self.total_connections < self.max_connections:
                    # Create new connection
                    conn = self.create_connection()
                    self.total_connections += 1
                    self.active_connections.add(conn)
                    return conn
            
            # Wait for connection to become available
            conn = self.available_connections.get(timeout=timeout)
            with self.lock:
                self.active_connections.add(conn)
            return conn
    
    def return_connection(self, conn):
        """Return connection to pool"""
        
        with self.lock:
            self.active_connections.discard(conn)
        
        if self.is_connection_healthy(conn):
            self.available_connections.put(conn)
        else:
            # Connection is unhealthy, don't return to pool
            self.total_connections -= 1
            
            # Ensure minimum connections
            if self.total_connections < self.min_connections:
                new_conn = self.create_connection()
                self.available_connections.put(new_conn)
                self.total_connections += 1
```

## Security Principles

### 1. Defense in Depth
```python
class SecurityLayeredDefense:
    def __init__(self):
        self.layers = [
            NetworkFirewall(),
            ApplicationFirewall(),
            AuthenticationLayer(),
            AuthorizationLayer(),
            InputValidationLayer(),
            OutputSanitizationLayer(),
            AuditingLayer()
        ]
    
    def process_request(self, request):
        """Process request through all security layers"""
        
        for layer in self.layers:
            try:
                request = layer.process(request)
                
                # If layer sets error response, stop processing
                if hasattr(request, 'security_error'):
                    return request.security_error
                    
            except SecurityException as e:
                # Log security incident
                self.audit_log.log_security_incident(
                    layer.__class__.__name__,
                    request,
                    str(e)
                )
                return self.security_error_response(e)
        
        return request

class AuthenticationLayer:
    def process(self, request):
        token = request.headers.get('Authorization')
        
        if not token:
            raise AuthenticationError("No authentication token provided")
        
        user = self.validate_token(token)
        request.user = user
        return request

class AuthorizationLayer:
    def process(self, request):
        if not hasattr(request, 'user'):
            raise AuthorizationError("User not authenticated")
        
        if not self.has_permission(request.user, request.path, request.method):
            raise AuthorizationError("Insufficient permissions")
        
        return request
```

### 2. Principle of Least Privilege
```python
class RoleBasedAccessControl:
    def __init__(self):
        self.roles = {
            'user': ['read_own_profile', 'update_own_profile'],
            'moderator': ['read_any_profile', 'moderate_content'],
            'admin': ['read_any_profile', 'update_any_profile', 'delete_user', 'view_analytics']
        }
        
        self.resource_permissions = {
            'GET:/api/users/{user_id}': ['read_own_profile', 'read_any_profile'],
            'PUT:/api/users/{user_id}': ['update_own_profile', 'update_any_profile'],
            'DELETE:/api/users/{user_id}': ['delete_user'],
            'GET:/api/analytics': ['view_analytics']
        }
    
    def check_permission(self, user, resource, action):
        """Check if user has permission for resource/action"""
        
        user_permissions = set()
        
        # Collect permissions from all user roles
        for role in user.roles:
            user_permissions.update(self.roles.get(role, []))
        
        # Check resource permissions
        resource_key = f"{action}:{resource}"
        required_permissions = self.resource_permissions.get(resource_key, [])
        
        # Check if user has any required permission
        return bool(user_permissions.intersection(required_permissions))
    
    def filter_accessible_resources(self, user, resources):
        """Filter resources based on user permissions"""
        
        accessible = []
        
        for resource in resources:
            if self.check_permission(user, resource.path, 'GET'):
                # Remove sensitive fields based on permissions
                filtered_resource = self.filter_sensitive_fields(user, resource)
                accessible.append(filtered_resource)
        
        return accessible
```

## Observability Principles

### 1. The Three Pillars of Observability
```python
class ObservabilitySystem:
    def __init__(self):
        # Metrics
        self.metrics_collector = MetricsCollector()
        
        # Logging
        self.logger = StructuredLogger()
        
        # Tracing
        self.tracer = DistributedTracer()
    
    def instrument_operation(self, operation_name):
        """Decorator to add observability to operations"""
        
        def decorator(func):
            def wrapper(*args, **kwargs):
                # Start trace
                with self.tracer.start_span(operation_name) as span:
                    start_time = time.time()
                    
                    try:
                        # Execute operation
                        result = func(*args, **kwargs)
                        
                        # Record success metrics
                        duration = time.time() - start_time
                        self.metrics_collector.record_operation(
                            operation_name, 'success', duration
                        )
                        
                        # Log operation
                        self.logger.info(
                            f"Operation {operation_name} completed",
                            duration=duration,
                            args=args,
                            result_type=type(result).__name__
                        )
                        
                        # Add span tags
                        span.set_tag('operation.success', True)
                        span.set_tag('operation.duration', duration)
                        
                        return result
                        
                    except Exception as e:
                        # Record failure metrics
                        duration = time.time() - start_time
                        self.metrics_collector.record_operation(
                            operation_name, 'failure', duration
                        )
                        
                        # Log error
                        self.logger.error(
                            f"Operation {operation_name} failed",
                            duration=duration,
                            error=str(e),
                            args=args
                        )
                        
                        # Add span tags
                        span.set_tag('operation.success', False)
                        span.set_tag('error.message', str(e))
                        
                        raise e
            
            return wrapper
        return decorator

# Usage
@observability.instrument_operation("create_user")
def create_user(user_data):
    # Business logic here
    return user_service.create_user(user_data)
```

### 2. Correlation IDs
```python
class CorrelationIDMiddleware:
    def __init__(self, header_name='X-Correlation-ID'):
        self.header_name = header_name
    
    def process_request(self, request):
        """Add correlation ID to request"""
        
        # Get existing correlation ID or create new one
        correlation_id = request.headers.get(self.header_name)
        if not correlation_id:
            correlation_id = str(uuid.uuid4())
            request.headers[self.header_name] = correlation_id
        
        # Add to request context
        request.correlation_id = correlation_id
        
        # Set in thread-local storage for logging
        self.set_correlation_context(correlation_id)
        
        return request
    
    def forward_correlation_id(self, outbound_request, correlation_id):
        """Forward correlation ID to downstream services"""
        
        outbound_request.headers[self.header_name] = correlation_id
        return outbound_request

class CorrelationAwareLogger:
    def __init__(self):
        self.context = threading.local()
    
    def set_correlation_id(self, correlation_id):
        self.context.correlation_id = correlation_id
    
    def log(self, level, message, **kwargs):
        """Log with correlation ID"""
        
        correlation_id = getattr(self.context, 'correlation_id', 'unknown')
        
        log_entry = {
            'timestamp': time.time(),
            'level': level,
            'message': message,
            'correlation_id': correlation_id,
            **kwargs
        }
        
        print(json.dumps(log_entry))
```

## Exercise Problems

1. Design a resilient service that handles multiple types of failures
2. Implement a plugin architecture for a content management system
3. Create an idempotent API for financial transactions
4. Design an observability strategy for a microservices architecture

## Key Takeaways

- SOLID principles apply to system architecture, not just code
- Design for failure from the beginning
- Loose coupling enables independent evolution of services
- Stateless design enables horizontal scaling
- Multiple levels of caching provide optimal performance
- Security should be implemented in layers
- Observability is essential for understanding system behavior
- Idempotency is crucial for reliable distributed systems
- Circuit breakers prevent cascade failures
- Correlation IDs enable distributed debugging

## Next Steps

Move to: **03-trade-offs-analysis.md**