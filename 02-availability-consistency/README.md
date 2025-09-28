# Availability and Consistency 🔄

Understanding the trade-offs between availability, consistency, and partition tolerance is crucial for designing robust distributed systems.

## Table of Contents

- [Availability Patterns](#availability-patterns)
- [Replication Strategies](#replication-strategies)
- [Consistency Models](#consistency-models)
- [CAP Theorem](#cap-theorem)
- [PACELC Theorem](#pacelc-theorem)
- [Real-World Applications](#real-world-applications)
- [Interview Questions](#interview-questions)

---

## Availability Patterns

### What is Availability?

Availability is the percentage of time a system remains operational under normal circumstances.

**Formula:**
```
Availability = (Total Time - Downtime) / Total Time × 100%
```

### Availability Levels (SLA Standards)

| Availability | Downtime/Year | Downtime/Month | Downtime/Week | Use Case |
|--------------|---------------|----------------|---------------|----------|
| 90% | 36.5 days | 3 days | 16.8 hours | Basic service |
| 95% | 18.25 days | 1.5 days | 8.4 hours | Standard web apps |
| 99% | 3.65 days | 7.2 hours | 1.68 hours | Important services |
| 99.9% | 8.76 hours | 43.2 minutes | 10.1 minutes | Critical services |
| 99.99% | 52.56 minutes | 4.32 minutes | 1.01 minutes | High availability |
| 99.999% | 5.26 minutes | 25.9 seconds | 6.05 seconds | Ultra-high availability |

### High Availability Patterns

#### 1. Failover Pattern

```
Primary Server ──────┐
                     │
                     ▼
               Health Check
                     │
                     ▼
               ┌──────────┐
               │   Load   │
               │ Balancer │
               └──────────┘
                     │
                     ▼
Secondary Server ────┘
```

**Active-Passive Failover:**
- Primary server handles all traffic
- Secondary server is on standby
- Switch occurs when primary fails

**Active-Active Failover:**
- Both servers handle traffic
- Load is distributed between them
- Better resource utilization

#### 2. Redundancy Pattern

```
┌─────────────────────────────────────┐
│            Application              │
├─────────────────────────────────────┤
│  Server 1  │  Server 2  │  Server 3 │
├─────────────────────────────────────┤
│    DB 1    │    DB 2    │    DB 3   │
└─────────────────────────────────────┘
```

**Benefits:**
- No single point of failure
- Graceful degradation
- Improved performance through load distribution

#### 3. Circuit Breaker Pattern

```
┌─────────┐     ┌─────────────┐     ┌─────────┐
│ Client  │────▶│   Circuit   │────▶│ Service │
└─────────┘     │   Breaker   │     └─────────┘
                └─────────────┘
                      │
                      ▼
                ┌─────────────┐
                │   Fallback  │
                │   Response  │
                └─────────────┘
```

**States:**
- **Closed**: Normal operation, requests pass through
- **Open**: Service unavailable, return fallback response
- **Half-Open**: Testing if service has recovered

---

## Replication Strategies

### Master-Slave Replication

```
                    ┌──────────┐
                    │  Master  │ ← Writes
                    │    DB    │
                    └─────┬────┘
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
        ┌─────────┐ ┌─────────┐ ┌─────────┐
        │ Slave 1 │ │ Slave 2 │ │ Slave 3 │ ← Reads
        └─────────┘ └─────────┘ └─────────┘
```

**Pros:**
- Read scalability
- Data backup
- Simple to implement

**Cons:**
- Single point of failure (master)
- Replication lag
- Complex failover process

### Master-Master Replication

```
        ┌─────────────┐ ◄──────► ┌─────────────┐
        │   Master 1  │          │   Master 2  │
        │             │ ◄──────► │             │
        └─────────────┘          └─────────────┘
              │                        │
              ▼                        ▼
        ┌─────────────┐          ┌─────────────┐
        │   Slave A   │          │   Slave B   │
        └─────────────┘          └─────────────┘
```

**Pros:**
- High availability
- Load distribution
- Geographic distribution

**Cons:**
- Complex conflict resolution
- Consistency challenges
- Network partitioning issues

### Asynchronous vs Synchronous Replication

#### Asynchronous Replication
```
Client ───► Master ───► Response
                │
                └─(async)─► Slaves
```

**Pros:**
- Better performance
- Network fault tolerance
- Lower latency

**Cons:**
- Potential data loss
- Eventual consistency
- Replication lag

#### Synchronous Replication
```
Client ───► Master ───► Slaves ───► Confirmation ───► Response
```

**Pros:**
- Strong consistency
- No data loss
- Immediate consistency

**Cons:**
- Higher latency
- Reduced availability
- Network dependency

---

## Consistency Models

### Strong Consistency

All nodes see the same data at the same time.

```
Time: T1        T2        T3
Node A: 100 ──► 150 ──► 150
Node B: 100 ──► 150 ──► 150
Node C: 100 ──► 150 ──► 150
```

**Characteristics:**
- Immediate consistency across all nodes
- Sacrifices availability for consistency
- Used in financial systems

**Implementation:**
- Two-phase commit (2PC)
- Three-phase commit (3PC)
- Consensus algorithms (Raft, PBFT)

### Eventual Consistency

All nodes will eventually converge to the same state.

```
Time: T1        T2        T3        T4
Node A: 100 ──► 150 ──► 150 ──► 150
Node B: 100 ──► 100 ──► 150 ──► 150
Node C: 100 ──► 100 ──► 100 ──► 150
```

**Characteristics:**
- Temporary inconsistencies allowed
- Better availability and performance
- Suitable for social media, DNS

**Examples:**
- DNS propagation
- Amazon DynamoDB
- Cassandra

### Weak Consistency

No guarantees about when all nodes will be consistent.

**Characteristics:**
- Best performance and availability
- Application must handle inconsistencies
- Used in gaming, real-time systems

### Consistency Patterns

#### Read-Your-Writes Consistency
```
User A writes ───► Database
         │              │
         └──► Read ◄────┘
            (sees own writes)
```

#### Session Consistency
```
User Session ───► Database
                    │
      ┌─────────────┼─────────────┐
      │             │             │
   Read 1        Read 2        Read 3
(consistent within session)
```

#### Monotonic Read Consistency
```
Time: T1 ──► T2 ──► T3
Read: V1 ──► V2 ──► V3
      (V1 ≤ V2 ≤ V3)
```

---

## CAP Theorem

The CAP theorem states that in a distributed system, you can only guarantee two out of three properties:

### The Three Properties

#### Consistency (C)
All nodes see the same data simultaneously.

#### Availability (A)  
System remains operational (available for read/write).

#### Partition Tolerance (P)
System continues operating despite network failures.

### CAP Triangle

```
            Consistency
                 │
                 │
                 │
    CA ──────────┼────────── CP
                 │
                 │
                 │
              Partition
             Tolerance
                 │
                 │
                 │
                AP
```

### System Examples

#### CA Systems (Consistency + Availability)
- **Examples**: Traditional RDBMS (MySQL, PostgreSQL)
- **Trade-off**: Cannot handle network partitions
- **Use Case**: Single data center applications

#### CP Systems (Consistency + Partition Tolerance)
- **Examples**: MongoDB, HBase, Redis
- **Trade-off**: May become unavailable during partitions
- **Use Case**: Banking systems, inventory management

#### AP Systems (Availability + Partition Tolerance)
- **Examples**: Cassandra, DynamoDB, CouchDB
- **Trade-off**: May return stale data
- **Use Case**: Social media, content delivery

### Real-World CAP Decisions

```
┌─────────────────────────────────────────────────┐
│                System Examples                  │
├─────────────────┬───────────────┬───────────────┤
│       CA        │      CP       │      AP       │
├─────────────────┼───────────────┼───────────────┤
│ • PostgreSQL    │ • MongoDB     │ • Cassandra   │
│ • MySQL         │ • HBase       │ • DynamoDB    │
│ • Oracle DB     │ • Redis       │ • Riak        │
│ • SQL Server    │ • Etcd        │ • CouchDB     │
└─────────────────┴───────────────┴───────────────┘
```

---

## PACELC Theorem

PACELC extends CAP theorem to consider latency during normal operation.

### The Extended Properties

**In case of Partition (P):**
- Choose between **Availability (A)** and **Consistency (C)**

**Else (E):**
- Choose between **Latency (L)** and **Consistency (C)**

### PACELC Classification

```
┌─────────────────────────────────────────────────┐
│              PACELC Examples                    │
├─────────────────┬───────────────────────────────┤
│   PA/EL         │        PA/EC                  │
│ (Availability   │   (Availability during        │
│  and Latency)   │    partition, Consistency     │
│                 │    during normal operation)   │
├─────────────────┼───────────────────────────────┤
│ • Cassandra     │ • MongoDB                     │
│ • DynamoDB      │ • PNUTS                       │
│ • Riak          │                               │
├─────────────────┼───────────────────────────────┤
│   PC/EL         │        PC/EC                  │
│ (Consistency    │   (Always Consistency)        │
│  during partition,│                             │
│  Latency during │                               │
│  normal ops)    │                               │
├─────────────────┼───────────────────────────────┤
│ • PNUTS         │ • Traditional RDBMS           │
│                 │ • HBase                       │
│                 │ • Megastore                   │
└─────────────────┴───────────────────────────────┘
```

---

## Real-World Applications

### E-commerce Platform

**Scenario**: Online shopping cart

**Requirements:**
- Cart updates must be consistent
- Product catalog can be eventually consistent
- High availability during sales

**Solution:**
```
┌─────────────────┐    ┌─────────────────┐
│   Cart Service  │    │ Catalog Service │
│      (CP)       │    │      (AP)       │
│                 │    │                 │
│ • Strong        │    │ • Eventual      │
│   consistency   │    │   consistency   │
│ • Immediate     │    │ • High          │
│   updates       │    │   availability  │
└─────────────────┘    └─────────────────┘
```

### Social Media Platform

**Scenario**: Facebook-like newsfeed

**Requirements:**
- Posts must be highly available
- Slight delays in consistency acceptable
- Global distribution needed

**Solution**: AP system with eventual consistency
```
User Posts ──► Regional Data Centers ──► Global Sync
            (Immediate availability)   (Eventual consistency)
```

### Banking System

**Scenario**: Money transfer

**Requirements:**
- Account balances must be accurate
- No double-spending allowed
- Strong consistency required

**Solution**: CP system with strong consistency
```
Transfer Request ──► Validation ──► Two-Phase Commit ──► Confirmation
```

---

## Interview Questions

### Basic Level

**Q1: Explain the CAP theorem in simple terms.**

**Answer:**
The CAP theorem states that in a distributed system, you can only guarantee 2 out of 3 properties:
- **Consistency**: All nodes see the same data at the same time
- **Availability**: System remains operational for read/write operations  
- **Partition Tolerance**: System works despite network failures between nodes

Real-world example: During a network partition between data centers, you must choose between keeping the system available (potentially serving stale data) or maintaining consistency (potentially becoming unavailable).

**Q2: What's the difference between strong and eventual consistency?**

**Answer:**
- **Strong Consistency**: All nodes see the same data immediately. Example: Banking systems where account balances must be accurate across all nodes instantly.
- **Eventual Consistency**: Nodes will eventually converge to the same state, but may temporarily show different values. Example: DNS updates that propagate across the internet over time.

**Q3: When would you choose availability over consistency?**

**Answer:**
Choose availability over consistency when:
- User experience is more important than perfect data accuracy
- Temporary inconsistencies are acceptable (social media feeds, comments)
- System must remain operational during network failures
- Revenue loss from downtime exceeds cost of occasional inconsistency
- Examples: Social media platforms, content delivery networks, shopping catalogs

### Intermediate Level

**Q4: Design a system that needs both high consistency for payments and high availability for product browsing.**

**Answer Framework:**
1. **Separate Services**: Different consistency requirements call for different architectures
2. **Payment Service (CP)**: 
   - Strong consistency with ACID transactions
   - Two-phase commit for distributed transactions
   - May become unavailable during network partitions
3. **Product Catalog (AP)**:
   - Eventual consistency acceptable
   - High availability with multiple replicas
   - CDN for global distribution
4. **Implementation**:
   - Microservices architecture
   - Different databases for different services
   - API gateway to route requests appropriately

**Q5: How would you implement eventual consistency in a distributed system?**

**Answer:**
1. **Asynchronous Replication**:
   - Write to primary node first
   - Replicate to other nodes asynchronously
   - Handle replication lag gracefully

2. **Conflict Resolution**:
   - Last-write-wins timestamps
   - Vector clocks for causality
   - Application-level conflict resolution

3. **Implementation Strategies**:
   - Message queues for async updates
   - Event sourcing for audit trail
   - Read-repair during queries
   - Anti-entropy processes for consistency checking

### Advanced Level

**Q6: Explain PACELC theorem and how it extends CAP theorem.**

**Answer:**
PACELC extends CAP by considering the trade-off during normal operation (no partition):

**During Partition**: Choose between Availability (A) and Consistency (C) - same as CAP
**Else (normal operation)**: Choose between Latency (L) and Consistency (C)

**Examples:**
- **PA/EL** (Cassandra): Chooses availability during partition, low latency during normal operation
- **PC/EC** (Traditional RDBMS): Chooses consistency during partition and normal operation
- **PA/EC** (MongoDB): Chooses availability during partition, consistency during normal operation

This helps make more nuanced decisions about system design.

**Q7: Design a global chat application considering consistency and availability requirements.**

**Answer Framework:**
1. **Message Ordering**: 
   - Eventual consistency for message delivery
   - Vector clocks for causal ordering
   - Local ordering within chat rooms

2. **Architecture**:
   - Regional clusters for low latency
   - AP system for high availability
   - Conflict-free replicated data types (CRDTs)

3. **Specific Designs**:
   - **Message Storage**: Eventually consistent, partition-tolerant
   - **User Presence**: Highly available, eventually consistent
   - **User Authentication**: Strongly consistent, may sacrifice availability
   - **File Sharing**: Eventually consistent with conflict resolution

4. **Implementation**:
   - WebSocket connections to regional servers
   - Message queues for cross-region synchronization
   - Operational transformation for conflict resolution

---

**Next:** Continue to [Database and Storage](../03-database-storage/README.md) to learn about data persistence strategies and database design patterns.