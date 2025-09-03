# Architectural Patterns

## Layered Architecture Pattern

### Traditional N-Tier Architecture
```
Presentation Layer (UI)
      ↓
Business Logic Layer
      ↓
Data Access Layer
      ↓
Database Layer
```

### Implementation Example
```python
# Presentation Layer
class UserController:
    def __init__(self, user_service):
        self.user_service = user_service
    
    def create_user(self, request):
        try:
            user_data = self.validate_request(request)
            user = self.user_service.create_user(user_data)
            return self.success_response(user)
        except ValidationError as e:
            return self.error_response(400, str(e))

# Business Logic Layer
class UserService:
    def __init__(self, user_repository, email_service):
        self.user_repository = user_repository
        self.email_service = email_service
    
    def create_user(self, user_data):
        # Business logic
        if self.user_repository.email_exists(user_data['email']):
            raise BusinessLogicError("Email already exists")
        
        user = self.user_repository.save(user_data)
        self.email_service.send_welcome_email(user['email'])
        
        return user

# Data Access Layer
class UserRepository:
    def __init__(self, database):
        self.db = database
    
    def save(self, user_data):
        return self.db.insert('users', user_data)
    
    def email_exists(self, email):
        return self.db.exists('users', {'email': email})
```

## Event-Driven Architecture

### Event Sourcing Pattern
```python
class EventStore:
    def __init__(self):
        self.events = []
        self.snapshots = {}
    
    def append_event(self, aggregate_id, event_type, event_data, version):
        event = {
            'aggregate_id': aggregate_id,
            'event_type': event_type,
            'event_data': event_data,
            'version': version,
            'timestamp': time.time()
        }
        
        self.events.append(event)
        return event
    
    def get_events(self, aggregate_id, from_version=0):
        return [
            event for event in self.events
            if event['aggregate_id'] == aggregate_id and event['version'] > from_version
        ]
    
    def create_snapshot(self, aggregate_id, state, version):
        self.snapshots[aggregate_id] = {
            'state': state,
            'version': version,
            'timestamp': time.time()
        }

class UserAggregate:
    def __init__(self, user_id, event_store):
        self.user_id = user_id
        self.event_store = event_store
        self.version = 0
        self.state = {}
    
    def create_user(self, user_data):
        """Create user command"""
        if self.state:
            raise ValueError("User already exists")
        
        event = self.event_store.append_event(
            self.user_id,
            'UserCreated',
            user_data,
            self.version + 1
        )
        
        self.apply_event(event)
    
    def update_email(self, new_email):
        """Update email command"""
        if not self.state:
            raise ValueError("User does not exist")
        
        event = self.event_store.append_event(
            self.user_id,
            'EmailUpdated',
            {'old_email': self.state['email'], 'new_email': new_email},
            self.version + 1
        )
        
        self.apply_event(event)
    
    def apply_event(self, event):
        """Apply event to aggregate state"""
        
        if event['event_type'] == 'UserCreated':
            self.state = event['event_data'].copy()
        elif event['event_type'] == 'EmailUpdated':
            self.state['email'] = event['event_data']['new_email']
        
        self.version = event['version']
    
    def load_from_history(self):
        """Rebuild state from events"""
        
        # Try to load from snapshot first
        snapshot = self.event_store.snapshots.get(self.user_id)
        if snapshot:
            self.state = snapshot['state'].copy()
            self.version = snapshot['version']
            from_version = snapshot['version']
        else:
            from_version = 0
        
        # Apply events after snapshot
        events = self.event_store.get_events(self.user_id, from_version)
        for event in events:
            self.apply_event(event)
```

### CQRS (Command Query Responsibility Segregation)
```python
# Command Side (Write Model)
class UserCommandHandler:
    def __init__(self, event_store, event_bus):
        self.event_store = event_store
        self.event_bus = event_bus
    
    def handle_create_user(self, command):
        user_aggregate = UserAggregate(command['user_id'], self.event_store)
        user_aggregate.create_user(command['user_data'])
        
        # Publish domain event
        self.event_bus.publish('UserCreated', {
            'user_id': command['user_id'],
            'user_data': command['user_data']
        })

# Query Side (Read Model)
class UserQueryHandler:
    def __init__(self, read_database):
        self.read_db = read_database
    
    def get_user_profile(self, user_id):
        """Optimized read model for user profiles"""
        return self.read_db.get_user_profile(user_id)
    
    def search_users(self, criteria):
        """Optimized search functionality"""
        return self.read_db.search_users(criteria)
    
    def get_user_statistics(self, user_id):
        """Pre-computed statistics"""
        return self.read_db.get_user_stats(user_id)

# Event Handler to update read models
class UserReadModelUpdater:
    def __init__(self, read_database, event_bus):
        self.read_db = read_database
        event_bus.subscribe('UserCreated', self.handle_user_created)
        event_bus.subscribe('EmailUpdated', self.handle_email_updated)
    
    def handle_user_created(self, event):
        """Update read model when user is created"""
        user_data = event['user_data']
        
        # Create optimized read model
        self.read_db.create_user_profile({
            'user_id': event['user_id'],
            'name': user_data['name'],
            'email': user_data['email'],
            'created_at': time.time(),
            'search_terms': self.generate_search_terms(user_data)
        })
    
    def handle_email_updated(self, event):
        """Update read model when email changes"""
        self.read_db.update_user_email(
            event['user_id'],
            event['new_email']
        )
```

## Hexagonal Architecture (Ports and Adapters)

### Core Domain
```python
# Domain Model (Pure business logic)
class User:
    def __init__(self, user_id, name, email):
        self.user_id = user_id
        self.name = name
        self.email = email
        self.created_at = time.time()
    
    def change_email(self, new_email):
        if not self.is_valid_email(new_email):
            raise ValueError("Invalid email format")
        
        old_email = self.email
        self.email = new_email
        
        return EmailChangedEvent(self.user_id, old_email, new_email)
    
    def is_valid_email(self, email):
        return "@" in email and "." in email

# Domain Service
class UserDomainService:
    def __init__(self, user_repository, email_service):
        self.user_repository = user_repository
        self.email_service = email_service
    
    def register_user(self, user_data):
        # Check business rules
        if self.user_repository.email_exists(user_data['email']):
            raise DomainError("Email already registered")
        
        # Create domain object
        user = User(
            user_id=str(uuid.uuid4()),
            name=user_data['name'],
            email=user_data['email']
        )
        
        # Persist
        self.user_repository.save(user)
        
        # Send welcome email
        self.email_service.send_welcome_email(user.email, user.name)
        
        return user
```

### Ports (Interfaces)
```python
from abc import ABC, abstractmethod

# Primary Port (API)
class UserService(ABC):
    @abstractmethod
    def register_user(self, user_data): pass
    
    @abstractmethod
    def get_user(self, user_id): pass
    
    @abstractmethod
    def update_user_email(self, user_id, new_email): pass

# Secondary Ports (Infrastructure interfaces)
class UserRepository(ABC):
    @abstractmethod
    def save(self, user): pass
    
    @abstractmethod
    def get_by_id(self, user_id): pass
    
    @abstractmethod
    def email_exists(self, email): pass

class EmailService(ABC):
    @abstractmethod
    def send_welcome_email(self, email, name): pass
    
    @abstractmethod
    def send_email_change_notification(self, old_email, new_email): pass
```

### Adapters (Implementations)
```python
# Primary Adapter (REST API)
class RESTUserAdapter:
    def __init__(self, user_service):
        self.user_service = user_service
    
    def post_users(self, request):
        """Handle POST /users"""
        try:
            user_data = request.json
            user = self.user_service.register_user(user_data)
            return {'status': 201, 'data': self.serialize_user(user)}
        except DomainError as e:
            return {'status': 400, 'error': str(e)}

# Secondary Adapter (Database)
class PostgreSQLUserRepository(UserRepository):
    def __init__(self, database):
        self.db = database
    
    def save(self, user):
        return self.db.insert('users', {
            'user_id': user.user_id,
            'name': user.name,
            'email': user.email,
            'created_at': user.created_at
        })
    
    def get_by_id(self, user_id):
        row = self.db.select_one('users', {'user_id': user_id})
        if row:
            return User(row['user_id'], row['name'], row['email'])
        return None
    
    def email_exists(self, email):
        return self.db.exists('users', {'email': email})

# Secondary Adapter (Email Service)
class SMTPEmailAdapter(EmailService):
    def __init__(self, smtp_config):
        self.smtp_config = smtp_config
    
    def send_welcome_email(self, email, name):
        subject = f"Welcome {name}!"
        body = f"Hello {name}, welcome to our platform!"
        self.send_email(email, subject, body)
    
    def send_email(self, to_email, subject, body):
        # SMTP implementation
        pass
```

## Microkernel Architecture

### Plugin-Based System
```python
class PluginManager:
    def __init__(self):
        self.plugins = {}
        self.hooks = defaultdict(list)
    
    def register_plugin(self, plugin_name, plugin_instance):
        """Register a plugin"""
        self.plugins[plugin_name] = plugin_instance
        
        # Register plugin hooks
        for hook_name in plugin_instance.get_hooks():
            self.hooks[hook_name].append(plugin_instance)
    
    def execute_hook(self, hook_name, *args, **kwargs):
        """Execute all plugins for a specific hook"""
        results = []
        
        for plugin in self.hooks[hook_name]:
            try:
                result = plugin.execute_hook(hook_name, *args, **kwargs)
                results.append(result)
            except Exception as e:
                log.error(f"Plugin {plugin.__class__.__name__} failed: {e}")
        
        return results

# Core System
class CoreSystem:
    def __init__(self, plugin_manager):
        self.plugin_manager = plugin_manager
    
    def process_user_registration(self, user_data):
        """Core user registration with plugin hooks"""
        
        # Pre-registration hooks
        self.plugin_manager.execute_hook('pre_user_registration', user_data)
        
        # Core registration logic
        user = self.create_user(user_data)
        
        # Post-registration hooks
        self.plugin_manager.execute_hook('post_user_registration', user)
        
        return user

# Example Plugins
class EmailNotificationPlugin:
    def get_hooks(self):
        return ['post_user_registration']
    
    def execute_hook(self, hook_name, *args, **kwargs):
        if hook_name == 'post_user_registration':
            user = args[0]
            self.send_welcome_email(user)

class AnalyticsPlugin:
    def get_hooks(self):
        return ['post_user_registration']
    
    def execute_hook(self, hook_name, *args, **kwargs):
        if hook_name == 'post_user_registration':
            user = args[0]
            self.track_user_registration(user)
```

## Service Mesh Architecture

### Sidecar Proxy Pattern
```python
class SidecarProxy:
    def __init__(self, service_name, service_port):
        self.service_name = service_name
        self.service_port = service_port
        self.metrics = ServiceMetrics()
        self.circuit_breaker = CircuitBreaker()
        self.retry_policy = RetryPolicy()
    
    async def handle_inbound_request(self, request):
        """Handle incoming requests to the service"""
        
        # Apply policies
        if not await self.rate_limit_check(request):
            return self.rate_limit_response()
        
        # Forward to local service
        response = await self.forward_to_local_service(request)
        
        # Record metrics
        self.metrics.record_inbound_request(request, response)
        
        return response
    
    async def handle_outbound_request(self, target_service, request):
        """Handle outgoing requests from the service"""
        
        # Service discovery
        target_instances = await self.discover_service(target_service)
        
        # Load balancing
        selected_instance = self.select_instance(target_instances)
        
        # Apply circuit breaker
        try:
            response = await self.circuit_breaker.call(
                self.make_request,
                selected_instance,
                request
            )
            
            self.metrics.record_outbound_request(target_service, request, response)
            return response
            
        except CircuitBreakerOpenError:
            return self.fallback_response(target_service, request)
```

### Service Mesh Control Plane
```python
class ServiceMeshControlPlane:
    def __init__(self):
        self.service_registry = {}
        self.policies = {}
        self.metrics_collector = MetricsCollector()
    
    def register_service(self, service_name, instance_info):
        """Register service instance"""
        if service_name not in self.service_registry:
            self.service_registry[service_name] = []
        
        self.service_registry[service_name].append(instance_info)
    
    def set_traffic_policy(self, service_name, policy):
        """Set traffic management policy"""
        self.policies[service_name] = policy
    
    def get_service_config(self, service_name):
        """Get configuration for service proxies"""
        
        return {
            'instances': self.service_registry.get(service_name, []),
            'policy': self.policies.get(service_name, self.default_policy()),
            'health_check_config': self.get_health_check_config(service_name)
        }
    
    def collect_telemetry(self, service_name, metrics_data):
        """Collect telemetry from service proxies"""
        self.metrics_collector.record(service_name, metrics_data)
```

## Serverless Architecture

### Function-as-a-Service Pattern
```python
# AWS Lambda function example
def lambda_handler(event, context):
    """Process user registration"""
    
    try:
        # Parse event
        user_data = json.loads(event['body'])
        
        # Validate input
        if not user_data.get('email'):
            return {
                'statusCode': 400,
                'body': json.dumps({'error': 'Email is required'})
            }
        
        # Business logic
        user_id = str(uuid.uuid4())
        
        # Store in database
        dynamodb = boto3.resource('dynamodb')
        table = dynamodb.Table('users')
        
        table.put_item(Item={
            'user_id': user_id,
            'email': user_data['email'],
            'name': user_data['name'],
            'created_at': int(time.time())
        })
        
        # Trigger welcome email (async)
        sns = boto3.client('sns')
        sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:welcome-emails',
            Message=json.dumps({
                'user_id': user_id,
                'email': user_data['email'],
                'name': user_data['name']
            })
        )
        
        return {
            'statusCode': 201,
            'body': json.dumps({'user_id': user_id})
        }
        
    except Exception as e:
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }

# Serverless workflow orchestration
class ServerlessWorkflow:
    def __init__(self, step_functions_client):
        self.step_functions = step_functions_client
    
    def start_order_processing(self, order_data):
        """Start serverless order processing workflow"""
        
        workflow_input = {
            'order_id': order_data['order_id'],
            'user_id': order_data['user_id'],
            'items': order_data['items'],
            'payment_info': order_data['payment_info']
        }
        
        response = self.step_functions.start_execution(
            stateMachineArn='arn:aws:states:us-east-1:123456789012:stateMachine:order-processing',
            input=json.dumps(workflow_input)
        )
        
        return response['executionArn']
```

## Data Pipeline Architectures

### Lambda Architecture
```python
class LambdaArchitecture:
    """Combines batch and stream processing"""
    
    def __init__(self):
        # Batch Layer (Hadoop/Spark)
        self.batch_processor = BatchProcessor()
        
        # Speed Layer (Storm/Kafka Streams)
        self.stream_processor = StreamProcessor()
        
        # Serving Layer (HBase/Cassandra)
        self.serving_layer = ServingLayer()
    
    def process_data(self, raw_data):
        """Process data through both batch and speed layers"""
        
        # Store raw data for batch processing
        self.batch_processor.store_raw_data(raw_data)
        
        # Process in real-time
        real_time_view = self.stream_processor.process(raw_data)
        
        # Merge with batch view
        batch_view = self.serving_layer.get_batch_view(raw_data.key)
        
        # Combine views
        final_view = self.merge_views(batch_view, real_time_view)
        
        return final_view
    
    def run_batch_job(self):
        """Periodic batch processing"""
        
        # Process all accumulated data
        batch_results = self.batch_processor.process_batch()
        
        # Update serving layer
        self.serving_layer.update_batch_views(batch_results)
        
        # Clean up processed real-time data
        self.stream_processor.cleanup_processed_data()
```

### Kappa Architecture
```python
class KappaArchitecture:
    """Stream processing only architecture"""
    
    def __init__(self):
        self.stream_processor = StreamProcessor()
        self.state_store = StateStore()
        self.event_log = EventLog()
    
    def process_event(self, event):
        """Process event through stream processor"""
        
        # Store event in log
        self.event_log.append(event)
        
        # Process through stream
        result = self.stream_processor.process(event)
        
        # Update state
        self.state_store.update(event.key, result)
        
        return result
    
    def reprocess_from_timestamp(self, timestamp):
        """Reprocess data from specific point in time"""
        
        # Reset state
        self.state_store.reset()
        
        # Replay events from timestamp
        events = self.event_log.get_events_from(timestamp)
        
        for event in events:
            self.process_event(event)
```

## Architectural Decision Framework

### Architecture Decision Records (ADRs)
```markdown
# ADR-001: Choose Database Technology

## Status
Accepted

## Context
We need to choose a primary database technology for our user management system.

Requirements:
- ACID transactions for user data
- Complex queries for reporting
- Mature ecosystem and tooling
- Strong consistency for financial data

## Decision
We will use PostgreSQL as our primary database.

## Consequences

### Positive
- Strong ACID guarantees
- Excellent query capabilities
- Mature ecosystem
- Strong consistency

### Negative
- Vertical scaling limitations
- More complex horizontal scaling
- Single point of failure without replication

### Mitigation
- Implement read replicas for scaling reads
- Set up automated backups
- Plan for sharding if needed in future
```

### Trade-off Analysis Matrix
```python
class ArchitectureEvaluator:
    def __init__(self):
        self.criteria = [
            'scalability', 'performance', 'reliability', 
            'maintainability', 'cost', 'complexity'
        ]
        self.weight = {
            'scalability': 0.2,
            'performance': 0.2,
            'reliability': 0.2,
            'maintainability': 0.15,
            'cost': 0.15,
            'complexity': 0.1
        }
    
    def evaluate_architecture(self, architecture_name, scores):
        """Evaluate architecture based on weighted criteria"""
        
        total_score = 0
        for criterion in self.criteria:
            score = scores.get(criterion, 0)
            weight = self.weight[criterion]
            total_score += score * weight
        
        return {
            'architecture': architecture_name,
            'total_score': total_score,
            'detailed_scores': scores
        }
    
    def compare_architectures(self, architectures):
        """Compare multiple architectural options"""
        
        evaluations = []
        
        for arch_name, scores in architectures.items():
            evaluation = self.evaluate_architecture(arch_name, scores)
            evaluations.append(evaluation)
        
        # Sort by total score
        evaluations.sort(key=lambda x: x['total_score'], reverse=True)
        
        return evaluations

# Example usage
evaluator = ArchitectureEvaluator()

architectures = {
    'monolith': {
        'scalability': 6,
        'performance': 8,
        'reliability': 7,
        'maintainability': 5,
        'cost': 9,
        'complexity': 9
    },
    'microservices': {
        'scalability': 9,
        'performance': 7,
        'reliability': 8,
        'maintainability': 7,
        'cost': 6,
        'complexity': 4
    },
    'serverless': {
        'scalability': 10,
        'performance': 6,
        'reliability': 8,
        'maintainability': 8,
        'cost': 8,
        'complexity': 6
    }
}

comparison = evaluator.compare_architectures(architectures)
```

## Domain-Driven Design (DDD) Patterns

### Bounded Context
```python
# User Management Bounded Context
class UserManagementContext:
    def __init__(self):
        self.user_repository = UserRepository()
        self.user_service = UserService(self.user_repository)
    
    class User:
        """User entity within this context"""
        def __init__(self, user_id, email, profile):
            self.user_id = user_id
            self.email = email
            self.profile = profile

# Order Management Bounded Context
class OrderManagementContext:
    def __init__(self):
        self.order_repository = OrderRepository()
        self.order_service = OrderService(self.order_repository)
    
    class User:
        """User entity within order context (different from UserManagementContext.User)"""
        def __init__(self, user_id, billing_address, shipping_address):
            self.user_id = user_id
            self.billing_address = billing_address
            self.shipping_address = shipping_address
```

### Aggregate Pattern
```python
class OrderAggregate:
    def __init__(self, order_id):
        self.order_id = order_id
        self.items = []
        self.status = 'draft'
        self.total_amount = 0
        self.events = []
    
    def add_item(self, product_id, quantity, price):
        """Add item to order"""
        
        # Business rule: Can't add items to confirmed orders
        if self.status != 'draft':
            raise DomainError("Cannot modify confirmed order")
        
        # Business rule: Maximum 10 items per order
        if len(self.items) >= 10:
            raise DomainError("Maximum 10 items per order")
        
        item = OrderItem(product_id, quantity, price)
        self.items.append(item)
        self.total_amount += quantity * price
        
        # Record domain event
        self.events.append(ItemAddedEvent(self.order_id, product_id, quantity))
    
    def confirm_order(self):
        """Confirm the order"""
        
        # Business rule: Order must have items
        if not self.items:
            raise DomainError("Cannot confirm empty order")
        
        self.status = 'confirmed'
        self.events.append(OrderConfirmedEvent(self.order_id, self.total_amount))
    
    def get_uncommitted_events(self):
        """Get events that haven't been persisted yet"""
        return self.events
    
    def mark_events_as_committed(self):
        """Clear events after they've been persisted"""
        self.events = []
```

### Repository Pattern with Unit of Work
```python
class UnitOfWork:
    def __init__(self):
        self.repositories = {}
        self.new_objects = []
        self.dirty_objects = []
        self.removed_objects = []
    
    def register_new(self, obj):
        self.new_objects.append(obj)
    
    def register_dirty(self, obj):
        if obj not in self.dirty_objects:
            self.dirty_objects.append(obj)
    
    def register_removed(self, obj):
        self.removed_objects.append(obj)
    
    def commit(self):
        """Commit all changes in a single transaction"""
        
        with database_transaction():
            # Insert new objects
            for obj in self.new_objects:
                self.get_repository(obj).insert(obj)
            
            # Update dirty objects
            for obj in self.dirty_objects:
                self.get_repository(obj).update(obj)
            
            # Delete removed objects
            for obj in self.removed_objects:
                self.get_repository(obj).delete(obj)
            
            # Publish domain events
            self.publish_domain_events()
        
        # Clear unit of work
        self.clear()
```

## Patterns for Scalability

### Sharding Patterns

#### Range-Based Sharding
```python
class RangeBasedSharding:
    def __init__(self, shards):
        self.shards = shards
        self.ranges = [
            {'min': 0, 'max': 1000000, 'shard': 'shard_1'},
            {'min': 1000001, 'max': 2000000, 'shard': 'shard_2'},
            {'min': 2000001, 'max': 3000000, 'shard': 'shard_3'}
        ]
    
    def get_shard(self, key):
        """Get shard based on key range"""
        
        for range_config in self.ranges:
            if range_config['min'] <= key <= range_config['max']:
                return self.shards[range_config['shard']]
        
        raise ValueError(f"No shard found for key: {key}")
```

#### Hash-Based Sharding
```python
class HashBasedSharding:
    def __init__(self, shards):
        self.shards = shards
        self.num_shards = len(shards)
    
    def get_shard(self, key):
        """Get shard based on hash of key"""
        
        hash_value = hash(str(key))
        shard_index = hash_value % self.num_shards
        
        return self.shards[shard_index]
    
    def add_shard(self, new_shard):
        """Add new shard (requires rehashing)"""
        
        old_shards = self.shards.copy()
        self.shards.append(new_shard)
        self.num_shards += 1
        
        # Rehash and migrate data
        self.migrate_data(old_shards)
```

#### Consistent Hashing
```python
class ConsistentHashing:
    def __init__(self, nodes, virtual_nodes=150):
        self.nodes = nodes
        self.virtual_nodes = virtual_nodes
        self.ring = {}
        self.sorted_keys = []
        self.build_ring()
    
    def build_ring(self):
        """Build the hash ring"""
        
        self.ring = {}
        
        for node in self.nodes:
            for i in range(self.virtual_nodes):
                virtual_key = f"{node}:{i}"
                hash_value = self.hash_function(virtual_key)
                self.ring[hash_value] = node
        
        self.sorted_keys = sorted(self.ring.keys())
    
    def get_node(self, key):
        """Get node for given key"""
        
        if not self.ring:
            return None
        
        hash_value = self.hash_function(key)
        
        # Find first node clockwise
        for ring_key in self.sorted_keys:
            if hash_value <= ring_key:
                return self.ring[ring_key]
        
        # Wrap around to first node
        return self.ring[self.sorted_keys[0]]
    
    def add_node(self, node):
        """Add new node with minimal data movement"""
        
        self.nodes.append(node)
        self.build_ring()
    
    def remove_node(self, node):
        """Remove node"""
        
        self.nodes.remove(node)
        self.build_ring()
```

### CQRS with Event Sourcing
```python
class CQRSEventSourcingSystem:
    def __init__(self):
        self.event_store = EventStore()
        self.command_handlers = {}
        self.event_handlers = {}
        self.read_models = {}
    
    def register_command_handler(self, command_type, handler):
        self.command_handlers[command_type] = handler
    
    def register_event_handler(self, event_type, handler):
        if event_type not in self.event_handlers:
            self.event_handlers[event_type] = []
        self.event_handlers[event_type].append(handler)
    
    def handle_command(self, command):
        """Handle command and generate events"""
        
        command_type = command['type']
        handler = self.command_handlers.get(command_type)
        
        if not handler:
            raise ValueError(f"No handler for command: {command_type}")
        
        # Execute command and get events
        events = handler.handle(command)
        
        # Store events
        for event in events:
            self.event_store.append(event)
            
            # Trigger event handlers
            self.handle_event(event)
    
    def handle_event(self, event):
        """Handle domain event"""
        
        event_type = event['type']
        handlers = self.event_handlers.get(event_type, [])
        
        for handler in handlers:
            try:
                handler.handle(event)
            except Exception as e:
                log.error(f"Event handler failed: {e}")
```

## Exercise Problems

1. Design an event-driven architecture for a real-time chat application
2. Compare monolithic vs microservices vs serverless architectures for a startup
3. Implement a hexagonal architecture for a payment processing system
4. Design a data pipeline architecture for real-time analytics

## Key Takeaways

- Choose architectural patterns based on system requirements
- Layered architecture provides clear separation of concerns
- Event-driven architectures enable loose coupling and scalability
- Hexagonal architecture makes systems more testable and maintainable
- Microkernel architecture provides flexibility through plugins
- Service mesh simplifies microservices communication
- Serverless architectures reduce operational overhead
- CQRS separates read and write concerns for better performance
- Consider trade-offs carefully when selecting patterns

## Next Steps

Move to: **02-design-principles.md**