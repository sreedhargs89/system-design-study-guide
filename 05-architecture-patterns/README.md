# Architecture Patterns 🏛️

Understanding different architectural approaches and their trade-offs for building scalable, maintainable systems.

## Table of Contents

- [Monolithic Architecture](#monolithic-architecture)
- [Microservices Architecture](#microservices-architecture)
- [Event-Driven Architecture](#event-driven-architecture)
- [API Gateway Pattern](#api-gateway-pattern)
- [Backend for Frontend (BFF)](#backend-for-frontend-bff)
- [Service Mesh](#service-mesh)
- [Serverless Architecture](#serverless-architecture)
- [Interview Questions](#interview-questions)

---

## Monolithic Architecture

### What is a Monolith?

A monolithic application is deployed as a single unit where all components are interconnected and interdependent.

```
┌─────────────────────────────────────────┐
│            Monolithic App               │
├─────────────────────────────────────────┤
│  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │   UI    │  │Business │  │ Data    │ │
│  │ Layer   │  │ Logic   │  │ Layer   │ │
│  └─────────┘  └─────────┘  └─────────┘ │
└─────────────────────────────────────────┘
                    │
                    ▼
            ┌─────────────────┐
            │    Database     │
            └─────────────────┘
```

### Advantages of Monoliths

#### 1. Simplicity
```python
# Single codebase example
class ECommerceApp:
    def __init__(self):
        self.user_service = UserService()
        self.product_service = ProductService()
        self.order_service = OrderService()
        self.payment_service = PaymentService()
    
    def process_order(self, user_id, product_id, payment_info):
        # Direct method calls - no network overhead
        user = self.user_service.get_user(user_id)
        product = self.product_service.get_product(product_id)
        
        # ACID transactions across all services
        with database.transaction():
            order = self.order_service.create_order(user, product)
            payment = self.payment_service.process_payment(payment_info)
            
        return order
```

#### 2. Performance Benefits
- **No Network Latency**: Direct method calls
- **Single Database**: ACID transactions
- **Shared Memory**: Efficient data sharing

#### 3. Development & Testing
- **Easy Debugging**: Single process to debug
- **Unified Testing**: Test entire system together
- **Simple Deployment**: Single artifact to deploy

### Disadvantages of Monoliths

#### 1. Scaling Challenges
```
Problem: Entire app must be scaled together
┌─────────────────────┐  ┌─────────────────────┐
│   Monolith v1       │  │   Monolith v2       │
│ (Needs more CPU     │  │ (Needs more memory  │
│  for orders)        │  │  for user queries)  │
└─────────────────────┘  └─────────────────────┘
         │                         │
         └─────────┬─────────────────┘
                   │
        ┌─────────────────────┐
        │ Must scale entire   │
        │ application even    │
        │ if only one part    │
        │ needs more resources│
        └─────────────────────┘
```

#### 2. Technology Lock-in
```python
# Entire application stuck with same technology
class MonolithicApp:
    # All components must use same:
    # - Programming language (e.g., Java)
    # - Framework (e.g., Spring)
    # - Database (e.g., PostgreSQL)
    # - Runtime (e.g., JVM)
    
    def __init__(self):
        # Can't use Python for ML, Go for performance, etc.
        pass
```

#### 3. Team Coordination
- **Code Conflicts**: Multiple teams editing same codebase
- **Release Coordination**: All teams must coordinate deployments
- **Blast Radius**: One team's bug can bring down entire system

### When to Use Monoliths

#### Ideal Scenarios
1. **Small Teams**: < 10 developers
2. **Simple Applications**: Limited complexity
3. **Rapid Prototyping**: Fast initial development
4. **Strong Consistency**: ACID transactions required
5. **Limited Scale**: < 1000 concurrent users

#### Example: Small E-commerce Site
```python
class SimpleECommerce:
    """
    Perfect for a small business with:
    - < 1000 products
    - < 100 orders/day
    - Small development team
    - Simple workflows
    """
    def __init__(self):
        self.db = Database('sqlite:///ecommerce.db')
    
    def get_products(self):
        return self.db.query("SELECT * FROM products")
    
    def create_order(self, user_id, items):
        with self.db.transaction():
            order_id = self.db.insert_order(user_id)
            for item in items:
                self.db.insert_order_item(order_id, item)
            self.update_inventory(items)
            return order_id
```

---

## Microservices Architecture

### What are Microservices?

Microservices break down applications into small, independent services that communicate over well-defined APIs.

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│    User     │    │   Product   │    │   Order     │
│   Service   │    │   Service   │    │   Service   │
│     │       │    │     │       │    │     │       │
│   ┌─────┐   │    │   ┌─────┐   │    │   ┌─────┐   │
│   │ DB  │   │    │   │ DB  │   │    │   │ DB  │   │
│   └─────┘   │    │   └─────┘   │    │   └─────┘   │
└─────────────┘    └─────────────┘    └─────────────┘
       │                   │                   │
       └─────────┬─────────┴─────────┬─────────┘
                 │                   │
          ┌─────────────┐    ┌─────────────┐
          │   Payment   │    │ Inventory   │
          │   Service   │    │   Service   │
          └─────────────┘    └─────────────┘
```

### Core Principles

#### 1. Single Responsibility
```python
# Each service has one business purpose
class UserService:
    """Handles only user-related operations"""
    def create_user(self, user_data): pass
    def authenticate(self, credentials): pass
    def get_user_profile(self, user_id): pass

class ProductService:
    """Handles only product catalog operations"""
    def add_product(self, product_data): pass
    def search_products(self, query): pass
    def get_product_details(self, product_id): pass

class OrderService:
    """Handles only order processing"""
    def create_order(self, order_data): pass
    def get_order_status(self, order_id): pass
    def cancel_order(self, order_id): pass
```

#### 2. Decentralized Governance
```
Each team owns their service:
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   Team Alpha    │  │   Team Beta     │  │   Team Gamma    │
├─────────────────┤  ├─────────────────┤  ├─────────────────┤
│ • Choose tech   │  │ • Own deployment│  │ • Define APIs   │
│ • Own data      │  │ • Set SLA       │  │ • Monitor       │
│ • Deploy        │  │ • Scale         │  │ • Operate       │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

#### 3. Communication via APIs
```python
# Inter-service communication example
class OrderService:
    def __init__(self):
        self.user_service = UserServiceClient()
        self.product_service = ProductServiceClient()
        self.payment_service = PaymentServiceClient()
    
    async def create_order(self, user_id, items, payment_info):
        # Call other services via HTTP/gRPC
        user = await self.user_service.get_user(user_id)
        
        # Validate products
        for item in items:
            product = await self.product_service.get_product(item.id)
            if not product.available:
                raise ProductNotAvailableError()
        
        # Process payment
        payment = await self.payment_service.charge(payment_info)
        
        # Create order
        order = Order(user_id=user_id, items=items, payment_id=payment.id)
        return await self.save_order(order)
```

### Advantages of Microservices

#### 1. Independent Scaling
```python
# Scale services based on demand
scaling_config = {
    'user-service': {'min_instances': 2, 'max_instances': 5},
    'product-service': {'min_instances': 3, 'max_instances': 10},  # High read load
    'order-service': {'min_instances': 5, 'max_instances': 20},   # High transaction volume
    'payment-service': {'min_instances': 3, 'max_instances': 8}
}
```

#### 2. Technology Diversity
```
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  User Service   │  │ Product Service │  │ Order Service   │
│                 │  │                 │  │                 │
│ • Node.js       │  │ • Python        │  │ • Java          │
│ • MongoDB       │  │ • Elasticsearch │  │ • PostgreSQL    │
│ • Redis Cache   │  │ • Redis Cache   │  │ • RabbitMQ      │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

#### 3. Fault Isolation
```python
# Circuit breaker pattern
class ServiceClient:
    def __init__(self, service_url):
        self.service_url = service_url
        self.circuit_breaker = CircuitBreaker(
            failure_threshold=5,
            recovery_timeout=60,
            expected_exception=ServiceError
        )
    
    @circuit_breaker
    def call_service(self, endpoint, data):
        response = requests.post(f"{self.service_url}/{endpoint}", json=data)
        if response.status_code != 200:
            raise ServiceError("Service call failed")
        return response.json()

# If product service fails, order service can still function with cached data
class OrderService:
    def create_order(self, user_id, items):
        try:
            products = self.product_service.get_products(items)
        except ServiceError:
            # Fallback to cached product data
            products = self.cache.get_products(items)
        
        return self.process_order(user_id, products)
```

### Disadvantages of Microservices

#### 1. Distributed System Complexity
```python
# Network failures, latency, partial failures
class DistributedOrderProcessor:
    async def create_order(self, user_id, items, payment_info):
        try:
            # Multiple network calls - each can fail
            user = await self.user_service.get_user(user_id)      # Network call 1
            products = await self.product_service.validate(items) # Network call 2  
            inventory = await self.inventory_service.reserve(items) # Network call 3
            payment = await self.payment_service.charge(payment_info) # Network call 4
            
            # What if payment succeeds but inventory reservation fails?
            # Need distributed transaction handling (Saga pattern)
            
        except NetworkError as e:
            # Handle partial failures, retries, timeouts
            await self.handle_failure(e)
```

#### 2. Data Consistency Challenges
```python
# No distributed ACID transactions
class OrderSaga:
    """Saga pattern for distributed transactions"""
    
    def __init__(self):
        self.steps = []
        self.compensations = []
    
    async def execute_order(self, order_data):
        try:
            # Step 1: Reserve inventory
            reservation = await self.inventory_service.reserve(order_data.items)
            self.steps.append(('inventory_reserved', reservation.id))
            self.compensations.append(('release_inventory', reservation.id))
            
            # Step 2: Charge payment
            payment = await self.payment_service.charge(order_data.payment)
            self.steps.append(('payment_charged', payment.id))
            self.compensations.append(('refund_payment', payment.id))
            
            # Step 3: Create order
            order = await self.order_service.create(order_data)
            self.steps.append(('order_created', order.id))
            
            return order
            
        except Exception as e:
            # Compensate for completed steps
            await self.compensate()
            raise
    
    async def compensate(self):
        # Execute compensation actions in reverse order
        for action, resource_id in reversed(self.compensations):
            await self.execute_compensation(action, resource_id)
```

#### 3. Operational Overhead
```yaml
# Multiple services to monitor, deploy, and maintain
services:
  - name: user-service
    monitoring: [cpu, memory, errors, latency]
    logging: centralized
    deployment: blue-green
    
  - name: product-service
    monitoring: [cpu, memory, errors, latency, search_index]
    logging: centralized
    deployment: canary
    
  - name: order-service
    monitoring: [cpu, memory, errors, latency, queue_depth]
    logging: centralized
    deployment: rolling
```

### When to Use Microservices

#### Ideal Scenarios
1. **Large Teams**: > 50 developers
2. **Complex Domain**: Multiple business areas
3. **High Scale**: > 10,000 concurrent users
4. **Different Requirements**: Services need different scaling/tech
5. **DevOps Maturity**: Strong CI/CD and monitoring

### Migration Strategy: Strangler Fig Pattern

```python
# Gradually extract services from monolith
class OrderMigration:
    """
    Phase 1: Extract order service but keep shared database
    Phase 2: Separate order database
    Phase 3: Remove order code from monolith
    """
    
    def __init__(self):
        self.legacy_monolith = LegacyMonolith()
        self.new_order_service = OrderService()
        self.feature_flag = FeatureFlag('use_new_order_service')
    
    def create_order(self, order_data):
        if self.feature_flag.enabled():
            return self.new_order_service.create_order(order_data)
        else:
            return self.legacy_monolith.create_order(order_data)
```

---

## Event-Driven Architecture

### Core Concepts

Event-driven architecture uses events to trigger and communicate between decoupled services.

```
┌─────────────┐    Event    ┌─────────────┐    Event    ┌─────────────┐
│   Producer  │──────────►  │Event Bus/   │──────────►  │  Consumer   │
│             │             │Message Queue│             │             │
└─────────────┘             └─────────────┘             └─────────────┘
```

### Event Types

#### 1. Domain Events
```python
# Events represent business occurrences
class DomainEvent:
    def __init__(self, event_type, aggregate_id, data, timestamp=None):
        self.event_type = event_type
        self.aggregate_id = aggregate_id
        self.data = data
        self.timestamp = timestamp or datetime.utcnow()

# Business events
class UserRegistered(DomainEvent):
    def __init__(self, user_id, email, name):
        super().__init__(
            event_type="user.registered",
            aggregate_id=user_id,
            data={"email": email, "name": name}
        )

class OrderPlaced(DomainEvent):
    def __init__(self, order_id, user_id, items, total):
        super().__init__(
            event_type="order.placed",
            aggregate_id=order_id,
            data={"user_id": user_id, "items": items, "total": total}
        )
```

#### 2. Integration Events
```python
# Events for cross-service communication
class IntegrationEvent:
    def __init__(self, event_type, correlation_id, data):
        self.event_type = event_type
        self.correlation_id = correlation_id
        self.data = data
        self.event_id = str(uuid.uuid4())
        self.timestamp = datetime.utcnow()

# Service integration events
class PaymentProcessed(IntegrationEvent):
    def __init__(self, payment_id, order_id, amount, status):
        super().__init__(
            event_type="payment.processed",
            correlation_id=order_id,
            data={"payment_id": payment_id, "amount": amount, "status": status}
        )
```

### Event Bus Implementation

#### 1. Simple In-Process Event Bus
```python
class EventBus:
    def __init__(self):
        self._handlers = defaultdict(list)
    
    def subscribe(self, event_type, handler):
        self._handlers[event_type].append(handler)
    
    def publish(self, event):
        handlers = self._handlers[event.event_type]
        for handler in handlers:
            try:
                handler.handle(event)
            except Exception as e:
                logger.error(f"Error handling event {event.event_type}: {e}")

# Usage
event_bus = EventBus()

# Register handlers
event_bus.subscribe("user.registered", SendWelcomeEmailHandler())
event_bus.subscribe("user.registered", CreateUserProfileHandler())
event_bus.subscribe("user.registered", SendAnalyticsEventHandler())

# Publish event
user_registered = UserRegistered(user_id=123, email="user@example.com", name="John Doe")
event_bus.publish(user_registered)
```

#### 2. Distributed Event Bus with Message Queues
```python
import pika  # RabbitMQ client

class DistributedEventBus:
    def __init__(self, rabbitmq_url):
        self.connection = pika.BlockingConnection(pika.URLParameters(rabbitmq_url))
        self.channel = self.connection.channel()
        
        # Declare exchange for events
        self.channel.exchange_declare(
            exchange='domain_events',
            exchange_type='topic',
            durable=True
        )
    
    def publish(self, event):
        message = json.dumps({
            'event_type': event.event_type,
            'aggregate_id': event.aggregate_id,
            'data': event.data,
            'timestamp': event.timestamp.isoformat()
        })
        
        self.channel.basic_publish(
            exchange='domain_events',
            routing_key=event.event_type,
            body=message,
            properties=pika.BasicProperties(
                delivery_mode=2,  # Persistent message
            )
        )
    
    def subscribe(self, event_pattern, handler):
        # Create queue for this handler
        queue_name = f"{handler.__class__.__name__}_{uuid.uuid4()}"
        self.channel.queue_declare(queue=queue_name, durable=True)
        self.channel.queue_bind(
            exchange='domain_events',
            queue=queue_name,
            routing_key=event_pattern
        )
        
        def callback(ch, method, properties, body):
            try:
                event_data = json.loads(body)
                handler.handle(event_data)
                ch.basic_ack(delivery_tag=method.delivery_tag)
            except Exception as e:
                logger.error(f"Failed to handle event: {e}")
                ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)
        
        self.channel.basic_consume(queue=queue_name, on_message_callback=callback)
```

### Event Sourcing Pattern

```python
class EventStore:
    def __init__(self):
        self.events = []  # In real implementation, use database
    
    def append_events(self, aggregate_id, events, expected_version):
        # Optimistic concurrency control
        current_version = self.get_current_version(aggregate_id)
        if current_version != expected_version:
            raise ConcurrencyError("Aggregate has been modified")
        
        for event in events:
            event.version = current_version + 1
            event.aggregate_id = aggregate_id
            self.events.append(event)
            current_version += 1
    
    def get_events(self, aggregate_id, from_version=0):
        return [e for e in self.events 
                if e.aggregate_id == aggregate_id and e.version > from_version]

class OrderAggregate:
    def __init__(self, order_id):
        self.order_id = order_id
        self.uncommitted_events = []
        self.version = 0
        self.items = []
        self.status = "draft"
    
    def add_item(self, product_id, quantity, price):
        if self.status != "draft":
            raise InvalidOperationError("Cannot modify confirmed order")
        
        event = ItemAdded(product_id, quantity, price)
        self.apply_event(event)
        self.uncommitted_events.append(event)
    
    def confirm_order(self):
        if not self.items:
            raise InvalidOperationError("Cannot confirm empty order")
        
        event = OrderConfirmed()
        self.apply_event(event)
        self.uncommitted_events.append(event)
    
    def apply_event(self, event):
        if isinstance(event, ItemAdded):
            self.items.append({
                'product_id': event.product_id,
                'quantity': event.quantity,
                'price': event.price
            })
        elif isinstance(event, OrderConfirmed):
            self.status = "confirmed"
        
        self.version += 1

# Rebuild aggregate from events
def load_order(order_id, event_store):
    order = OrderAggregate(order_id)
    events = event_store.get_events(order_id)
    
    for event in events:
        order.apply_event(event)
    
    return order
```

### Benefits of Event-Driven Architecture

#### 1. Loose Coupling
```python
# Publisher doesn't know about consumers
class OrderService:
    def place_order(self, order_data):
        # Process order
        order = self.create_order(order_data)
        
        # Publish event - don't care who listens
        self.event_bus.publish(OrderPlaced(
            order_id=order.id,
            user_id=order.user_id,
            items=order.items,
            total=order.total
        ))
        
        return order

# Multiple consumers can react independently
class EmailService:
    def handle(self, event):
        if isinstance(event, OrderPlaced):
            self.send_order_confirmation(event.data['user_id'], event.aggregate_id)

class InventoryService:
    def handle(self, event):
        if isinstance(event, OrderPlaced):
            self.reserve_inventory(event.data['items'])

class AnalyticsService:
    def handle(self, event):
        if isinstance(event, OrderPlaced):
            self.track_sale(event.data['total'], event.data['items'])
```

#### 2. Scalability
```python
# Events can be processed asynchronously
class AsyncEventHandler:
    def __init__(self, queue_size=1000):
        self.event_queue = asyncio.Queue(maxsize=queue_size)
        self.workers = []
    
    async def start_workers(self, worker_count=5):
        for i in range(worker_count):
            worker = asyncio.create_task(self.worker(f"worker-{i}"))
            self.workers.append(worker)
    
    async def worker(self, name):
        while True:
            try:
                event = await self.event_queue.get()
                await self.process_event(event)
                self.event_queue.task_done()
            except Exception as e:
                logger.error(f"Worker {name} error: {e}")
    
    async def handle_event(self, event):
        await self.event_queue.put(event)
```

### Challenges and Solutions

#### 1. Event Ordering
```python
# Ensure events are processed in order per aggregate
class OrderedEventProcessor:
    def __init__(self):
        self.aggregate_locks = {}
    
    async def process_event(self, event):
        aggregate_id = event.aggregate_id
        
        # Acquire lock for this aggregate
        if aggregate_id not in self.aggregate_locks:
            self.aggregate_locks[aggregate_id] = asyncio.Lock()
        
        async with self.aggregate_locks[aggregate_id]:
            # Process events for this aggregate sequentially
            await self.handle_event(event)
```

#### 2. Event Versioning
```python
# Handle event schema evolution
class EventHandler:
    def handle(self, event):
        handler_method = getattr(self, f"handle_{event.event_type}_v{event.version}", None)
        if handler_method:
            return handler_method(event)
        
        # Fallback to latest version handler
        return self.handle_latest(event)
    
    def handle_user_registered_v1(self, event):
        # Handle old event format
        user_id = event.data['user_id']
        email = event.data['email']
        # name field didn't exist in v1
        name = "Unknown"
        self.create_user_profile(user_id, email, name)
    
    def handle_user_registered_v2(self, event):
        # Handle new event format
        user_id = event.data['user_id']
        email = event.data['email']
        name = event.data['name']
        self.create_user_profile(user_id, email, name)
```

---

## API Gateway Pattern

### What is an API Gateway?

An API Gateway is a server that acts as a single entry point for multiple microservices, handling routing, authentication, rate limiting, and more.

```
┌─────────────┐
│   Clients   │
│ (Web, Mobile│
│  Partners)  │
└─────┬───────┘
      │
      ▼
┌─────────────┐
│ API Gateway │ ◄── Single Entry Point
└─────┬───────┘
      │
      ├─────► User Service
      ├─────► Product Service  
      ├─────► Order Service
      ├─────► Payment Service
      └─────► Notification Service
```

### Core Responsibilities

#### 1. Request Routing
```python
class APIGateway:
    def __init__(self):
        self.routes = {
            '/api/v1/users': 'user-service:8080',
            '/api/v1/products': 'product-service:8081',
            '/api/v1/orders': 'order-service:8082',
            '/api/v1/payments': 'payment-service:8083'
        }
        
        self.load_balancer = LoadBalancer()
    
    async def handle_request(self, request):
        # Extract route
        path = request.path
        service_route = self.match_route(path)
        
        if not service_route:
            return Response(status=404, body="Route not found")
        
        # Get healthy service instance
        service_url = await self.load_balancer.get_instance(service_route)
        
        # Forward request
        return await self.forward_request(request, service_url)
    
    def match_route(self, path):
        for route_pattern, service in self.routes.items():
            if path.startswith(route_pattern):
                return service
        return None
    
    async def forward_request(self, request, service_url):
        async with httpx.AsyncClient() as client:
            response = await client.request(
                method=request.method,
                url=f"http://{service_url}{request.path}",
                headers=request.headers,
                content=request.body
            )
            return response
```

#### 2. Authentication & Authorization
```python
class AuthenticationMiddleware:
    def __init__(self, jwt_secret):
        self.jwt_secret = jwt_secret
        self.public_routes = ['/api/v1/auth/login', '/api/v1/health']
    
    async def authenticate(self, request):
        if request.path in self.public_routes:
            return True
        
        # Extract JWT token
        auth_header = request.headers.get('Authorization', '')
        if not auth_header.startswith('Bearer '):
            raise AuthenticationError("Missing or invalid authentication token")
        
        token = auth_header[7:]  # Remove 'Bearer '
        
        try:
            # Verify and decode JWT
            payload = jwt.decode(token, self.jwt_secret, algorithms=['HS256'])
            
            # Add user context to request
            request.user = {
                'user_id': payload['user_id'],
                'roles': payload.get('roles', []),
                'permissions': payload.get('permissions', [])
            }
            
            return True
            
        except jwt.ExpiredSignatureError:
            raise AuthenticationError("Token has expired")
        except jwt.InvalidTokenError:
            raise AuthenticationError("Invalid token")

class AuthorizationMiddleware:
    def __init__(self):
        self.role_permissions = {
            'admin': ['read', 'write', 'delete'],
            'user': ['read', 'write'],
            'guest': ['read']
        }
    
    async def authorize(self, request, required_permission):
        if not hasattr(request, 'user'):
            raise AuthorizationError("User not authenticated")
        
        user_roles = request.user.get('roles', [])
        user_permissions = set()
        
        # Collect permissions from roles
        for role in user_roles:
            user_permissions.update(self.role_permissions.get(role, []))
        
        if required_permission not in user_permissions:
            raise AuthorizationError(f"Insufficient permissions for {required_permission}")
        
        return True
```

#### 3. Rate Limiting
```python
class RateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client
    
    async def check_rate_limit(self, client_id, endpoint, limit=100, window=3600):
        """
        Token bucket algorithm implementation
        limit: requests per window
        window: time window in seconds
        """
        key = f"rate_limit:{client_id}:{endpoint}"
        current_time = int(time.time())
        window_start = current_time - window
        
        # Remove old entries
        await self.redis.zremrangebyscore(key, 0, window_start)
        
        # Count current requests
        current_requests = await self.redis.zcard(key)
        
        if current_requests >= limit:
            raise RateLimitExceededError(f"Rate limit exceeded for {endpoint}")
        
        # Add current request
        await self.redis.zadd(key, {str(current_time): current_time})
        await self.redis.expire(key, window)
        
        return True

# Usage in API Gateway
class APIGateway:
    async def handle_request(self, request):
        # Rate limiting
        client_id = self.get_client_id(request)
        await self.rate_limiter.check_rate_limit(client_id, request.path)
        
        # Authentication
        await self.auth_middleware.authenticate(request)
        
        # Route to service
        return await self.route_request(request)
```

#### 4. Request/Response Transformation
```python
class RequestTransformer:
    def transform_request(self, request, target_service):
        """Transform request format for different services"""
        
        if target_service == 'legacy-service':
            # Convert REST to SOAP for legacy service
            return self.rest_to_soap(request)
        elif target_service == 'external-api':
            # Add API key for external service
            request.headers['X-API-Key'] = self.get_external_api_key()
        
        return request
    
    def transform_response(self, response, client_type):
        """Transform response format for different clients"""
        
        if client_type == 'mobile':
            # Minimize response for mobile clients
            return self.minimize_response(response)
        elif client_type == 'web':
            # Include additional metadata for web clients
            return self.enrich_response(response)
        
        return response
    
    def rest_to_soap(self, request):
        # Convert REST request to SOAP format
        soap_body = f"""
        <soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
            <soap:Body>
                <{request.method.lower()}>
                    {self.json_to_xml(request.json)}
                </{request.method.lower()}>
            </soap:Body>
        </soap:Envelope>
        """
        
        return Request(
            method='POST',
            path=request.path,
            headers={'Content-Type': 'text/xml'},
            body=soap_body
        )
```

### Advanced Gateway Features

#### 1. Circuit Breaker
```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=60):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
    
    async def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = "HALF_OPEN"
            else:
                raise CircuitBreakerOpenError("Service unavailable")
        
        try:
            result = await func(*args, **kwargs)
            
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
                self.failure_count = 0
            
            return result
            
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            
            if self.failure_count >= self.failure_threshold:
                self.state = "OPEN"
            
            raise
```

#### 2. Load Balancing Strategies
```python
class LoadBalancer:
    def __init__(self):
        self.services = {}
        self.current_index = {}
    
    def add_service(self, service_name, instances):
        self.services[service_name] = instances
        self.current_index[service_name] = 0
    
    def round_robin(self, service_name):
        instances = self.services[service_name]
        index = self.current_index[service_name]
        
        instance = instances[index]
        self.current_index[service_name] = (index + 1) % len(instances)
        
        return instance
    
    def weighted_round_robin(self, service_name):
        instances = self.services[service_name]
        
        # Select based on weights
        total_weight = sum(instance.weight for instance in instances)
        random_weight = random.randint(1, total_weight)
        
        current_weight = 0
        for instance in instances:
            current_weight += instance.weight
            if random_weight <= current_weight:
                return instance
    
    async def least_connections(self, service_name):
        instances = self.services[service_name]
        
        # Get instance with least active connections
        min_connections = float('inf')
        selected_instance = None
        
        for instance in instances:
            connections = await self.get_active_connections(instance)
            if connections < min_connections:
                min_connections = connections
                selected_instance = instance
        
        return selected_instance
```

### Benefits and Trade-offs

#### Benefits
1. **Single Entry Point**: Simplified client integration
2. **Cross-Cutting Concerns**: Centralized auth, logging, monitoring
3. **Protocol Translation**: Handle different protocols/formats
4. **Service Discovery**: Abstract service locations from clients

#### Trade-offs
1. **Single Point of Failure**: Gateway becomes critical component
2. **Performance Bottleneck**: Additional network hop
3. **Complexity**: Gateway logic can become complex
4. **Development Bottleneck**: Changes require gateway updates

---

## Backend for Frontend (BFF)

### The Problem BFF Solves

Different clients (mobile, web, IoT) have different data requirements and constraints.

```
Without BFF:
┌─────────┐ ┌─────────┐ ┌─────────┐
│  Web    │ │ Mobile  │ │   IoT   │
│ Client  │ │ Client  │ │ Device  │
└────┬────┘ └────┬────┘ └────┬────┘
     │           │           │
     └───────────┼───────────┘
                 │
          ┌─────────────┐
          │ Monolithic  │
          │   Backend   │
          └─────────────┘
          
With BFF:
┌─────────┐ ┌─────────┐ ┌─────────┐
│  Web    │ │ Mobile  │ │   IoT   │
│ Client  │ │ Client  │ │ Device  │
└────┬────┘ └────┬────┘ └────┬────┘
     │           │           │
     ▼           ▼           ▼
┌─────────┐ ┌─────────┐ ┌─────────┐
│ Web BFF │ │Mobile   │ │ IoT BFF │
│         │ │ BFF     │ │         │
└────┬────┘ └────┬────┘ └────┬────┘
     │           │           │
     └───────────┼───────────┘
                 │
      ┌─────────────────────┐
      │   Shared Services   │
      │  (User, Product,    │
      │   Order, Payment)   │
      └─────────────────────┘
```

### BFF Implementation Examples

#### 1. Mobile BFF (Optimized for Bandwidth)
```python
class MobileBFF:
    def __init__(self):
        self.user_service = UserServiceClient()
        self.product_service = ProductServiceClient()
        self.order_service = OrderServiceClient()
    
    async def get_user_dashboard(self, user_id):
        """Optimized dashboard for mobile - minimal data"""
        
        # Parallel calls to services
        user_task = self.user_service.get_user(user_id)
        recent_orders_task = self.order_service.get_recent_orders(user_id, limit=3)
        
        user, recent_orders = await asyncio.gather(user_task, recent_orders_task)
        
        # Return minimal mobile-optimized response
        return {
            "user": {
                "name": user.name,
                "avatar_url": user.avatar_thumbnail,  # Smaller image for mobile
            },
            "recent_orders": [
                {
                    "id": order.id,
                    "status": order.status,
                    "total": order.total,
                    "date": order.created_at.strftime("%m/%d")  # Short date format
                }
                for order in recent_orders
            ],
            "notifications_count": user.unread_notifications_count
        }
    
    async def get_product_list(self, category_id, page=1):
        """Mobile product list - smaller images, essential info only"""
        
        products = await self.product_service.get_products(
            category_id=category_id,
            page=page,
            limit=20
        )
        
        return {
            "products": [
                {
                    "id": product.id,
                    "name": product.name,
                    "price": product.price,
                    "image_url": product.thumbnail_url,  # Small image
                    "rating": product.average_rating,
                    "has_discount": product.discount_percentage > 0
                }
                for product in products
            ]
        }
```

#### 2. Web BFF (Rich Data for Desktop)
```python
class WebBFF:
    def __init__(self):
        self.user_service = UserServiceClient()
        self.product_service = ProductServiceClient()
        self.order_service = OrderServiceClient()
        self.recommendation_service = RecommendationServiceClient()
    
    async def get_user_dashboard(self, user_id):
        """Rich dashboard for web - detailed information"""
        
        # More parallel calls for richer data
        user_task = self.user_service.get_user_profile(user_id)
        orders_task = self.order_service.get_orders(user_id, limit=10)
        recommendations_task = self.recommendation_service.get_recommendations(user_id)
        analytics_task = self.user_service.get_user_analytics(user_id)
        
        user, orders, recommendations, analytics = await asyncio.gather(
            user_task, orders_task, recommendations_task, analytics_task
        )
        
        return {
            "user": {
                "id": user.id,
                "name": user.name,
                "email": user.email,
                "avatar_url": user.avatar_high_res,  # High-res image for web
                "membership_level": user.membership_level,
                "points": user.loyalty_points
            },
            "orders": [
                {
                    "id": order.id,
                    "status": order.status,
                    "total": order.total,
                    "date": order.created_at.strftime("%B %d, %Y"),  # Full date
                    "items_count": len(order.items),
                    "tracking_number": order.tracking_number
                }
                for order in orders
            ],
            "recommendations": recommendations,
            "analytics": {
                "total_spent": analytics.total_spent,
                "orders_this_month": analytics.orders_this_month,
                "favorite_categories": analytics.favorite_categories
            }
        }
    
    async def get_product_details(self, product_id, user_id=None):
        """Rich product details for web"""
        
        product_task = self.product_service.get_product(product_id)
        reviews_task = self.product_service.get_reviews(product_id, limit=20)
        related_task = self.product_service.get_related_products(product_id)
        
        tasks = [product_task, reviews_task, related_task]
        
        if user_id:
            # Add personalized data for logged-in users
            wishlist_task = self.user_service.is_in_wishlist(user_id, product_id)
            recommendations_task = self.recommendation_service.get_similar_products(
                user_id, product_id
            )
            tasks.extend([wishlist_task, recommendations_task])
        
        results = await asyncio.gather(*tasks)
        product, reviews, related = results[:3]
        
        response = {
            "product": {
                "id": product.id,
                "name": product.name,
                "description": product.full_description,  # Full description for web
                "price": product.price,
                "images": product.image_gallery,  # Multiple high-res images
                "specifications": product.specifications,
                "availability": product.inventory_status,
                "shipping_info": product.shipping_options
            },
            "reviews": reviews,
            "related_products": related
        }
        
        if user_id:
            in_wishlist, personalized_recommendations = results[3:]
            response.update({
                "in_wishlist": in_wishlist,
                "personalized_recommendations": personalized_recommendations
            })
        
        return response
```

#### 3. IoT BFF (Minimal, Efficient)
```python
class IoTBFF:
    def __init__(self):
        self.device_service = DeviceServiceClient()
        self.user_service = UserServiceClient()
    
    async def get_device_status(self, device_id):
        """Minimal response for IoT devices"""
        
        device = await self.device_service.get_device(device_id)
        
        # Return only essential data in compact format
        return {
            "s": device.status,  # 's' instead of 'status' to save bytes
            "t": int(device.temperature) if device.temperature else None,
            "b": device.battery_level,
            "ts": int(device.last_update.timestamp())
        }
    
    async def update_device_settings(self, device_id, settings):
        """Accept minimal settings update"""
        
        # Validate and expand abbreviated settings
        full_settings = self.expand_settings(settings)
        
        return await self.device_service.update_settings(device_id, full_settings)
    
    def expand_settings(self, abbreviated):
        """Convert abbreviated IoT format to full format"""
        mapping = {
            "t": "target_temperature",
            "m": "mode",
            "s": "schedule",
            "a": "auto_mode_enabled"
        }
        
        return {mapping.get(k, k): v for k, v in abbreviated.items()}
```

### BFF Benefits

#### 1. Client-Optimized APIs
```python
# Same data, different formats
class ProductBFF:
    async def get_product_for_mobile(self, product_id):
        product = await self.product_service.get_product(product_id)
        
        return {
            "id": product.id,
            "name": product.name[:30] + "..." if len(product.name) > 30 else product.name,
            "price": product.price,
            "image": product.mobile_thumbnail,
            "rating": round(product.rating, 1)
        }
    
    async def get_product_for_web(self, product_id):
        product = await self.product_service.get_product(product_id)
        
        return {
            "id": product.id,
            "name": product.name,
            "full_description": product.description,
            "price": product.price,
            "original_price": product.original_price,
            "discount_percentage": product.discount_percentage,
            "images": product.image_gallery,
            "rating": product.rating,
            "review_count": product.review_count,
            "specifications": product.specifications,
            "shipping_options": product.shipping_options
        }
```

#### 2. Reduced Client Complexity
```python
# BFF handles complex orchestration
class CheckoutBFF:
    async def initialize_checkout(self, user_id, items):
        """Single endpoint that orchestrates multiple services"""
        
        # Validate user
        user = await self.user_service.get_user(user_id)
        if not user.can_checkout:
            raise CheckoutError("User cannot checkout")
        
        # Validate and price items
        validated_items = await self.validate_items(items)
        
        # Calculate pricing with discounts
        pricing = await self.pricing_service.calculate_total(
            user_id, validated_items
        )
        
        # Get shipping options
        shipping_options = await self.shipping_service.get_options(
            user.address, validated_items
        )
        
        # Get payment methods
        payment_methods = await self.payment_service.get_user_methods(user_id)
        
        # Return everything client needs in one response
        return {
            "items": validated_items,
            "pricing": pricing,
            "shipping_options": shipping_options,
            "payment_methods": payment_methods,
            "user": {
                "addresses": user.addresses,
                "default_payment": user.default_payment_method
            }
        }
```

---

## Interview Questions

### Basic Level

**Q1: When would you choose a monolithic architecture over microservices?**

**Answer:**
Choose monoliths when:
- **Small team** (< 10 developers)
- **Simple domain** with limited complexity
- **Strong consistency** requirements across all operations
- **Rapid prototyping** or MVP development
- **Limited DevOps maturity**
- **Low to medium scale** (< 1000 concurrent users)

Example: A small e-commerce site for a local business with simple inventory management and order processing.

**Q2: What are the main challenges of microservices architecture?**

**Answer:**
Key challenges:
1. **Distributed System Complexity**: Network failures, latency, partial failures
2. **Data Consistency**: No ACID transactions across services
3. **Service Discovery**: Finding and connecting to services
4. **Monitoring**: Tracking requests across multiple services
5. **Testing**: Integration testing becomes complex
6. **Deployment**: Coordinating deployments across services

**Q3: Explain the role of an API Gateway.**

**Answer:**
API Gateway serves as:
- **Single Entry Point**: All client requests go through the gateway
- **Authentication Hub**: Centralizes security and token validation
- **Request Router**: Routes requests to appropriate microservices
- **Protocol Translator**: Handles different protocols (REST, GraphQL, gRPC)
- **Cross-cutting Concerns**: Rate limiting, logging, monitoring

### Intermediate Level

**Q4: Design an event-driven architecture for an e-commerce order processing system.**

**Answer Framework:**
1. **Events**:
   - OrderPlaced, PaymentProcessed, InventoryReserved, OrderShipped

2. **Services**:
   - Order Service (publishes OrderPlaced)
   - Payment Service (subscribes to OrderPlaced, publishes PaymentProcessed)
   - Inventory Service (subscribes to OrderPlaced, publishes InventoryReserved)
   - Fulfillment Service (subscribes to PaymentProcessed + InventoryReserved)

3. **Event Bus**: Apache Kafka for durability and scaling

4. **Error Handling**: Dead letter queues and compensation events

**Q5: How would you implement the Backend for Frontend (BFF) pattern?**

**Answer Framework:**
1. **Separate BFFs per Client Type**:
   ```
   Mobile BFF → Lightweight, minimal data
   Web BFF → Rich data, full features
   Partner API BFF → Specific partner requirements
   ```

2. **Implementation**:
   - Each BFF calls multiple downstream services
   - Aggregates and transforms data for specific client needs
   - Handles client-specific authentication/authorization

3. **Benefits**:
   - Optimized APIs for each client
   - Independent evolution of client experiences
   - Reduced client complexity

### Advanced Level

**Q6: Design a migration strategy from monolith to microservices for a large e-commerce platform.**

**Answer Framework:**
1. **Strangler Fig Pattern**:
   ```
   Phase 1: Extract User Service (low risk)
   Phase 2: Extract Product Catalog (read-heavy)
   Phase 3: Extract Order Processing (complex workflows)
   Phase 4: Extract Payment Processing (high security needs)
   ```

2. **Implementation Steps**:
   - Start with new features as separate services
   - Extract bounded contexts one at a time
   - Use feature flags to route traffic gradually
   - Maintain data consistency during transition

3. **Risk Mitigation**:
   - Canary deployments
   - Circuit breakers between services
   - Comprehensive monitoring and alerting
   - Rollback procedures for each phase

**Q7: How would you handle distributed transactions in a microservices architecture?**

**Answer Framework:**
1. **Avoid When Possible**:
   - Redesign boundaries to minimize distributed transactions
   - Use eventual consistency where acceptable

2. **Saga Pattern**:
   ```python
   # Choreography-based saga
   1. Order Service creates order → publishes OrderCreated
   2. Payment Service charges → publishes PaymentProcessed
   3. Inventory Service reserves → publishes InventoryReserved
   4. If any step fails → publish compensation events
   ```

3. **Implementation Options**:
   - **Choreography**: Services react to events independently
   - **Orchestration**: Central coordinator manages the saga
   - **Two-Phase Commit**: For critical consistency (rare in microservices)

4. **Error Handling**:
   - Compensation actions for each step
   - Timeout handling and retries
   - Dead letter queues for failed messages

---

**Next:** Continue to [Communication Protocols](../06-communication/README.md) to learn about different ways services can communicate with each other.