# Foundational Concepts 🏗️

Understanding the core principles of system design is crucial for building scalable, reliable, and performant systems.

## Table of Contents

- [Introduction to System Design](#introduction-to-system-design)
- [Scalability and Performance](#scalability-and-performance)
- [Latency vs Throughput](#latency-vs-throughput)
- [Architectural Patterns Overview](#architectural-patterns-overview)
- [Key Terms and Definitions](#key-terms-and-definitions)
- [Interview Questions](#interview-questions)

---

## Introduction to System Design

### What is System Design?

System design is the process of defining the architecture, components, modules, interfaces, and data for a system to satisfy specified requirements. It involves making high-level decisions about:

- **Architecture**: How components interact
- **Data Storage**: Where and how data is stored
- **Scalability**: How the system handles growth
- **Reliability**: How the system handles failures
- **Security**: How the system protects data

### Why System Design Matters

```
Business Requirements
        ↓
System Design Decisions
        ↓
Technical Implementation
        ↓
User Experience & Business Success
```

### The System Design Process

1. **Requirements Gathering**
   - Functional requirements (what the system should do)
   - Non-functional requirements (performance, scalability, reliability)
   - Constraints (budget, time, existing systems)

2. **High-Level Design**
   - System architecture
   - Component identification
   - Data flow

3. **Detailed Design**
   - API design
   - Database schema
   - Algorithm specifications

4. **Scale and Optimize**
   - Identify bottlenecks
   - Plan for growth
   - Optimize critical paths

---

## Scalability and Performance

### Types of Scaling

#### Vertical Scaling (Scale Up)
Adding more power to existing machines.

**Pros:**
- Simple implementation
- No code changes needed
- Maintains data consistency

**Cons:**
- Hardware limits
- Single point of failure
- Expensive at scale

**Example:**
```
Before: 4 CPU cores, 8GB RAM
After:  16 CPU cores, 64GB RAM
```

#### Horizontal Scaling (Scale Out)
Adding more machines to the pool of resources.

**Pros:**
- No upper limit
- Better fault tolerance
- Cost-effective

**Cons:**
- Increased complexity
- Data consistency challenges
- Network overhead

**Example:**
```
Before: 1 server handling 1000 users
After:  10 servers each handling 100 users
```

### Performance Metrics

#### Key Performance Indicators (KPIs)

1. **Response Time**: Time to process a single request
2. **Throughput**: Number of requests processed per unit time
3. **Resource Utilization**: CPU, memory, disk, network usage
4. **Error Rate**: Percentage of failed requests
5. **Availability**: System uptime percentage

#### Performance Optimization Strategies

```
┌─────────────────────────────────────────┐
│               Optimization               │
├─────────────────────────────────────────┤
│ 1. Database Optimization                │
│    • Indexing                          │
│    • Query optimization                │
│    • Connection pooling                │
│                                        │
│ 2. Caching                             │
│    • Application-level cache           │
│    • Database query cache              │
│    • CDN for static content            │
│                                        │
│ 3. Load Distribution                   │
│    • Load balancers                    │
│    • Horizontal partitioning           │
│    • Microservices                     │
│                                        │
│ 4. Asynchronous Processing             │
│    • Message queues                    │
│    • Background jobs                   │
│    • Event-driven architecture         │
└─────────────────────────────────────────┘
```

---

## Latency vs Throughput

### Latency
**Definition**: The time it takes to process a single request.

**Analogy**: Time for one car to travel from point A to point B.

**Measurement**: Usually measured in milliseconds (ms)

**Examples:**
- Database query: 50ms
- API call: 200ms
- File read: 10ms

### Throughput
**Definition**: The number of requests processed per unit of time.

**Analogy**: Number of cars that can pass through a toll booth per hour.

**Measurement**: Requests per second (RPS) or Transactions per second (TPS)

**Examples:**
- Web server: 1000 RPS
- Database: 500 TPS
- Message queue: 10,000 messages/second

### The Relationship

```
┌─────────────────────────────────────────┐
│          Latency vs Throughput          │
├─────────────────────────────────────────┤
│                                        │
│    High Throughput ──────────────┐     │
│         │                        │     │
│         │                        │     │
│         │        Sweet Spot      │     │
│         │                        │     │
│         │                        │     │
│         └────────────── High Latency   │
│      Low Latency                       │
│                                        │
└─────────────────────────────────────────┘
```

**Key Points:**
- You can have high throughput with high latency
- You can have low latency with low throughput  
- The goal is often to optimize both
- Trade-offs exist between the two

### Real-World Examples

| System | Latency Priority | Throughput Priority |
|--------|------------------|-------------------|
| Gaming | ✅ Critical | ⚠️ Moderate |
| Video Streaming | ⚠️ Important | ✅ Critical |
| Financial Trading | ✅ Critical | ✅ Critical |
| Batch Processing | ⚠️ Less Important | ✅ Critical |

---

## Architectural Patterns Overview

### Layered Architecture (N-Tier)

```
┌─────────────────────────┐
│    Presentation Layer   │  ← User Interface
├─────────────────────────┤
│     Business Layer      │  ← Business Logic
├─────────────────────────┤
│   Persistence Layer     │  ← Data Access
├─────────────────────────┤
│     Database Layer      │  ← Data Storage
└─────────────────────────┘
```

**Pros:**
- Clear separation of concerns
- Easy to understand and maintain
- Good for traditional web applications

**Cons:**
- Can become monolithic
- Performance overhead
- Tight coupling between layers

### Service-Oriented Architecture (SOA)

```
┌─────────┐    ┌─────────┐    ┌─────────┐
│Service A│    │Service B│    │Service C│
│         │    │         │    │         │
└─────┬───┘    └───┬─────┘    └───┬─────┘
      │            │              │
      └────────────┼──────────────┘
                   │
            ┌──────┴──────┐
            │Service Bus  │
            └─────────────┘
```

**Pros:**
- Reusability
- Flexibility
- Technology diversity

**Cons:**
- Network overhead
- Complex service management
- Performance concerns

### Microservices Architecture

```
┌─────────┐  ┌─────────┐  ┌─────────┐
│Service A│  │Service B│  │Service C│
│+ DB A   │  │+ DB B   │  │+ DB C   │
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     └────────────┼────────────┘
                  │
           ┌──────┴──────┐
           │   Gateway   │
           └─────────────┘
```

**Pros:**
- Independent deployment
- Technology diversity
- Fault isolation
- Scalability

**Cons:**
- Distributed system complexity
- Network latency
- Data consistency challenges
- Operational overhead

### Event-Driven Architecture

```
┌─────────┐    ┌───────────────┐    ┌─────────┐
│Producer │───▶│  Event Bus    │───▶│Consumer │
└─────────┘    │               │    └─────────┘
               │  ┌─────────┐  │    ┌─────────┐
               │  │ Events  │  │───▶│Consumer │
               │  │ Queue   │  │    └─────────┘
               │  └─────────┘  │    ┌─────────┐
               └───────────────┘───▶│Consumer │
                                   └─────────┘
```

**Pros:**
- Loose coupling
- Scalability
- Flexibility
- Real-time processing

**Cons:**
- Complex debugging
- Event ordering issues
- Eventual consistency
- Message delivery guarantees

---

## Key Terms and Definitions

### Scalability Terms
- **Horizontal Scaling**: Adding more servers
- **Vertical Scaling**: Adding more power to existing servers
- **Load Balancer**: Distributes incoming requests across multiple servers
- **Auto-scaling**: Automatically adjusting resources based on demand

### Performance Terms
- **Latency**: Time to process a single request
- **Throughput**: Number of requests processed per unit time
- **Bandwidth**: Maximum data transfer rate
- **Bottleneck**: The limiting factor in system performance

### Reliability Terms
- **High Availability**: System remains operational most of the time
- **Fault Tolerance**: System continues operating despite failures
- **Redundancy**: Having backup components or systems
- **Disaster Recovery**: Plan for recovering from major failures

### Data Terms
- **ACID**: Atomicity, Consistency, Isolation, Durability
- **BASE**: Basically Available, Soft state, Eventual consistency
- **Sharding**: Partitioning data across multiple databases
- **Replication**: Copying data to multiple locations

---

## Interview Questions

### Basic Level

**Q1: What is the difference between scalability and performance?**

**Answer:**
- **Performance** refers to how fast a system responds to requests (latency and throughput)
- **Scalability** refers to how well a system can handle increased load by adding resources
- A system can be performant but not scalable (fast for small loads, slow for large loads)
- A system can be scalable but not performant (handles large loads but with high latency)

**Q2: When would you choose vertical scaling over horizontal scaling?**

**Answer:**
Vertical scaling is preferred when:
- Application is not designed for distributed computing
- Data consistency is critical and hard to achieve in distributed systems
- Quick solution is needed with minimal code changes
- Working with legacy systems
- Cost is less important than simplicity

**Q3: Explain the difference between latency and throughput.**

**Answer:**
- **Latency**: Time taken for a single request to be processed (e.g., 100ms)
- **Throughput**: Number of requests processed per unit time (e.g., 1000 requests/second)
- They're not directly correlated - you can have high throughput with high latency
- Example: A highway can move many cars per hour (high throughput) but each car takes a long time to reach destination due to traffic (high latency)

### Intermediate Level

**Q4: How would you design a system to handle 1 million concurrent users?**

**Answer Framework:**
1. **Load Distribution**: Use load balancers to distribute traffic
2. **Horizontal Scaling**: Deploy multiple application servers
3. **Database Scaling**: Use read replicas, sharding, or NoSQL databases
4. **Caching**: Implement multiple cache layers (CDN, application cache, database cache)
5. **Asynchronous Processing**: Use message queues for non-critical operations
6. **Monitoring**: Implement comprehensive monitoring and alerting

**Q5: What factors would you consider when choosing between different architectural patterns?**

**Answer:**
Consider these factors:
- **Team Size and Expertise**: Microservices require more DevOps skills
- **System Complexity**: Simple systems may not need microservices
- **Scalability Requirements**: Different patterns scale differently
- **Development Speed**: Monoliths are faster to develop initially
- **Deployment Frequency**: Microservices enable independent deployments
- **Technology Constraints**: Existing systems and team preferences
- **Performance Requirements**: Network latency in distributed systems

### Advanced Level

**Q6: Design a system that needs to process 100,000 transactions per second with sub-100ms latency.**

**Answer Framework:**
1. **Architecture**: Event-driven microservices
2. **Database**: In-memory databases (Redis) + eventual persistence
3. **Load Balancing**: Multiple layers with geographic distribution
4. **Caching**: Multi-level caching strategy
5. **Network**: Optimized protocols (binary protocols, connection pooling)
6. **Hardware**: SSD storage, high-performance CPUs, optimized network
7. **Monitoring**: Real-time performance tracking

**Q7: How would you handle the trade-offs between consistency and availability in a distributed system?**

**Answer:**
1. **Understand CAP Theorem**: Can't have all three (Consistency, Availability, Partition Tolerance)
2. **Business Requirements**: Determine what matters most for your use case
3. **Strategies**:
   - **CP System**: Banking systems (consistency over availability)
   - **AP System**: Social media feeds (availability over consistency)
   - **Eventual Consistency**: Many distributed systems use this approach
4. **Implementation**:
   - Use consensus algorithms (Raft, PBFT)
   - Implement conflict resolution strategies
   - Design for graceful degradation

---

**Next:** Continue to [Availability and Consistency](../02-availability-consistency/README.md) to learn about building reliable distributed systems.