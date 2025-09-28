# Interview Preparation 🎯

Master system design interviews with structured approaches, common questions, and proven frameworks.

## Table of Contents

- [Interview Framework](#interview-framework)
- [Common System Design Questions](#common-system-design-questions)
- [Step-by-Step Solutions](#step-by-step-solutions)
- [Tips and Best Practices](#tips-and-best-practices)
- [Red Flags to Avoid](#red-flags-to-avoid)

---

## Interview Framework

### The RADIO Framework

#### **R - Requirements**
- **Functional Requirements**: What should the system do?
- **Non-Functional Requirements**: Performance, scalability, availability
- **Constraints**: Budget, timeline, existing systems

#### **A - Architecture**
- **High-level design**: Major components and their interactions
- **Data flow**: How information moves through the system
- **API design**: Key endpoints and interfaces

#### **D - Data Model**
- **Database schema**: Tables, relationships, indexes
- **Data storage**: SQL vs NoSQL decisions
- **Data partitioning**: Sharding strategies

#### **I - Implementation Details**
- **Deep dive**: Focus on 1-2 critical components
- **Algorithms**: Core business logic
- **Technology choices**: Justify decisions

#### **O - Operations & Scale**
- **Scalability**: How to handle growth
- **Monitoring**: Key metrics and alerting
- **Trade-offs**: Discuss pros and cons

---

## Common System Design Questions

### 1. Design a URL Shortener (like bit.ly)

**Requirements Gathering:**
```
Functional Requirements:
- Shorten long URLs
- Redirect short URLs to original URLs
- Custom aliases (optional)
- Analytics (optional)

Non-Functional Requirements:
- 100M URLs shortened per day
- 100:1 read/write ratio
- Low latency < 100ms
- High availability 99.9%

Scale:
- 100M writes/day = 1,157 writes/second
- 10B reads/day = 115,700 reads/second
- Storage: 100M * 365 * 5 years = 182B URLs
- Storage size: 182B * 500 bytes = 91TB
```

**High-Level Design:**
```
[Client] → [Load Balancer] → [App Servers] → [Cache] → [Database]
                                    ↓
                              [Analytics DB]
```

**Database Schema:**
```sql
urls:
- short_url (PK, varchar(7))
- long_url (text)
- user_id (int)
- created_at (timestamp)
- expires_at (timestamp)

url_analytics:
- short_url (FK)
- timestamp
- user_agent
- ip_address
- referer
```

**Algorithm for Short URL Generation:**
```python
import hashlib
import base64

def generate_short_url(long_url, custom=None):
    if custom:
        # Check if custom alias is available
        if is_available(custom):
            return custom
        else:
            raise Exception("Custom alias not available")
    
    # Base62 encoding approach
    counter = get_next_counter()  # Auto-incrementing counter
    short_url = base62_encode(counter)
    
    return short_url

def base62_encode(num):
    chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
    result = ""
    while num > 0:
        result = chars[num % 62] + result
        num //= 62
    return result
```

### 2. Design a Chat System

**Requirements:**
```
Functional:
- 1-on-1 messaging
- Group chat (up to 100 members)
- Online/offline status
- Message delivery confirmation
- Push notifications

Non-Functional:
- 50M daily active users
- Each user sends 40 messages/day
- Message size: 100 bytes average
- Real-time messaging
```

**Architecture:**
```
[Mobile App] ↔ [WebSocket Gateway] ↔ [Message Service]
                        ↓                    ↓
[Notification Service] [Load Balancer] [Message Queue]
                                            ↓
                                    [Database Cluster]
```

**Components:**
1. **WebSocket Gateway**: Maintains persistent connections
2. **Message Service**: Handles message routing and storage
3. **Presence Service**: Tracks user online/offline status
4. **Notification Service**: Sends push notifications
5. **Message Queue**: Ensures message delivery

### 3. Design a Social Media Newsfeed

**Requirements:**
```
Functional:
- Post creation (text, images, videos)
- Timeline generation
- Like, comment, share
- Follow/unfollow users

Scale:
- 1B users, 200M daily active
- Each user follows 200 people
- Each user sees 20 posts in timeline
- 2M posts created per day
```

**Feed Generation Approaches:**

#### Pull Model (On-Demand)
```python
def generate_timeline(user_id):
    following = get_following_list(user_id)
    posts = []
    
    for followed_user in following:
        user_posts = get_recent_posts(followed_user, limit=10)
        posts.extend(user_posts)
    
    # Sort by timestamp and return top N
    return sorted(posts, key=lambda x: x.timestamp)[-20:]
```

**Pros:** Storage efficient, good for users with many followers
**Cons:** High latency, expensive for active users

#### Push Model (Pre-compute)
```python
def on_post_created(user_id, post):
    followers = get_followers(user_id)
    
    for follower in followers:
        add_to_timeline(follower, post)
```

**Pros:** Fast timeline generation
**Cons:** Storage intensive, slow posting for users with many followers

#### Hybrid Approach
```python
def generate_timeline(user_id):
    # Pre-computed timeline for most users
    timeline = get_precomputed_timeline(user_id)
    
    # On-demand fetch for celebrity posts
    celebrity_following = get_celebrity_following(user_id)
    celebrity_posts = []
    
    for celebrity in celebrity_following:
        posts = get_recent_posts(celebrity, limit=5)
        celebrity_posts.extend(posts)
    
    # Merge and sort
    return merge_and_sort(timeline, celebrity_posts)
```

---

## Step-by-Step Solutions

### Design Approach Template

#### Step 1: Requirements (5-10 minutes)
```
Questions to Ask:
- Who are the users?
- What features are needed?
- How many users do we expect?
- What's the expected load?
- What devices will be used?
- What's the budget/timeline?
```

#### Step 2: High-Level Design (10-15 minutes)
```
Draw Major Components:
1. Client applications
2. Load balancers
3. Application servers
4. Databases
5. Cache layers
6. External services

Show data flow between components
```

#### Step 3: Data Model (10-15 minutes)
```
Define:
1. Core entities
2. Relationships between entities
3. Database choice (SQL vs NoSQL)
4. Partitioning strategy
```

#### Step 4: API Design (5-10 minutes)
```
Key APIs:
- POST /users - Create user
- GET /users/{id}/posts - Get user posts
- POST /posts - Create post
- GET /feed/{user_id} - Get user timeline
```

#### Step 5: Scale and Deep Dive (15-20 minutes)
```
Scaling Strategies:
1. Identify bottlenecks
2. Propose solutions
3. Discuss trade-offs
4. Consider edge cases
```

### Design Patterns for Scale

#### Read-Heavy Systems
```
Solutions:
- Multiple read replicas
- CDN for static content
- Multi-level caching
- Denormalized data structures

Example: Social media feeds, news websites
```

#### Write-Heavy Systems
```
Solutions:
- Write-optimized databases (Cassandra)
- Message queues for async processing
- Batch processing
- Partitioning/Sharding

Example: Chat systems, analytics platforms
```

#### Mixed Workloads
```
Solutions:
- CQRS (Command Query Responsibility Segregation)
- Event sourcing
- Microservices architecture
- Database per service

Example: E-commerce platforms
```

---

## Tips and Best Practices

### Before the Interview

1. **Practice Drawing**: Get comfortable with whiteboards/online tools
2. **Time Management**: Practice 45-60 minute sessions
3. **Study Real Systems**: Read engineering blogs from major companies
4. **Understand Trade-offs**: Every decision has pros and cons
5. **Know Your Numbers**: Memorize latency numbers and capacity planning

### During the Interview

1. **Start with Requirements**: Don't jump into solution immediately
2. **Think Out Loud**: Explain your thought process
3. **Ask Questions**: Clarify ambiguities
4. **Start Simple**: Build basic version first, then scale
5. **Be Realistic**: Don't over-engineer or under-estimate complexity

### Key Numbers to Remember

```
Latency Numbers:
- L1 cache: 0.5 ns
- L2 cache: 7 ns
- RAM: 100 ns
- SSD: 150,000 ns
- Network within datacenter: 500,000 ns
- Disk: 10,000,000 ns
- Network cross-continental: 150,000,000 ns

Capacity Planning:
- 1 byte = 8 bits
- 1 KB = 1,024 bytes
- 1 MB = 1,024 KB
- 1 GB = 1,024 MB
- 1 TB = 1,024 GB

QPS Estimates:
- 1M users, 10% daily active = 100K DAU
- Each user makes 10 requests/day
- Peak load = average load * 3
- 100K * 10 / 86400 = 11.6 QPS average
- Peak = 35 QPS
```

### Architecture Decision Framework

```python
def choose_database(requirements):
    if requirements.needs_acid_transactions:
        return "SQL Database (PostgreSQL, MySQL)"
    elif requirements.needs_flexible_schema:
        return "Document Database (MongoDB)"
    elif requirements.needs_high_write_throughput:
        return "Column Store (Cassandra)"
    elif requirements.needs_simple_key_value:
        return "Key-Value Store (Redis, DynamoDB)"
    elif requirements.needs_graph_relationships:
        return "Graph Database (Neo4j)"

def choose_caching_strategy(access_pattern):
    if access_pattern == "read_heavy":
        return "Cache-aside with Redis"
    elif access_pattern == "write_through_consistency":
        return "Write-through cache"
    elif access_pattern == "write_heavy_eventual_consistency":
        return "Write-behind cache"
```

---

## Red Flags to Avoid

### Technical Red Flags

1. **No Requirements Gathering**: Jumping straight into solution
2. **Over-Engineering**: Adding unnecessary complexity
3. **Ignoring Trade-offs**: Not discussing pros/cons of decisions
4. **Unrealistic Assumptions**: Ignoring real-world constraints
5. **Single Point of Failure**: Not considering fault tolerance

### Communication Red Flags

1. **Not Asking Questions**: Assuming requirements without clarification
2. **Silent Thinking**: Not explaining thought process
3. **Arguing with Interviewer**: Being defensive about feedback
4. **Going Too Deep Initially**: Missing the big picture
5. **Ignoring Time Management**: Spending too long on one aspect

### Example Bad Approaches

#### Bad: Jumping to Solution
```
"For a URL shortener, I'll use MongoDB to store URLs 
and Redis for caching..."
```

#### Good: Starting with Requirements
```
"Let me first understand the requirements. How many URLs 
do we expect to shorten per day? What's the read/write ratio? 
Do we need analytics? What's our latency requirement?"
```

#### Bad: Over-Engineering
```
"I'll use a microservices architecture with Kubernetes, 
service mesh, event sourcing, CQRS..."
```

#### Good: Right-Sized Solution
```
"For this scale, a simple 3-tier architecture with load 
balancing, application servers, and a database cluster 
should work well. We can consider microservices as we scale."
```

### Quick Self-Check

Before proposing any solution, ask yourself:
1. ✅ Did I gather requirements first?
2. ✅ Am I solving the right problem?
3. ✅ Is my solution appropriately complex for the scale?
4. ✅ Have I considered failure scenarios?
5. ✅ Can I explain the trade-offs?

---

**Next:** Practice with [Real-World Examples](../09-real-world/README.md) to see how major companies solve similar problems.