# API Gateway Patterns

## What is an API Gateway?

An API Gateway is a server that acts as an entry point for all client requests to backend microservices. It handles request routing, composition, and protocol translation.

## API Gateway Responsibilities

### Core Functions
1. **Request Routing**: Direct requests to appropriate services
2. **Authentication & Authorization**: Centralized security
3. **Rate Limiting**: Protect backend services
4. **Request/Response Transformation**: Protocol translation
5. **Monitoring & Analytics**: Centralized observability

### Advanced Functions
6. **Caching**: Reduce backend load
7. **Load Balancing**: Distribute traffic
8. **Circuit Breaking**: Fault tolerance
9. **API Versioning**: Backward compatibility
10. **Compression**: Optimize bandwidth

## API Gateway Implementation

### Basic Gateway Structure
```python
class APIGateway:
    def __init__(self):
        self.routes = {}
        self.middlewares = []
        self.service_discovery = ServiceDiscovery()
        self.auth_service = AuthenticationService()
        self.rate_limiter = RateLimiter()
        self.cache = Cache()
    
    def add_route(self, path_pattern, target_service, methods=['GET']):
        """Register route to service mapping"""
        self.routes[path_pattern] = {
            'service': target_service,
            'methods': methods
        }
    
    def add_middleware(self, middleware):
        """Add middleware to processing pipeline"""
        self.middlewares.append(middleware)
    
    async def handle_request(self, request):
        """Process incoming request through pipeline"""
        
        try:
            # Apply middlewares
            for middleware in self.middlewares:
                request = await middleware.process_request(request)
                if hasattr(request, 'response'):
                    return request.response
            
            # Route to backend service
            response = await self.route_request(request)
            
            # Apply response middlewares (reverse order)
            for middleware in reversed(self.middlewares):
                response = await middleware.process_response(request, response)
            
            return response
            
        except Exception as e:
            return self.error_response(500, str(e))
    
    async def route_request(self, request):
        """Route request to appropriate backend service"""
        
        # Find matching route
        route = self.find_route(request.path, request.method)
        if not route:
            return self.error_response(404, "Route not found")
        
        # Discover service instances
        instances = self.service_discovery.get_instances(route['service'])
        if not instances:
            return self.error_response(503, "Service unavailable")
        
        # Select instance (load balancing)
        instance = self.select_instance(instances)
        
        # Forward request
        return await self.forward_request(request, instance)
```

## Gateway Patterns

### 1. Backend for Frontend (BFF)
```python
class MobileBFF(APIGateway):
    """Mobile-optimized API gateway"""
    
    def __init__(self):
        super().__init__()
        self.user_service = UserServiceClient()
        self.product_service = ProductServiceClient()
        self.order_service = OrderServiceClient()
    
    async def get_mobile_dashboard(self, user_id):
        """Optimized dashboard for mobile clients"""
        
        # Fetch data in parallel
        user_task = asyncio.create_task(
            self.user_service.get_user_summary(user_id)
        )
        orders_task = asyncio.create_task(
            self.order_service.get_recent_orders(user_id, limit=3)
        )
        recommendations_task = asyncio.create_task(
            self.product_service.get_recommendations(user_id, limit=5)
        )
        
        # Wait for all responses
        user_data, recent_orders, recommendations = await asyncio.gather(
            user_task, orders_task, recommendations_task
        )
        
        # Compose mobile-optimized response
        return {
            'user': {
                'name': user_data['name'],
                'avatar': user_data['avatar_thumbnail']  # Smaller image for mobile
            },
            'recent_orders': [
                {
                    'id': order['id'],
                    'status': order['status'],
                    'total': order['total']
                } for order in recent_orders
            ],
            'recommendations': [
                {
                    'id': product['id'],
                    'name': product['name'],
                    'price': product['price'],
                    'image': product['thumbnail']  # Mobile-optimized image
                } for product in recommendations
            ]
        }

class WebBFF(APIGateway):
    """Web-optimized API gateway"""
    
    async def get_web_dashboard(self, user_id):
        """Full-featured dashboard for web clients"""
        
        # Fetch comprehensive data
        user_data = await self.user_service.get_full_profile(user_id)
        order_history = await self.order_service.get_order_history(user_id)
        detailed_recommendations = await self.product_service.get_detailed_recommendations(user_id)
        analytics = await self.analytics_service.get_user_analytics(user_id)
        
        return {
            'user': user_data,
            'order_history': order_history,
            'recommendations': detailed_recommendations,
            'analytics': analytics,
            'preferences': await self.user_service.get_preferences(user_id)
        }
```

### 2. Aggregator Pattern
```python
class AggregatorGateway:
    """Aggregate data from multiple services"""
    
    def __init__(self):
        self.services = {}
        self.cache = Cache()
    
    async def get_product_details(self, product_id):
        """Aggregate product information from multiple services"""
        
        # Check cache first
        cache_key = f"product_details:{product_id}"
        cached_data = self.cache.get(cache_key)
        if cached_data:
            return cached_data
        
        # Fetch from multiple services
        tasks = {
            'basic_info': self.product_service.get_product(product_id),
            'inventory': self.inventory_service.get_inventory(product_id),
            'reviews': self.review_service.get_reviews(product_id, limit=5),
            'pricing': self.pricing_service.get_current_price(product_id),
            'recommendations': self.recommendation_service.get_similar_products(product_id)
        }
        
        # Execute all requests concurrently
        results = {}
        for key, task in tasks.items():
            try:
                results[key] = await task
            except Exception as e:
                # Handle partial failures gracefully
                results[key] = self.get_fallback_data(key, product_id)
                log.warning(f"Service call failed for {key}: {e}")
        
        # Compose aggregated response
        aggregated_data = {
            'product': results['basic_info'],
            'availability': results['inventory']['available'],
            'price': results['pricing']['current_price'],
            'rating': self.calculate_average_rating(results['reviews']),
            'reviews': results['reviews']['items'],
            'similar_products': results['recommendations']
        }
        
        # Cache the aggregated result
        self.cache.set(cache_key, aggregated_data, ttl=300)
        
        return aggregated_data
```

### 3. Proxy Pattern
```python
class ProxyGateway:
    """Simple proxy to backend services"""
    
    def __init__(self):
        self.service_clients = {}
        self.circuit_breakers = {}
    
    async def proxy_request(self, request):
        """Forward request to backend service"""
        
        # Determine target service
        service_name = self.extract_service_name(request.path)
        
        # Get or create circuit breaker for service
        if service_name not in self.circuit_breakers:
            self.circuit_breakers[service_name] = CircuitBreaker()
        
        circuit_breaker = self.circuit_breakers[service_name]
        
        try:
            # Forward request through circuit breaker
            response = await circuit_breaker.call(
                self.forward_to_service,
                service_name,
                request
            )
            return response
            
        except CircuitBreakerOpenError:
            return self.error_response(503, "Service temporarily unavailable")
    
    async def forward_to_service(self, service_name, request):
        """Forward request to specific service"""
        
        # Get service instance
        instance = self.service_discovery.get_instance(service_name)
        
        # Transform request
        backend_url = f"http://{instance['host']}:{instance['port']}{request.path}"
        
        # Forward request
        async with aiohttp.ClientSession() as session:
            async with session.request(
                method=request.method,
                url=backend_url,
                headers=request.headers,
                data=request.body
            ) as response:
                return await response.json()
```

## Gateway Middleware

### 1. Authentication Middleware
```python
class AuthenticationMiddleware:
    def __init__(self, auth_service):
        self.auth_service = auth_service
        self.public_paths = ['/health', '/login', '/register']
    
    async def process_request(self, request):
        """Authenticate request"""
        
        # Skip authentication for public paths
        if request.path in self.public_paths:
            return request
        
        # Extract token
        auth_header = request.headers.get('Authorization')
        if not auth_header or not auth_header.startswith('Bearer '):
            request.response = self.unauthorized_response()
            return request
        
        token = auth_header[7:]  # Remove 'Bearer '
        
        try:
            # Validate token
            user = await self.auth_service.validate_token(token)
            request.user = user
            return request
            
        except InvalidTokenError:
            request.response = self.unauthorized_response()
            return request
    
    def unauthorized_response(self):
        return {
            'status': 401,
            'body': {'error': 'Unauthorized'},
            'headers': {'Content-Type': 'application/json'}
        }
```

### 2. Rate Limiting Middleware
```python
class RateLimitingMiddleware:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.default_limit = 1000  # requests per hour
        self.user_limits = {}
    
    async def process_request(self, request):
        """Apply rate limiting"""
        
        # Get user identifier
        user_id = getattr(request, 'user', {}).get('id', request.client_ip)
        
        # Get rate limit for user
        rate_limit = self.get_rate_limit(user_id)
        
        # Check current usage
        current_usage = await self.get_current_usage(user_id)
        
        if current_usage >= rate_limit:
            request.response = self.rate_limit_response(rate_limit)
            return request
        
        # Increment usage counter
        await self.increment_usage(user_id)
        
        return request
    
    async def get_current_usage(self, user_id):
        """Get current usage from Redis"""
        key = f"rate_limit:{user_id}:{self.get_current_hour()}"
        usage = await self.redis.get(key)
        return int(usage) if usage else 0
    
    async def increment_usage(self, user_id):
        """Increment usage counter"""
        key = f"rate_limit:{user_id}:{self.get_current_hour()}"
        await self.redis.incr(key)
        await self.redis.expire(key, 3600)  # 1 hour TTL
```

### 3. Caching Middleware
```python
class CachingMiddleware:
    def __init__(self, cache_client):
        self.cache = cache_client
        self.cacheable_methods = ['GET']
        self.cache_rules = {}
    
    def add_cache_rule(self, path_pattern, ttl, vary_headers=None):
        """Add caching rule for specific paths"""
        self.cache_rules[path_pattern] = {
            'ttl': ttl,
            'vary_headers': vary_headers or []
        }
    
    async def process_request(self, request):
        """Check cache before forwarding request"""
        
        if request.method not in self.cacheable_methods:
            return request
        
        # Generate cache key
        cache_key = self.generate_cache_key(request)
        
        # Check cache
        cached_response = await self.cache.get(cache_key)
        if cached_response:
            request.response = cached_response
            request.response['headers']['X-Cache'] = 'HIT'
            return request
        
        return request
    
    async def process_response(self, request, response):
        """Cache response if applicable"""
        
        if request.method not in self.cacheable_methods:
            return response
        
        # Check if response is cacheable
        if response['status'] == 200 and self.is_cacheable(request.path):
            cache_key = self.generate_cache_key(request)
            ttl = self.get_cache_ttl(request.path)
            
            # Cache the response
            await self.cache.set(cache_key, response, ttl=ttl)
            response['headers']['X-Cache'] = 'MISS'
        
        return response
```

## Advanced Gateway Features

### 1. Request Transformation
```python
class RequestTransformationMiddleware:
    def __init__(self):
        self.transformations = {}
    
    def add_transformation(self, path_pattern, transformer):
        """Add request transformation for specific paths"""
        self.transformations[path_pattern] = transformer
    
    async def process_request(self, request):
        """Transform request based on configured rules"""
        
        for pattern, transformer in self.transformations.items():
            if re.match(pattern, request.path):
                request = await transformer.transform(request)
                break
        
        return request

class LegacyAPITransformer:
    """Transform modern API requests to legacy format"""
    
    async def transform(self, request):
        """Transform REST request to SOAP"""
        
        if request.path.startswith('/api/v2/users/'):
            user_id = request.path.split('/')[-1]
            
            # Transform REST to SOAP envelope
            soap_body = f"""
            <soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
                <soap:Body>
                    <GetUser xmlns="http://legacy.example.com/">
                        <UserId>{user_id}</UserId>
                    </GetUser>
                </soap:Body>
            </soap:Envelope>
            """
            
            request.body = soap_body
            request.headers['Content-Type'] = 'text/xml'
            request.headers['SOAPAction'] = 'GetUser'
        
        return request
```

### 2. Response Aggregation
```python
class ResponseAggregationMiddleware:
    def __init__(self, service_clients):
        self.service_clients = service_clients
        self.aggregation_rules = {}
    
    def add_aggregation_rule(self, path_pattern, aggregator):
        """Add response aggregation rule"""
        self.aggregation_rules[path_pattern] = aggregator
    
    async def process_request(self, request):
        """Check if request needs aggregation"""
        
        for pattern, aggregator in self.aggregation_rules.items():
            if re.match(pattern, request.path):
                # Handle aggregation instead of forwarding
                response = await aggregator.aggregate(request)
                request.response = response
                break
        
        return request

class DashboardAggregator:
    def __init__(self, service_clients):
        self.user_service = service_clients['user']
        self.order_service = service_clients['order']
        self.recommendation_service = service_clients['recommendation']
    
    async def aggregate(self, request):
        """Aggregate dashboard data"""
        
        user_id = request.path_params['user_id']
        
        # Parallel service calls
        tasks = {
            'user': self.user_service.get_user(user_id),
            'orders': self.order_service.get_recent_orders(user_id, limit=5),
            'recommendations': self.recommendation_service.get_recommendations(user_id)
        }
        
        results = {}
        for key, task in tasks.items():
            try:
                results[key] = await task
            except Exception as e:
                # Graceful degradation
                results[key] = self.get_fallback_data(key)
                log.warning(f"Service call failed for {key}: {e}")
        
        return {
            'status': 200,
            'body': {
                'user_info': results['user'],
                'recent_activity': results['orders'],
                'suggested_products': results['recommendations']
            }
        }
```

### 3. Protocol Translation
```python
class ProtocolTranslationGateway:
    def __init__(self):
        self.grpc_clients = {}
        self.soap_clients = {}
    
    async def handle_rest_to_grpc(self, request):
        """Translate REST request to gRPC call"""
        
        if request.path.startswith('/api/users/'):
            user_id = request.path.split('/')[-1]
            
            # Create gRPC request
            grpc_request = user_pb2.GetUserRequest(user_id=user_id)
            
            # Make gRPC call
            grpc_client = self.grpc_clients['user_service']
            grpc_response = await grpc_client.GetUser(grpc_request)
            
            # Transform gRPC response to REST JSON
            rest_response = {
                'id': grpc_response.user_id,
                'name': grpc_response.name,
                'email': grpc_response.email,
                'created_at': grpc_response.created_at.ToJsonString()
            }
            
            return {
                'status': 200,
                'body': rest_response,
                'headers': {'Content-Type': 'application/json'}
            }
```

## Gateway Security

### 1. API Key Management
```python
class APIKeyMiddleware:
    def __init__(self, key_store):
        self.key_store = key_store
    
    async def process_request(self, request):
        """Validate API key"""
        
        api_key = request.headers.get('X-API-Key')
        if not api_key:
            request.response = self.error_response(401, "API key required")
            return request
        
        # Validate key
        key_info = await self.key_store.get_key_info(api_key)
        if not key_info or not key_info['active']:
            request.response = self.error_response(401, "Invalid API key")
            return request
        
        # Check key permissions
        if not self.check_permissions(key_info, request.path, request.method):
            request.response = self.error_response(403, "Insufficient permissions")
            return request
        
        # Add key info to request
        request.api_key_info = key_info
        return request
    
    def check_permissions(self, key_info, path, method):
        """Check if API key has permission for this operation"""
        
        permissions = key_info.get('permissions', [])
        
        for permission in permissions:
            if (re.match(permission['path_pattern'], path) and 
                method in permission['methods']):
                return True
        
        return False
```

### 2. OAuth2 Integration
```python
class OAuth2Middleware:
    def __init__(self, oauth_provider):
        self.oauth_provider = oauth_provider
        self.token_cache = Cache()
    
    async def process_request(self, request):
        """Validate OAuth2 token"""
        
        auth_header = request.headers.get('Authorization')
        if not auth_header or not auth_header.startswith('Bearer '):
            request.response = self.unauthorized_response()
            return request
        
        token = auth_header[7:]
        
        # Check token cache
        cached_user = await self.token_cache.get(f"token:{token}")
        if cached_user:
            request.user = cached_user
            return request
        
        try:
            # Validate token with OAuth provider
            user_info = await self.oauth_provider.validate_token(token)
            
            # Cache user info
            await self.token_cache.set(
                f"token:{token}", 
                user_info, 
                ttl=300  # 5 minutes
            )
            
            request.user = user_info
            return request
            
        except InvalidTokenError:
            request.response = self.unauthorized_response()
            return request
```

### 3. Input Validation
```python
class InputValidationMiddleware:
    def __init__(self):
        self.schemas = {}
    
    def add_schema(self, path_pattern, method, schema):
        """Add validation schema for endpoint"""
        key = f"{method}:{path_pattern}"
        self.schemas[key] = schema
    
    async def process_request(self, request):
        """Validate request against schema"""
        
        # Find matching schema
        schema = self.find_schema(request.path, request.method)
        if not schema:
            return request
        
        try:
            # Validate request body
            if request.body:
                jsonschema.validate(request.body, schema['body'])
            
            # Validate query parameters
            if request.query_params:
                jsonschema.validate(request.query_params, schema['query'])
            
            # Validate path parameters
            if request.path_params:
                jsonschema.validate(request.path_params, schema['path'])
            
            return request
            
        except ValidationError as e:
            request.response = self.validation_error_response(str(e))
            return request
```

## Gateway Performance Optimization

### 1. Connection Pooling
```python
class ConnectionPoolManager:
    def __init__(self):
        self.pools = {}
    
    def get_pool(self, service_name):
        """Get connection pool for service"""
        
        if service_name not in self.pools:
            service_config = self.get_service_config(service_name)
            
            self.pools[service_name] = aiohttp.TCPConnector(
                limit=100,  # Total connections
                limit_per_host=20,  # Per host limit
                keepalive_timeout=30,
                enable_cleanup_closed=True
            )
        
        return self.pools[service_name]
    
    async def make_request(self, service_name, method, path, **kwargs):
        """Make request using connection pool"""
        
        pool = self.get_pool(service_name)
        
        async with aiohttp.ClientSession(connector=pool) as session:
            async with session.request(method, path, **kwargs) as response:
                return await response.json()
```

### 2. Request Batching
```python
class RequestBatchingMiddleware:
    def __init__(self, batch_size=10, batch_timeout=100):
        self.batch_size = batch_size
        self.batch_timeout = batch_timeout  # milliseconds
        self.pending_requests = {}
    
    async def process_request(self, request):
        """Batch similar requests together"""
        
        # Check if request can be batched
        batch_key = self.get_batch_key(request)
        if not batch_key:
            return request
        
        # Add to pending batch
        if batch_key not in self.pending_requests:
            self.pending_requests[batch_key] = {
                'requests': [],
                'created_at': time.time()
            }
        
        batch = self.pending_requests[batch_key]
        batch['requests'].append(request)
        
        # Check if batch is ready
        if (len(batch['requests']) >= self.batch_size or
            (time.time() - batch['created_at']) * 1000 >= self.batch_timeout):
            
            # Process batch
            responses = await self.process_batch(batch_key, batch['requests'])
            
            # Set responses for all requests
            for req, resp in zip(batch['requests'], responses):
                req.response = resp
            
            # Clear batch
            del self.pending_requests[batch_key]
        
        return request
    
    def get_batch_key(self, request):
        """Generate batch key for similar requests"""
        
        if request.path.startswith('/api/users/') and request.method == 'GET':
            return 'get_users'
        elif request.path.startswith('/api/products/') and request.method == 'GET':
            return 'get_products'
        
        return None
    
    async def process_batch(self, batch_type, requests):
        """Process batch of similar requests"""
        
        if batch_type == 'get_users':
            user_ids = [req.path.split('/')[-1] for req in requests]
            users = await self.user_service.get_users_batch(user_ids)
            
            return [
                {'status': 200, 'body': user}
                for user in users
            ]
```

## Gateway Deployment Patterns

### 1. Single Gateway
```yaml
# kubernetes deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
      - name: gateway
        image: api-gateway:v1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: SERVICE_DISCOVERY_URL
          value: "http://consul:8500"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

### 2. Multiple Gateway Deployment
```python
class GatewayCluster:
    """Deploy multiple gateways for different purposes"""
    
    def __init__(self):
        self.public_gateway = PublicAPIGateway()    # External clients
        self.internal_gateway = InternalAPIGateway() # Service-to-service
        self.admin_gateway = AdminAPIGateway()       # Admin operations
    
    def route_by_client_type(self, request):
        """Route to appropriate gateway"""
        
        client_type = self.identify_client_type(request)
        
        if client_type == 'public':
            return self.public_gateway.handle_request(request)
        elif client_type == 'internal':
            return self.internal_gateway.handle_request(request)
        elif client_type == 'admin':
            return self.admin_gateway.handle_request(request)
        else:
            return self.error_response(400, "Unknown client type")
```

## Gateway Monitoring

### 1. Metrics Collection
```python
class GatewayMetrics:
    def __init__(self):
        self.request_counter = Counter('gateway_requests_total', 
                                     ['method', 'path', 'status'])
        self.request_duration = Histogram('gateway_request_duration_seconds',
                                        ['method', 'path'])
        self.active_connections = Gauge('gateway_active_connections')
        self.service_health = Gauge('gateway_backend_service_health',
                                  ['service_name'])
    
    def record_request(self, method, path, status_code, duration):
        """Record request metrics"""
        
        self.request_counter.labels(
            method=method,
            path=self.normalize_path(path),
            status=status_code
        ).inc()
        
        self.request_duration.labels(
            method=method,
            path=self.normalize_path(path)
        ).observe(duration)
    
    def update_service_health(self, service_name, is_healthy):
        """Update backend service health metric"""
        
        health_value = 1 if is_healthy else 0
        self.service_health.labels(service_name=service_name).set(health_value)
```

### 2. Distributed Tracing
```python
class GatewayTracing:
    def __init__(self, tracer):
        self.tracer = tracer
    
    async def trace_request(self, request):
        """Add tracing to gateway request"""
        
        # Start new trace or continue existing
        trace_context = self.extract_trace_context(request.headers)
        
        with self.tracer.start_span('api_gateway_request', child_of=trace_context) as span:
            # Add span tags
            span.set_tag('http.method', request.method)
            span.set_tag('http.url', request.path)
            span.set_tag('component', 'api_gateway')
            
            # Add correlation ID to headers
            correlation_id = str(uuid.uuid4())
            request.headers['X-Correlation-ID'] = correlation_id
            span.set_tag('correlation_id', correlation_id)
            
            try:
                # Process request
                response = await self.process_request(request)
                
                span.set_tag('http.status_code', response['status'])
                return response
                
            except Exception as e:
                span.set_tag('error', True)
                span.set_tag('error.message', str(e))
                raise e
```

## Exercise Problems

1. Design an API gateway for a microservices-based e-commerce platform
2. How would you implement rate limiting across multiple gateway instances?
3. Design a BFF pattern for mobile and web clients of a social media app
4. Implement request/response transformation for legacy service integration

## Key Takeaways

- API gateways centralize cross-cutting concerns
- Choose between single gateway vs multiple specialized gateways
- Implement circuit breakers to prevent cascade failures
- Monitor gateway performance and backend service health
- Consider protocol translation for legacy system integration
- BFF pattern optimizes for different client types
- Security should be implemented at the gateway level
- Plan for gateway scalability and high availability

## Next Steps

Complete Module 2 and move to: **Module 3: Design Patterns & Principles**