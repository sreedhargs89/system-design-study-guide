# Database and Storage 🗄️

Understanding different database types, scaling strategies, and storage patterns is essential for building data-intensive applications.

## Table of Contents

- [Relational Databases](#relational-databases)
- [Database Isolation Levels](#database-isolation-levels)
- [Scaling Databases](#scaling-databases)
- [Sharding and Partitioning](#sharding-and-partitioning)
- [Non-Relational Databases](#non-relational-databases)
- [Database Selection Guide](#database-selection-guide)
- [Storage Patterns](#storage-patterns)
- [Interview Questions](#interview-questions)

---

## Relational Databases

### ACID Properties

Relational databases follow ACID principles to ensure data integrity:

#### Atomicity
```
Transaction: Transfer $100 from Account A to Account B

BEGIN TRANSACTION
  UPDATE accounts SET balance = balance - 100 WHERE id = 'A';
  UPDATE accounts SET balance = balance + 100 WHERE id = 'B';
COMMIT; -- All operations succeed, or all fail
```

#### Consistency
```
Before: A=$500, B=$300, Total=$800
After:  A=$400, B=$400, Total=$800
```
Database constraints and triggers ensure data remains valid.

#### Isolation
```
Transaction 1: Reading account balance
Transaction 2: Updating same account
```
Transactions don't interfere with each other during concurrent execution.

#### Durability
```
Transaction commits ──► Data written to disk ──► Survives system crash
```
Committed data persists even if system fails.

### SQL Database Examples

#### PostgreSQL
```
┌─────────────────────────────────────┐
│            PostgreSQL               │
├─────────────────────────────────────┤
│ • ACID compliant                   │
│ • JSON/JSONB support               │
│ • Advanced indexing                │
│ • Full-text search                 │
│ • Extensible                       │
│                                    │
│ Use Cases:                         │
│ • Complex queries                  │
│ • Analytics                        │
│ • Data warehousing                 │
└─────────────────────────────────────┘
```

#### MySQL
```
┌─────────────────────────────────────┐
│              MySQL                  │
├─────────────────────────────────────┤
│ • High performance                 │
│ • Easy replication                 │
│ • Large community                  │
│ • Multiple storage engines         │
│                                    │
│ Use Cases:                         │
│ • Web applications                 │
│ • E-commerce                       │
│ • Content management               │
└─────────────────────────────────────┘
```

### Database Design Principles

#### Normalization
Breaking down tables to reduce redundancy:

```sql
-- Before Normalization (1NF violation)
Orders: [order_id, customer_name, customer_email, product_name, product_price]

-- After Normalization
Customers: [customer_id, name, email]
Products:  [product_id, name, price]  
Orders:    [order_id, customer_id, product_id, quantity]
```

#### Denormalization
Strategic redundancy for performance:

```sql
-- Denormalized for faster reads
OrderSummary: [order_id, customer_name, total_amount, product_count]
```

---

## Database Isolation Levels

### Read Uncommitted
```
Transaction A: UPDATE balance = 100
Transaction B: SELECT balance  -- Reads 100 (uncommitted)
Transaction A: ROLLBACK        -- Balance reverts
```
**Issues**: Dirty reads
**Use Case**: Analytics where approximate data is acceptable

### Read Committed
```
Transaction A: UPDATE balance = 100
Transaction B: SELECT balance  -- Reads old value until A commits
Transaction A: COMMIT
Transaction B: SELECT balance  -- Now reads 100
```
**Issues**: Non-repeatable reads
**Use Case**: Most web applications (PostgreSQL default)

### Repeatable Read
```
Transaction A: SELECT balance  -- Reads 50
Transaction B: UPDATE balance = 100; COMMIT
Transaction A: SELECT balance  -- Still reads 50
```
**Issues**: Phantom reads
**Use Case**: Financial applications (MySQL InnoDB default)

### Serializable
```
Transactions execute as if they ran one after another
```
**Issues**: Lowest concurrency, highest isolation
**Use Case**: Critical financial operations

### Comparison Table

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|----------------|------------|-------------------|--------------|-------------|
| Read Uncommitted | ✓ | ✓ | ✓ | Highest |
| Read Committed | ✗ | ✓ | ✓ | High |
| Repeatable Read | ✗ | ✗ | ✓ | Medium |
| Serializable | ✗ | ✗ | ✗ | Lowest |

---

## Scaling Databases

### Vertical Scaling (Scale Up)
```
┌─────────────────┐     ┌─────────────────┐
│   Database      │ ──► │   Database      │
│   4 CPU cores   │     │   16 CPU cores  │
│   8GB RAM       │     │   64GB RAM      │
│   100GB SSD     │     │   1TB SSD       │
└─────────────────┘     └─────────────────┘
```

**Pros:**
- Simple implementation
- ACID properties maintained
- No application changes needed

**Cons:**
- Hardware limits
- Expensive at scale
- Single point of failure

### Horizontal Scaling (Scale Out)

#### Read Replicas
```
                  ┌──────────┐
                  │  Master  │ ← Writes
                  │    DB    │
                  └─────┬────┘
                        │
            ┌───────────┼───────────┐
            ▼           ▼           ▼
      ┌─────────┐ ┌─────────┐ ┌─────────┐
      │Replica 1│ │Replica 2│ │Replica 3│ ← Reads
      └─────────┘ └─────────┘ └─────────┘
```

**Benefits:**
- Increased read capacity
- Geographic distribution
- Backup and analytics

**Challenges:**
- Replication lag
- Eventually consistent reads
- Write bottleneck on master

#### Database Sharding
```
┌─────────────────────────────────────────┐
│            Application                  │
└─────────────┬───────────────────────────┘
              │
      ┌───────┼───────┐
      ▼       ▼       ▼
┌─────────┐ ┌─────────┐ ┌─────────┐
│ Shard 1 │ │ Shard 2 │ │ Shard 3 │
│Users    │ │Users    │ │Users    │
│A-F      │ │G-M      │ │N-Z      │
└─────────┘ └─────────┘ └─────────┘
```

**Benefits:**
- Horizontal scalability
- Improved performance
- Fault isolation

**Challenges:**
- Application complexity
- Cross-shard queries
- Rebalancing difficulties

---

## Sharding and Partitioning

### Sharding Strategies

#### Range-Based Sharding
```
Shard 1: user_id 1-1000
Shard 2: user_id 1001-2000  
Shard 3: user_id 2001-3000
```

**Pros:**
- Simple to implement
- Range queries are efficient

**Cons:**
- Hotspots possible
- Uneven data distribution

#### Hash-Based Sharding
```
user_id → hash(user_id) % num_shards → shard_id

Example:
user_id 12345 → hash(12345) % 3 → Shard 1
user_id 67890 → hash(67890) % 3 → Shard 2
```

**Pros:**
- Even distribution
- No hotspots

**Cons:**
- Range queries difficult
- Resharding complexity

#### Directory-Based Sharding
```
┌─────────────────┐
│  Lookup Service │
├─────────────────┤
│ user_1 → shard_1│
│ user_2 → shard_3│
│ user_3 → shard_2│
└─────────────────┘
```

**Pros:**
- Flexible shard assignment
- Easy to rebalance

**Cons:**
- Additional complexity
- Lookup service bottleneck

### Partitioning Types

#### Horizontal Partitioning (Sharding)
```
Original Table:
┌─────────────────────────────┐
│ id | name    | region       │
│ 1  | Alice   | North        │
│ 2  | Bob     | South        │
│ 3  | Charlie | North        │
│ 4  | David   | South        │
└─────────────────────────────┘

Partitioned by Region:
North Partition:     South Partition:
┌───────────────┐    ┌───────────────┐
│ id | name     │    │ id | name     │
│ 1  | Alice    │    │ 2  | Bob      │
│ 3  | Charlie  │    │ 4  | David    │
└───────────────┘    └───────────────┘
```

#### Vertical Partitioning
```
Original Table:
┌─────────────────────────────────┐
│ id | name  | email      | bio   │
└─────────────────────────────────┘

Partitioned Vertically:
Basic Info:          Extended Info:
┌─────────────┐     ┌─────────────┐
│ id | name   │     │ id | bio    │
└─────────────┘     └─────────────┘
```

---

## Non-Relational Databases

### Document Databases

#### MongoDB
```json
{
  "_id": "user123",
  "name": "Alice",
  "email": "alice@example.com",
  "addresses": [
    {
      "type": "home",
      "street": "123 Main St",
      "city": "Anytown"
    },
    {
      "type": "work", 
      "street": "456 Office Blvd",
      "city": "Downtown"
    }
  ],
  "preferences": {
    "theme": "dark",
    "notifications": true
  }
}
```

**Pros:**
- Flexible schema
- Natural object mapping
- Horizontal scaling
- Rich querying

**Cons:**
- Limited ACID properties
- Memory usage
- Complex transactions

**Use Cases:**
- Content management
- Product catalogs
- User profiles
- Real-time analytics

### Key-Value Stores

#### Redis
```
SET user:123:name "Alice"
SET user:123:email "alice@example.com"
HSET user:123:prefs theme "dark" notifications "true"

GET user:123:name    → "Alice"
HGET user:123:prefs theme → "dark"
```

**Pros:**
- Extremely fast (in-memory)
- Simple data model
- Excellent for caching
- Pub/Sub capabilities

**Cons:**
- Limited querying
- Memory constraints
- No complex relationships

**Use Cases:**
- Caching
- Session storage
- Real-time leaderboards
- Message queues

#### Amazon DynamoDB
```
Table: Users
Partition Key: user_id
Sort Key: timestamp

Items:
{
  "user_id": "alice",
  "timestamp": "2024-01-01T00:00:00Z",
  "action": "login",
  "metadata": {...}
}
```

**Pros:**
- Serverless
- Automatic scaling
- High availability
- Consistent performance

**Cons:**
- AWS lock-in
- Limited querying
- Learning curve
- Costs can be high

### Column-Family Databases

#### Cassandra
```
Column Family: Users

Row Key: user_id
Columns: name, email, created_at, last_login

user123: {
  name: "Alice",
  email: "alice@example.com", 
  created_at: "2024-01-01",
  last_login: "2024-01-15"
}
```

**Pros:**
- Excellent write performance
- Linear scalability
- No single point of failure
- Tunable consistency

**Cons:**
- Limited query flexibility
- Complex operations
- Memory intensive
- Learning curve

**Use Cases:**
- Time-series data
- IoT applications
- Real-time analytics
- High-write applications

### Graph Databases

#### Neo4j
```
Nodes: (Person), (Movie), (Genre)
Relationships: [ACTED_IN], [DIRECTED], [BELONGS_TO]

(Alice:Person)-[ACTED_IN]->(Movie1:Movie)-[BELONGS_TO]->(Action:Genre)
(Bob:Person)-[DIRECTED]->(Movie1:Movie)
```

**Pros:**
- Natural relationship modeling
- Efficient graph traversals
- Flexible schema
- ACID properties

**Cons:**
- Specialized use cases
- Learning curve
- Memory requirements
- Limited horizontal scaling

**Use Cases:**
- Social networks
- Recommendation engines
- Fraud detection
- Network analysis

---

## Database Selection Guide

### Selection Criteria Matrix

| Database Type | Consistency | Scalability | Query Flexibility | Performance | Use Cases |
|---------------|-------------|-------------|------------------|-------------|-----------|
| **Relational** | Strong | Moderate | High | Good | OLTP, Complex queries |
| **Document** | Eventual | High | Medium | Good | CMS, Catalogs |
| **Key-Value** | Tunable | Very High | Low | Excellent | Caching, Sessions |
| **Column** | Tunable | Very High | Low | Excellent (writes) | Analytics, IoT |
| **Graph** | Strong | Moderate | High (graphs) | Good (traversals) | Social, Recommendations |

### Decision Tree

```
Do you need ACID transactions?
├── Yes ──► Relational Database
│           ├── Complex queries? ──► PostgreSQL
│           └── High performance? ──► MySQL
│
└── No ──► What's your data model?
    ├── Documents ──► MongoDB/CouchDB
    ├── Key-Value ──► Redis/DynamoDB
    ├── Time-series ──► Cassandra/InfluxDB
    ├── Graphs ──► Neo4j/Amazon Neptune
    └── Analytics ──► Elasticsearch/BigQuery
```

### Real-World Examples

#### E-commerce Platform
```
┌─────────────────────────────────────┐
│           E-commerce System         │
├─────────────────────────────────────┤
│ • User accounts ──► PostgreSQL      │
│ • Product catalog ──► MongoDB       │
│ • Shopping carts ──► Redis          │
│ • Order history ──► MySQL           │
│ • Search ──► Elasticsearch          │
│ • Recommendations ──► Neo4j         │
│ • Analytics ──► Cassandra           │
└─────────────────────────────────────┘
```

#### Social Media Platform
```
┌─────────────────────────────────────┐
│          Social Media System        │
├─────────────────────────────────────┤
│ • User profiles ──► MongoDB         │
│ • Friendships ──► Neo4j             │
│ • Posts/Comments ──► Cassandra      │
│ • Real-time feeds ──► Redis         │
│ • Media metadata ──► PostgreSQL     │
│ • Search ──► Elasticsearch          │
└─────────────────────────────────────┘
```

---

## Storage Patterns

### CQRS (Command Query Responsibility Segregation)

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Commands  │───▶│ Command DB  │───▶│   Events    │
│  (Writes)   │    │ (Optimized  │    │             │
│             │    │ for writes) │    │             │
└─────────────┘    └─────────────┘    └──────┬──────┘
                                             │
                                             ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Queries   │◄───│  Query DB   │◄───│  Projector  │
│   (Reads)   │    │ (Optimized  │    │             │
│             │    │ for reads)  │    │             │
└─────────────┘    └─────────────┘    └─────────────┘
```

**Benefits:**
- Optimized read and write models
- Independent scaling
- Complex query support

**Trade-offs:**
- Increased complexity
- Eventual consistency
- Synchronization challenges

### Event Sourcing

```
Events Stream:
UserCreated(id=123, name="Alice")
EmailUpdated(id=123, email="alice@example.com")
ProfileUpdated(id=123, bio="Software engineer")

Current State = Replay all events
```

**Benefits:**
- Complete audit trail
- Temporal queries
- Easy debugging
- Rollback capabilities

**Trade-offs:**
- Storage overhead
- Complex queries
- Event schema evolution

### Data Lake vs Data Warehouse

#### Data Lake
```
┌─────────────────────────────────────┐
│             Data Lake               │
├─────────────────────────────────────┤
│ Raw Data:                          │
│ • JSON logs                        │
│ • CSV files                        │
│ • Images                           │
│ • Videos                           │
│ • IoT sensor data                  │
│                                    │
│ Schema-on-Read                     │
└─────────────────────────────────────┘
```

#### Data Warehouse
```
┌─────────────────────────────────────┐
│           Data Warehouse            │
├─────────────────────────────────────┤
│ Structured Data:                   │
│ • Cleansed                         │
│ • Transformed                      │
│ • Aggregated                       │
│ • Optimized for queries            │
│                                    │
│ Schema-on-Write                    │
└─────────────────────────────────────┘
```

---

## Interview Questions

### Basic Level

**Q1: Explain ACID properties with examples.**

**Answer:**
- **Atomicity**: All operations in a transaction succeed or all fail. Example: Transferring money between accounts - both debit and credit must succeed.
- **Consistency**: Database remains in valid state. Example: Foreign key constraints prevent orphaned records.
- **Isolation**: Concurrent transactions don't interfere. Example: Two users updating inventory don't see each other's uncommitted changes.
- **Durability**: Committed data survives system failures. Example: Completed purchase remains recorded even if server crashes.

**Q2: When would you use NoSQL over SQL databases?**

**Answer:**
Use NoSQL when:
- **Flexible schema**: Product catalogs with varying attributes
- **Horizontal scaling**: Need to handle millions of users
- **High write throughput**: Logging, analytics, IoT data
- **Simple queries**: Key-value lookups, document retrieval
- **Big data**: Storing large volumes of unstructured data
- **Rapid development**: Prototyping with changing requirements

**Q3: What's the difference between sharding and replication?**

**Answer:**
- **Replication**: Copying the same data to multiple servers for availability and read scaling. All replicas contain the same data.
- **Sharding**: Splitting data across multiple servers based on a key. Each shard contains different data.
- **Combined approach**: Often used together - each shard is replicated for high availability.

### Intermediate Level

**Q4: Design a database schema for a social media platform.**

**Answer Framework:**
1. **Core Entities**:
   ```sql
   Users: user_id, username, email, created_at
   Posts: post_id, user_id, content, timestamp
   Follows: follower_id, following_id, created_at
   Likes: user_id, post_id, timestamp
   Comments: comment_id, post_id, user_id, content, timestamp
   ```

2. **Scaling Considerations**:
   - Shard users by user_id or geographic region
   - Separate tables for different post types
   - Denormalize for newsfeed generation
   - Use NoSQL for activity feeds

3. **Indexing Strategy**:
   - Index on user_id for user-specific queries
   - Composite index on (post_id, timestamp) for comments
   - Index on follower_id for feed generation

**Q5: How would you scale a database that's running out of capacity?**

**Answer Framework:**
1. **Immediate Solutions**:
   - Add read replicas for read-heavy workloads
   - Implement caching (Redis) for frequently accessed data
   - Optimize queries and add indexes

2. **Medium-term Solutions**:
   - Vertical scaling (larger instance)
   - Archive old data to cheaper storage
   - Implement connection pooling

3. **Long-term Solutions**:
   - Database sharding based on access patterns
   - Move to microservices with separate databases
   - Consider NoSQL alternatives for specific use cases

### Advanced Level

**Q6: Design a globally distributed database system for an e-commerce platform.**

**Answer Framework:**
1. **Data Categorization**:
   - **User data**: Partition by user_id, replicate regionally
   - **Product catalog**: Replicate globally, eventual consistency OK
   - **Inventory**: Strong consistency required, partition by product/region
   - **Orders**: Strong consistency, partition by user or time

2. **Regional Distribution**:
   ```
   US Region: Primary for US users
   EU Region: Primary for EU users, GDPR compliance
   Asia Region: Primary for Asia users
   ```

3. **Consistency Strategy**:
   - **Strong**: Payment processing, inventory updates
   - **Eventual**: Product descriptions, reviews
   - **Session**: Shopping cart, user preferences

4. **Implementation**:
   - Multi-master replication with conflict resolution
   - Read replicas in each region
   - Global load balancer with geographic routing
   - Eventual consistency with compensation transactions

**Q7: How would you migrate from a monolithic database to microservices with separate databases?**

**Answer Framework:**
1. **Migration Strategy**:
   - **Strangler Fig Pattern**: Gradually replace functionality
   - **Database-per-Service**: Each microservice owns its data
   - **Saga Pattern**: Handle distributed transactions

2. **Implementation Steps**:
   ```
   Phase 1: Create microservices with shared database
   Phase 2: Extract specific tables to microservice databases  
   Phase 3: Implement event-driven communication
   Phase 4: Remove shared database dependencies
   ```

3. **Data Consistency**:
   - Event sourcing for audit trail
   - CQRS for read/write optimization
   - Eventual consistency between services
   - Compensating transactions for rollbacks

4. **Challenges & Solutions**:
   - **Cross-service queries**: Implement API composition or CQRS
   - **Transactions**: Use saga pattern or event sourcing
   - **Data migration**: Blue-green deployment with sync mechanisms

---

**Next:** Continue to [Caching and Async Processing](../04-caching-async/README.md) to learn about improving system performance and handling asynchronous workloads.