# Caching and Async Processing ⚡

Optimizing system performance through strategic caching and asynchronous processing patterns.

## Table of Contents

- [Caching Strategies](#caching-strategies)
- [Cache Patterns](#cache-patterns)
- [Distributed Caching](#distributed-caching)
- [Message Queues](#message-queues)
- [Asynchronous Processing](#asynchronous-processing)
- [Event Streaming](#event-streaming)
- [Interview Questions](#interview-questions)

---

## Caching Strategies

### What is Caching?

Caching stores frequently accessed data in fast storage to reduce response time and system load.

```
Without Cache:
Client ──► Application ──► Database (slow)

With Cache:
Client ──► Application ──► Cache (fast) ──► Database (if cache miss)
```

### Cache Levels

#### 1. Browser Cache
```
┌─────────────┐    ┌─────────────┐
│   Browser   │───▶│    CDN      │
│   Cache     │    │   Cache     │
└─────────────┘    └─────────────┘
```
- Static assets (CSS, JS, images)
- TTL: Hours to days
- Controlled by HTTP headers

#### 2. CDN (Content Delivery Network)
```
     ┌─────────────┐
     │   Origin    │
     │   Server    │
     └──────┬──────┘
            │
    ┌───────┼───────┐
    ▼       ▼       ▼
┌─────┐ ┌─────┐ ┌─────┐
│CDN 1│ │CDN 2│ │CDN 3│
└─────┘ └─────┘ └─────┘
```
- Geographic distribution
- Static and dynamic content
- Edge locations for low latency

#### 3. Application-Level Cache
```
┌─────────────────────────────────┐
│         Application             │
│                                │
│  ┌─────────────────────────┐   │
│  │    In-Memory Cache      │   │
│  │   (Redis, Memcached)    │   │
│  └─────────────────────────┘   │
└─────────────────────────────────┘
```

#### 4. Database Cache
```
┌─────────────────────────────────┐
│           Database              │
│                                │
│  ┌─────────────────────────┐   │
│  │     Query Cache         │   │
│  │   (Buffer Pool)         │   │
│  └─────────────────────────┘   │
└─────────────────────────────────┘
```

---

## Cache Patterns

### Cache-Aside (Lazy Loading)

```python
def get_user(user_id):
    # Try cache first
    user = cache.get(f"user:{user_id}")
    if user is None:
        # Cache miss - get from database
        user = database.get_user(user_id)
        if user:
            cache.set(f"user:{user_id}", user, ttl=3600)
    return user

def update_user(user_id, data):
    # Update database first
    database.update_user(user_id, data)
    # Invalidate cache
    cache.delete(f"user:{user_id}")
```

**Flow:**
```
1. Application checks cache
2. If miss, reads from database
3. Application updates cache
4. Returns data
```

**Pros:**
- Cache only what's needed
- Resilient to cache failures
- Simple to implement

**Cons:**
- Cache miss penalty
- Data consistency issues
- More complex code

### Write-Through

```python
def update_user(user_id, data):
    # Update database and cache simultaneously
    database.update_user(user_id, data)
    cache.set(f"user:{user_id}", data, ttl=3600)
```

**Flow:**
```
1. Application writes to cache
2. Cache immediately writes to database
3. Confirms write completion
```

**Pros:**
- Data consistency
- No cache miss penalty
- Simple read operations

**Cons:**
- Higher write latency
- Caches data that may not be read
- More complex infrastructure

### Write-Behind (Write-Back)

```python
def update_user(user_id, data):
    # Update cache immediately
    cache.set(f"user:{user_id}", data, ttl=3600)
    # Queue database update for later
    queue.enqueue('update_user', user_id, data)

def background_worker():
    while True:
        task = queue.dequeue()
        if task.type == 'update_user':
            database.update_user(task.user_id, task.data)
```

**Flow:**
```
1. Application writes to cache
2. Cache confirms write
3. Database write happens asynchronously
```

**Pros:**
- Lowest write latency
- Better write throughput
- Reduced database load

**Cons:**
- Risk of data loss
- Complex error handling
- Eventual consistency

### Refresh-Ahead

```python
import threading
from datetime import datetime, timedelta

def get_user_with_refresh(user_id):
    cache_key = f"user:{user_id}"
    cached_data = cache.get_with_metadata(cache_key)
    
    if cached_data is None:
        # Cache miss - fetch from database
        return fetch_and_cache_user(user_id)
    
    # Check if refresh is needed (before expiration)
    if cached_data.expires_at - datetime.now() < timedelta(minutes=5):
        # Trigger async refresh
        threading.Thread(target=refresh_user_cache, args=(user_id,)).start()
    
    return cached_data.value

def refresh_user_cache(user_id):
    user = database.get_user(user_id)
    cache.set(f"user:{user_id}", user, ttl=3600)
```

**Pros:**
- Low latency reads
- Automatic cache warming
- High cache hit rate

**Cons:**
- Complex implementation
- Potential for stale data
- Higher resource usage

---

## Distributed Caching

### Redis Architecture

#### Single Instance
```
┌─────────────┐    ┌─────────────┐
│Application 1│───▶│    Redis    │
└─────────────┘    │   Instance  │
┌─────────────┐    │             │
│Application 2│───▶│             │
└─────────────┘    └─────────────┘
```

**Pros:**
- Simple setup
- ACID operations
- Rich data structures

**Cons:**
- Single point of failure
- Limited scalability
- Memory constraints

#### Redis Cluster
```
┌─────────┐  ┌─────────┐  ┌─────────┐
│ Shard 1 │  │ Shard 2 │  │ Shard 3 │
│ Slots   │  │ Slots   │  │ Slots   │
│ 0-5460  │  │5461-10922│ │10923-16383│
└─────────┘  └─────────┘  └─────────┘
     │            │            │
     │            │            │
┌─────────┐  ┌─────────┐  ┌─────────┐
│Replica 1│  │Replica 2│  │Replica 3│
└─────────┘  └─────────┘  └─────────┘
```

**Features:**
- Automatic sharding
- High availability
- Linear scalability
- No single point of failure

### Cache Eviction Policies

#### LRU (Least Recently Used)
```
Cache: [A, B, C, D] (D is most recent)
Access: E
Result: [B, C, D, E] (A evicted)
```

#### LFU (Least Frequently Used)
```
Access counts: A(5), B(2), C(8), D(1)
New item: E
Result: D evicted (lowest frequency)
```

#### TTL (Time To Live)
```python
cache.set("key", "value", ttl=3600)  # Expires in 1 hour
```

#### Random
```
Randomly select item for eviction
Good when access patterns are unpredictable
```

### Cache Consistency

#### Cache Invalidation Strategies

1. **TTL-Based**
```python
cache.set("user:123", user_data, ttl=300)  # 5 minutes
```

2. **Event-Based**
```python
def on_user_update(user_id):
    cache.delete(f"user:{user_id}")
    publish_event("user_updated", {"user_id": user_id})
```

3. **Version-Based**
```python
def get_user(user_id):
    version = get_user_version(user_id)
    cache_key = f"user:{user_id}:v{version}"
    return cache.get(cache_key)
```

---

## Message Queues

### Why Message Queues?

```
Synchronous Processing:
Client ──► API ──► Heavy Processing ──► Response
       (waits for entire process)

Asynchronous Processing:
Client ──► API ──► Queue ──► Response (immediate)
                    │
                    └──► Background Worker
```

### Queue Patterns

#### Work Queue (Point-to-Point)
```
Producer ──► Queue ──► Consumer 1
                │
                └──► Consumer 2
                │
                └──► Consumer 3
```

**Use Cases:**
- Task processing
- Background jobs
- Load distribution

#### Pub/Sub (Publish-Subscribe)
```
Publisher ──► Topic ──► Subscriber 1
                │
                ├──► Subscriber 2
                │
                └──► Subscriber 3
```

**Use Cases:**
- Event notifications
- Real-time updates
- Microservices communication

### RabbitMQ

#### Architecture
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Producer   │───▶│  Exchange   │───▶│    Queue    │
└─────────────┘    └─────────────┘    └─────────────┘
                         │                    │
                         │                    ▼
                         │            ┌─────────────┐
                         │            │  Consumer   │
                         │            └─────────────┘
                         │
                   ┌─────────────┐
                   │   Binding   │
                   └─────────────┘
```

#### Exchange Types

**Direct Exchange:**
```python
# Publisher
channel.basic_publish(
    exchange='direct_logs',
    routing_key='error',
    body='Error message'
)

# Consumer
channel.queue_bind(
    exchange='direct_logs',
    queue='error_queue',
    routing_key='error'
)
```

**Topic Exchange:**
```python
# Publisher
channel.basic_publish(
    exchange='topic_logs',
    routing_key='user.profile.update',
    body='User profile updated'
)

# Consumer - matches user.* patterns
channel.queue_bind(
    exchange='topic_logs',
    queue='user_queue',
    routing_key='user.*'
)
```

**Fanout Exchange:**
```python
# Publisher
channel.basic_publish(
    exchange='fanout_logs',
    routing_key='',  # ignored
    body='Broadcast message'
)

# All bound queues receive the message
```

### Apache Kafka

#### Architecture
```
┌─────────────────────────────────────────────────┐
│                Kafka Cluster                    │
│                                                │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐         │
│  │Broker 1 │  │Broker 2 │  │Broker 3 │         │
│  └─────────┘  └─────────┘  └─────────┘         │
│                                                │
│  Topic: user-events                            │
│  ┌─────────────────────────────────────────┐   │
│  │ Partition 0 │ Partition 1 │ Partition 2 │   │
│  └─────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

#### Key Concepts

**Topics and Partitions:**
```
Topic: user-events
├── Partition 0: [msg1, msg4, msg7, ...]
├── Partition 1: [msg2, msg5, msg8, ...]
└── Partition 2: [msg3, msg6, msg9, ...]
```

**Consumer Groups:**
```python
# Consumer Group: user-processors
Consumer 1 ──► Partition 0, 1
Consumer 2 ──► Partition 2

# Each partition consumed by only one consumer in the group
```

**Kafka vs RabbitMQ:**

| Feature | Kafka | RabbitMQ |
|---------|-------|----------|
| **Message Ordering** | Per-partition | Per-queue |
| **Message Retention** | Configurable | Consumed and deleted |
| **Throughput** | Very High | High |
| **Use Case** | Stream processing | Task queues |
| **Complexity** | Higher | Lower |

---

## Asynchronous Processing

### Task Queue Patterns

#### Celery with Redis/RabbitMQ
```python
from celery import Celery

app = Celery('myapp', broker='redis://localhost:6379')

@app.task
def send_email(user_id, subject, body):
    user = get_user(user_id)
    email_service.send(user.email, subject, body)

# Usage
def register_user(user_data):
    user = create_user(user_data)
    # Async email sending
    send_email.delay(user.id, "Welcome!", "Thanks for registering!")
    return user
```

#### Background Job Processing
```python
# Job definition
class ProcessImageJob:
    def __init__(self, image_url, user_id):
        self.image_url = image_url
        self.user_id = user_id
    
    def execute(self):
        # Download image
        image = download_image(self.image_url)
        # Process image
        processed = resize_and_optimize(image)
        # Upload to storage
        url = upload_to_s3(processed)
        # Update database
        update_user_avatar(self.user_id, url)

# Job queue
job_queue = Queue('image_processing')

# Producer
def upload_avatar(user_id, image_url):
    job = ProcessImageJob(image_url, user_id)
    job_queue.enqueue(job)
    return {"status": "processing"}

# Consumer
def worker():
    while True:
        job = job_queue.dequeue(timeout=30)
        if job:
            job.execute()
```

### Event-Driven Architecture

#### Event Sourcing Pattern
```python
# Events
class UserRegistered:
    def __init__(self, user_id, email, timestamp):
        self.user_id = user_id
        self.email = email
        self.timestamp = timestamp

class EmailVerified:
    def __init__(self, user_id, timestamp):
        self.user_id = user_id
        self.timestamp = timestamp

# Event Store
class EventStore:
    def append(self, stream_id, events):
        for event in events:
            self.store_event(stream_id, event)
    
    def get_events(self, stream_id):
        return self.load_events(stream_id)

# Event Handler
def handle_user_registered(event):
    # Send welcome email
    send_welcome_email.delay(event.user_id)
    # Create user profile
    create_user_profile.delay(event.user_id)
    # Send analytics event
    track_user_registration.delay(event.user_id)
```

#### Saga Pattern for Distributed Transactions
```python
class OrderSaga:
    def __init__(self, order_id):
        self.order_id = order_id
        self.state = "STARTED"
    
    def execute(self):
        try:
            # Step 1: Reserve inventory
            inventory_service.reserve(self.order_id)
            self.state = "INVENTORY_RESERVED"
            
            # Step 2: Process payment
            payment_service.charge(self.order_id)
            self.state = "PAYMENT_PROCESSED"
            
            # Step 3: Update order status
            order_service.confirm(self.order_id)
            self.state = "COMPLETED"
            
        except Exception as e:
            self.compensate()
    
    def compensate(self):
        if self.state == "PAYMENT_PROCESSED":
            payment_service.refund(self.order_id)
        if self.state == "INVENTORY_RESERVED":
            inventory_service.release(self.order_id)
        self.state = "COMPENSATED"
```

---

## Event Streaming

### Stream Processing Concepts

#### Batch vs Stream Processing
```
Batch Processing:
Data ──► [Collect] ──► [Process] ──► Results
         (hours)      (minutes)

Stream Processing:
Data ──► [Process] ──► Results
         (milliseconds)
```

#### Apache Kafka Streams
```python
from kafka import KafkaProducer, KafkaConsumer

# Producer
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode()
)

# Send user events
producer.send('user-events', {
    'user_id': 123,
    'event': 'page_view',
    'page': '/products',
    'timestamp': time.time()
})

# Consumer
consumer = KafkaConsumer(
    'user-events',
    bootstrap_servers=['localhost:9092'],
    value_deserializer=lambda m: json.loads(m.decode())
)

for message in consumer:
    event = message.value
    # Process event in real-time
    process_user_event(event)
```

### Stream Processing Patterns

#### Windowing
```python
# Time-based windowing
def process_user_activity():
    window_size = 300  # 5 minutes
    
    for window_start in time_windows(window_size):
        events = get_events_in_window(window_start, window_size)
        
        # Calculate metrics
        page_views = count_page_views(events)
        unique_users = count_unique_users(events)
        
        # Store results
        store_metrics(window_start, page_views, unique_users)
```

#### Event Aggregation
```python
# Real-time user activity aggregation
class UserActivityAggregator:
    def __init__(self):
        self.user_sessions = {}
    
    def process_event(self, event):
        user_id = event['user_id']
        
        if user_id not in self.user_sessions:
            self.user_sessions[user_id] = {
                'page_views': 0,
                'session_start': event['timestamp'],
                'last_activity': event['timestamp']
            }
        
        session = self.user_sessions[user_id]
        session['page_views'] += 1
        session['last_activity'] = event['timestamp']
        
        # Emit updated session
        self.emit_session_update(user_id, session)
```

---

## Interview Questions

### Basic Level

**Q1: What are the benefits of caching?**

**Answer:**
- **Reduced Latency**: Faster data access from memory vs disk
- **Decreased Load**: Reduces database queries and external API calls
- **Better Scalability**: Handle more requests with same infrastructure
- **Cost Savings**: Reduces expensive operations (database queries, API calls)
- **Improved User Experience**: Faster page loads and responses

**Q2: Explain the difference between synchronous and asynchronous processing.**

**Answer:**
- **Synchronous**: Client waits for operation to complete before getting response
  - Example: User uploads file → Server processes → Returns result
  - Pros: Simple, immediate feedback
  - Cons: Slow response times, blocking operations

- **Asynchronous**: Client gets immediate response, processing happens in background
  - Example: User uploads file → Server queues job → Returns "Processing" → Worker processes file
  - Pros: Fast responses, better scalability
  - Cons: Complex error handling, eventual consistency

**Q3: When would you use a message queue?**

**Answer:**
Use message queues when:
- **Decoupling**: Separate producers from consumers
- **Reliability**: Ensure messages aren't lost during processing
- **Scalability**: Handle traffic spikes by queuing requests
- **Background Processing**: Send emails, generate reports, process images
- **Microservices Communication**: Reliable inter-service messaging

### Intermediate Level

**Q4: Compare cache-aside vs write-through caching patterns.**

**Answer:**

| Aspect | Cache-Aside | Write-Through |
|--------|-------------|---------------|
| **Read** | App checks cache first, loads from DB on miss | Same as cache-aside |
| **Write** | App updates DB, then invalidates/updates cache | App updates cache, cache updates DB |
| **Consistency** | Eventually consistent | Strongly consistent |
| **Performance** | Cache miss penalty | Higher write latency |
| **Use Case** | Read-heavy workloads | Write-heavy with consistency needs |

**Example scenarios:**
- **Cache-aside**: Social media feeds, product catalogs
- **Write-through**: User profiles, configuration settings

**Q5: Design a caching strategy for an e-commerce product catalog.**

**Answer Framework:**
1. **Multi-Level Caching**:
   - CDN: Product images, static content (TTL: 24 hours)
   - Application Cache: Popular products (TTL: 1 hour)
   - Database Query Cache: Search results (TTL: 30 minutes)

2. **Cache Keys**:
   ```
   product:{product_id}
   category:{category_id}:products
   search:{query_hash}:page:{page_num}
   ```

3. **Invalidation Strategy**:
   - Product updates: Invalidate specific product and related category caches
   - Inventory updates: Use TTL-based expiration
   - Price changes: Immediate invalidation with event-driven updates

4. **Implementation**:
   ```python
   def get_product(product_id):
       # Try cache first
       product = redis.get(f"product:{product_id}")
       if not product:
           product = db.get_product(product_id)
           redis.setex(f"product:{product_id}", 3600, product)
       return product
   ```

### Advanced Level

**Q6: Design a system to process 1 million events per second with message queues.**

**Answer Framework:**
1. **Architecture**:
   ```
   Producers → Load Balancer → Kafka Cluster → Consumer Groups → Processing
   ```

2. **Kafka Configuration**:
   - **Partitions**: 100+ partitions per topic for parallelism
   - **Replication**: 3 replicas for fault tolerance
   - **Batch Size**: Optimize for throughput vs latency

3. **Consumer Strategy**:
   - **Consumer Groups**: Multiple groups for different processing needs
   - **Parallel Processing**: One consumer per partition
   - **Auto-scaling**: Scale consumers based on lag metrics

4. **Optimizations**:
   - **Compression**: Use Snappy/LZ4 compression
   - **Batching**: Process events in batches
   - **Async Processing**: Non-blocking I/O operations
   - **Circuit Breakers**: Handle downstream service failures

5. **Monitoring**:
   ```python
   # Key metrics to monitor
   - Consumer lag per partition
   - Throughput (messages/second)
   - Error rates
   - Processing time percentiles
   ```

**Q7: How would you implement cache warming for a high-traffic application?**

**Answer Framework:**
1. **Strategies**:
   - **Scheduled Warming**: Cron jobs to preload popular data
   - **Predictive Warming**: ML models to predict cache needs
   - **Event-Driven Warming**: Warm cache on data updates

2. **Implementation**:
   ```python
   class CacheWarmer:
       def __init__(self):
           self.scheduler = BackgroundScheduler()
       
       def warm_popular_products(self):
           popular_ids = analytics.get_popular_product_ids()
           for product_id in popular_ids:
               product = db.get_product(product_id)
               cache.set(f"product:{product_id}", product, ttl=7200)
       
       def start(self):
           self.scheduler.add_job(
               self.warm_popular_products,
               'interval',
               minutes=30
           )
           self.scheduler.start()
   ```

3. **Advanced Techniques**:
   - **Gradual Warming**: Spread cache warming over time
   - **Priority-Based**: Warm most important data first
   - **Feedback Loop**: Monitor cache hit rates and adjust

4. **Considerations**:
   - **Resource Impact**: Don't overwhelm database during warming
   - **Timing**: Warm during low-traffic periods
   - **Monitoring**: Track warming effectiveness and costs

---

**Next:** Continue to [Architecture Patterns](../05-architecture-patterns/README.md) to learn about different architectural approaches for building scalable systems.