# Resiliency and System Essentials ⚡

Building robust, fault-tolerant systems that can handle failures gracefully and scale reliably.

## Table of Contents

- [Load Balancers](#load-balancers)
- [Circuit Breakers](#circuit-breakers)
- [Consistent Hashing](#consistent-hashing)
- [Health Checks and Monitoring](#health-checks-and-monitoring)
- [Retry Strategies](#retry-strategies)
- [Rate Limiting](#rate-limiting)
- [Networking Essentials](#networking-essentials)
- [Disaster Recovery](#disaster-recovery)
- [Interview Questions](#interview-questions)

---

## Load Balancers

### What is Load Balancing?

Load balancing distributes incoming requests across multiple servers to prevent overloading any single server and improve system reliability.

```
                    ┌─────────────┐
                    │   Clients   │
                    └─────┬───────┘
                          │
                          ▼
                    ┌─────────────┐
                    │Load Balancer│
                    └─────┬───────┘
                          │
           ┌──────────────┼──────────────┐
           ▼              ▼              ▼
    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
    │  Server 1   │ │  Server 2   │ │  Server 3   │
    └─────────────┘ └─────────────┘ └─────────────┘
```

### Load Balancing Algorithms

#### 1. Round Robin
```python
class RoundRobinLoadBalancer:
    def __init__(self, servers):
        self.servers = servers
        self.current_index = 0
    
    def get_server(self):
        server = self.servers[self.current_index]
        self.current_index = (self.current_index + 1) % len(self.servers)
        return server

# Usage
lb = RoundRobinLoadBalancer(['server1:80', 'server2:80', 'server3:80'])
print(lb.get_server())  # server1:80
print(lb.get_server())  # server2:80
print(lb.get_server())  # server3:80
print(lb.get_server())  # server1:80 (wraps around)
```

**Pros**: Simple, fair distribution
**Cons**: Doesn't consider server capacity or current load

#### 2. Weighted Round Robin
```python
class WeightedRoundRobinLoadBalancer:
    def __init__(self, servers_weights):
        # servers_weights: [(server, weight), ...]
        self.servers_weights = servers_weights
        self.weighted_list = self._create_weighted_list()
        self.current_index = 0
    
    def _create_weighted_list(self):
        weighted_list = []
        for server, weight in self.servers_weights:
            weighted_list.extend([server] * weight)
        return weighted_list
    
    def get_server(self):
        server = self.weighted_list[self.current_index]
        self.current_index = (self.current_index + 1) % len(self.weighted_list)
        return server

# Usage - server1 gets 50% traffic, server2 gets 30%, server3 gets 20%
lb = WeightedRoundRobinLoadBalancer([
    ('server1:80', 5),  # Weight 5
    ('server2:80', 3),  # Weight 3  
    ('server3:80', 2)   # Weight 2
])
```

**Pros**: Accounts for server capacity differences
**Cons**: Static weights, doesn't adapt to actual server performance

#### 3. Least Connections
```python
import heapq

class LeastConnectionsLoadBalancer:
    def __init__(self, servers):
        # Use min-heap to track connections: [(connections, server)]
        self.servers_heap = [(0, server) for server in servers]
        heapq.heapify(self.servers_heap)
        self.server_connections = {server: 0 for server in servers}
    
    def get_server(self):
        # Get server with least connections
        connections, server = heapq.heappop(self.servers_heap)
        
        # Increment connection count
        self.server_connections[server] += 1
        
        # Put back in heap with updated count
        heapq.heappush(self.servers_heap, (connections + 1, server))
        
        return server
    
    def release_connection(self, server):
        # Called when connection ends
        if self.server_connections[server] > 0:
            self.server_connections[server] -= 1
```

**Pros**: Adapts to actual server load
**Cons**: Requires connection state tracking

#### 4. Least Response Time
```python
import time

class LeastResponseTimeLoadBalancer:
    def __init__(self, servers):
        self.servers = servers
        self.response_times = {server: [] for server in servers}  # Sliding window
        self.max_samples = 10  # Keep last 10 response times
    
    def get_server(self):
        best_server = None
        best_avg_time = float('inf')
        
        for server in self.servers:
            avg_time = self._get_average_response_time(server)
            if avg_time < best_avg_time:
                best_avg_time = avg_time
                best_server = server
        
        return best_server
    
    def record_response_time(self, server, response_time):
        times = self.response_times[server]
        times.append(response_time)
        
        # Keep only recent samples
        if len(times) > self.max_samples:
            times.pop(0)
    
    def _get_average_response_time(self, server):
        times = self.response_times[server]
        if not times:
            return 0.1  # Default low value for new servers
        return sum(times) / len(times)
```

**Pros**: Considers actual server performance
**Cons**: More complex, requires response time tracking

#### 5. IP Hash
```python
import hashlib

class IPHashLoadBalancer:
    def __init__(self, servers):
        self.servers = servers
    
    def get_server(self, client_ip):
        # Hash client IP to get consistent server assignment
        hash_value = int(hashlib.md5(client_ip.encode()).hexdigest(), 16)
        server_index = hash_value % len(self.servers)
        return self.servers[server_index]

# Usage
lb = IPHashLoadBalancer(['server1:80', 'server2:80', 'server3:80'])
print(lb.get_server('192.168.1.100'))  # Always same server for this IP
```

**Pros**: Session affinity (sticky sessions)
**Cons**: Uneven distribution if clients aren't evenly distributed

### Types of Load Balancers

#### Layer 4 (Transport Layer) Load Balancer
```python
# Routes based on IP and port information only
# Doesn't inspect packet contents

class L4LoadBalancer:
    def __init__(self, servers):
        self.servers = servers
        self.algorithm = RoundRobinLoadBalancer(servers)
    
    def route_request(self, client_ip, client_port, dest_port):
        # Route based on network info only
        server = self.algorithm.get_server()
        
        # Forward packet to selected server
        return {
            'action': 'forward',
            'destination': server,
            'client_ip': client_ip,
            'client_port': client_port
        }
```

**Pros**: Fast (no content inspection), low latency
**Cons**: Limited routing intelligence, can't route based on content

#### Layer 7 (Application Layer) Load Balancer
```python
# Routes based on application content (HTTP headers, URLs, etc.)

class L7LoadBalancer:
    def __init__(self, server_groups):
        self.server_groups = server_groups  # {'api': [...], 'web': [...]}
    
    def route_request(self, http_request):
        url = http_request['url']
        headers = http_request['headers']
        
        # Route based on URL path
        if url.startswith('/api/'):
            servers = self.server_groups['api']
        elif url.startswith('/admin/'):
            servers = self.server_groups['admin'] 
        else:
            servers = self.server_groups['web']
        
        # Route based on user type
        if 'X-User-Type' in headers and headers['X-User-Type'] == 'premium':
            servers = self.server_groups.get('premium', servers)
        
        # Select server from appropriate group
        lb = RoundRobinLoadBalancer(servers)
        return lb.get_server()
```

**Pros**: Intelligent routing, content-aware decisions
**Cons**: Higher latency (content inspection), more CPU intensive

### Health Checks

```python
import requests
import threading
import time

class HealthChecker:
    def __init__(self, servers, check_interval=30):
        self.servers = servers
        self.check_interval = check_interval
        self.healthy_servers = set(servers)  # Assume all healthy initially
        self.health_check_thread = None
        self.running = False
    
    def start_health_checks(self):
        self.running = True
        self.health_check_thread = threading.Thread(target=self._health_check_loop)
        self.health_check_thread.daemon = True
        self.health_check_thread.start()
    
    def stop_health_checks(self):
        self.running = False
    
    def _health_check_loop(self):
        while self.running:
            for server in self.servers:
                if self._check_server_health(server):
                    self.healthy_servers.add(server)
                else:
                    self.healthy_servers.discard(server)
            
            time.sleep(self.check_interval)
    
    def _check_server_health(self, server):
        try:
            # Simple HTTP health check
            response = requests.get(f'http://{server}/health', timeout=5)
            return response.status_code == 200
        except:
            return False
    
    def get_healthy_servers(self):
        return list(self.healthy_servers)

class HealthAwareLoadBalancer:
    def __init__(self, servers):
        self.all_servers = servers
        self.health_checker = HealthChecker(servers)
        self.health_checker.start_health_checks()
        self.algorithm = RoundRobinLoadBalancer(servers)
    
    def get_server(self):
        healthy_servers = self.health_checker.get_healthy_servers()
        
        if not healthy_servers:
            # Fallback: return any server (better than failing completely)
            return self.all_servers[0]
        
        # Update algorithm with healthy servers
        self.algorithm.servers = healthy_servers
        return self.algorithm.get_server()
```

---

## Circuit Breakers

### What is a Circuit Breaker?

A circuit breaker prevents cascading failures by monitoring service calls and "opening" when failures exceed a threshold, allowing the system to fail fast.

```
Circuit Breaker States:

CLOSED (Normal) ──failure_threshold──► OPEN (Failing)
    ▲                                       │
    │                                       │
    └──success──── HALF_OPEN ◄──timeout────┘
                   (Testing)
```

### Circuit Breaker Implementation

```python
import time
import threading
from enum import Enum
from collections import deque

class CircuitState(Enum):
    CLOSED = "CLOSED"      # Normal operation
    OPEN = "OPEN"         # Failing fast
    HALF_OPEN = "HALF_OPEN"  # Testing recovery

class CircuitBreakerError(Exception):
    pass

class CircuitBreaker:
    def __init__(self, 
                 failure_threshold=5,     # Number of failures to trigger open
                 recovery_timeout=60,     # Seconds before trying half-open
                 success_threshold=3,     # Successes needed to close from half-open
                 timeout=30,              # Request timeout seconds
                 window_size=10):         # Size of sliding window for failure rate
        
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.success_threshold = success_threshold
        self.timeout = timeout
        self.window_size = window_size
        
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None
        
        # Sliding window for failure tracking
        self.call_history = deque(maxlen=window_size)
        
        self._lock = threading.Lock()
    
    def call(self, func, *args, **kwargs):
        with self._lock:
            state = self._get_state()
            
            if state == CircuitState.OPEN:
                raise CircuitBreakerError("Circuit breaker is OPEN")
            
            # CLOSED or HALF_OPEN: attempt the call
            start_time = time.time()
            
            try:
                result = self._execute_with_timeout(func, args, kwargs)
                self._record_success()
                return result
                
            except Exception as e:
                self._record_failure()
                raise e
    
    def _get_state(self):
        if self.state == CircuitState.OPEN:
            # Check if we should transition to HALF_OPEN
            if (self.last_failure_time and 
                time.time() - self.last_failure_time > self.recovery_timeout):
                self.state = CircuitState.HALF_OPEN
                self.success_count = 0
        
        return self.state
    
    def _execute_with_timeout(self, func, args, kwargs):
        # Simple timeout implementation
        import signal
        
        def timeout_handler(signum, frame):
            raise TimeoutError("Function call timed out")
        
        old_handler = signal.signal(signal.SIGALRM, timeout_handler)
        signal.alarm(self.timeout)
        
        try:
            result = func(*args, **kwargs)
            return result
        finally:
            signal.alarm(0)
            signal.signal(signal.SIGALRM, old_handler)
    
    def _record_success(self):
        self.call_history.append(True)  # True = success
        
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.success_threshold:
                # Enough successes to close the circuit
                self.state = CircuitState.CLOSED
                self.failure_count = 0
    
    def _record_failure(self):
        self.call_history.append(False)  # False = failure
        self.last_failure_time = time.time()
        
        if self.state == CircuitState.HALF_OPEN:
            # Any failure in half-open state goes back to open
            self.state = CircuitState.OPEN
            return
        
        # Calculate failure rate in sliding window
        if len(self.call_history) >= self.window_size:
            failure_rate = sum(1 for success in self.call_history if not success) / len(self.call_history)
            
            if failure_rate >= (self.failure_threshold / self.window_size):
                self.state = CircuitState.OPEN
    
    def get_stats(self):
        return {
            'state': self.state.value,
            'failure_count': self.failure_count,
            'success_count': self.success_count,
            'call_history_length': len(self.call_history),
            'last_failure_time': self.last_failure_time
        }

# Usage example
def unreliable_service():
    import random
    if random.random() < 0.7:  # 70% failure rate
        raise Exception("Service failed")
    return "Success"

# Decorator pattern
def circuit_breaker(failure_threshold=3, recovery_timeout=10):
    def decorator(func):
        cb = CircuitBreaker(failure_threshold=failure_threshold, 
                          recovery_timeout=recovery_timeout)
        
        def wrapper(*args, **kwargs):
            return cb.call(func, *args, **kwargs)
        
        wrapper.circuit_breaker = cb  # Expose circuit breaker for monitoring
        return wrapper
    
    return decorator

@circuit_breaker(failure_threshold=3, recovery_timeout=5)
def call_external_api():
    # Simulate external API call
    return unreliable_service()

# Test the circuit breaker
if __name__ == "__main__":
    for i in range(20):
        try:
            result = call_external_api()
            print(f"Call {i+1}: {result}")
        except Exception as e:
            print(f"Call {i+1}: FAILED - {e}")
        
        time.sleep(1)
        
        # Print circuit breaker stats every 5 calls
        if (i + 1) % 5 == 0:
            stats = call_external_api.circuit_breaker.get_stats()
            print(f"Circuit Breaker Stats: {stats}")
```

### Advanced Circuit Breaker Features

#### Bulkhead Pattern
```python
class BulkheadCircuitBreaker:
    """Isolate different types of calls with separate circuit breakers"""
    
    def __init__(self):
        self.circuit_breakers = {
            'database': CircuitBreaker(failure_threshold=3, recovery_timeout=30),
            'external_api': CircuitBreaker(failure_threshold=5, recovery_timeout=60),
            'cache': CircuitBreaker(failure_threshold=10, recovery_timeout=10)
        }
    
    def call(self, service_type, func, *args, **kwargs):
        if service_type not in self.circuit_breakers:
            raise ValueError(f"Unknown service type: {service_type}")
        
        return self.circuit_breakers[service_type].call(func, *args, **kwargs)
    
    def get_all_stats(self):
        return {
            service: cb.get_stats() 
            for service, cb in self.circuit_breakers.items()
        }
```

---

## Consistent Hashing

### What is Consistent Hashing?

Consistent hashing is a technique used to distribute data across nodes in a way that minimizes reorganization when nodes are added or removed.

```
Traditional Hashing Problem:
hash(key) % N  # N = number of servers

When a server is added/removed, most keys need to be remapped!

Consistent Hashing Solution:
- Hash both keys and servers onto a ring
- Each key goes to the next server clockwise on the ring
- Adding/removing servers only affects adjacent keys
```

### Consistent Hashing Implementation

```python
import hashlib
import bisect
from typing import List, Any, Optional

class ConsistentHash:
    def __init__(self, nodes: List[str] = None, replicas: int = 150):
        """
        Initialize consistent hash ring
        
        Args:
            nodes: List of node identifiers
            replicas: Number of virtual nodes per physical node (more = better distribution)
        """
        self.replicas = replicas
        self.ring = {}  # hash -> node mapping
        self.sorted_hashes = []  # Sorted list of hashes for binary search
        
        if nodes:
            for node in nodes:
                self.add_node(node)
    
    def _hash(self, key: str) -> int:
        """Hash function - using MD5 for good distribution"""
        return int(hashlib.md5(key.encode('utf-8')).hexdigest(), 16)
    
    def add_node(self, node: str) -> None:
        """Add a node to the ring"""
        for i in range(self.replicas):
            # Create virtual nodes
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)
            
            self.ring[hash_value] = node
            bisect.insort(self.sorted_hashes, hash_value)
    
    def remove_node(self, node: str) -> None:
        """Remove a node from the ring"""
        for i in range(self.replicas):
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)
            
            if hash_value in self.ring:
                del self.ring[hash_value]
                self.sorted_hashes.remove(hash_value)
    
    def get_node(self, key: str) -> Optional[str]:
        """Get the node responsible for a key"""
        if not self.ring:
            return None
        
        hash_value = self._hash(key)
        
        # Find the first node hash >= our key hash
        index = bisect.bisect_right(self.sorted_hashes, hash_value)
        
        # If we're past the end, wrap around to the beginning
        if index == len(self.sorted_hashes):
            index = 0
        
        node_hash = self.sorted_hashes[index]
        return self.ring[node_hash]
    
    def get_nodes(self, key: str, count: int) -> List[str]:
        """Get multiple nodes for redundancy"""
        if not self.ring or count <= 0:
            return []
        
        hash_value = self._hash(key)
        nodes = []
        seen_nodes = set()
        
        # Find the starting position
        start_index = bisect.bisect_right(self.sorted_hashes, hash_value)
        
        # Collect unique nodes, wrapping around the ring
        for i in range(len(self.sorted_hashes)):
            index = (start_index + i) % len(self.sorted_hashes)
            node_hash = self.sorted_hashes[index]
            node = self.ring[node_hash]
            
            if node not in seen_nodes:
                nodes.append(node)
                seen_nodes.add(node)
                
                if len(nodes) >= count:
                    break
        
        return nodes
    
    def get_stats(self) -> dict:
        """Get statistics about the hash ring"""
        if not self.ring:
            return {'nodes': 0, 'virtual_nodes': 0}
        
        node_counts = {}
        for node in self.ring.values():
            node_counts[node] = node_counts.get(node, 0) + 1
        
        return {
            'nodes': len(set(self.ring.values())),
            'virtual_nodes': len(self.ring),
            'replicas_per_node': self.replicas,
            'node_distribution': node_counts
        }

# Usage example
class DistributedCache:
    def __init__(self, cache_nodes: List[str]):
        self.cache_nodes = {node: {} for node in cache_nodes}  # Simulate cache storage
        self.hash_ring = ConsistentHash(cache_nodes)
    
    def put(self, key: str, value: Any) -> None:
        """Store a key-value pair"""
        node = self.hash_ring.get_node(key)
        if node:
            self.cache_nodes[node][key] = value
            print(f"Stored {key}={value} on node {node}")
    
    def get(self, key: str) -> Any:
        """Retrieve a value by key"""
        node = self.hash_ring.get_node(key)
        if node and key in self.cache_nodes[node]:
            return self.cache_nodes[node][key]
        return None
    
    def put_replicated(self, key: str, value: Any, replication_factor: int = 2) -> None:
        """Store with replication"""
        nodes = self.hash_ring.get_nodes(key, replication_factor)
        for node in nodes:
            self.cache_nodes[node][key] = value
        print(f"Stored {key}={value} on nodes {nodes}")
    
    def get_replicated(self, key: str) -> Any:
        """Retrieve with replication (try multiple nodes)"""
        nodes = self.hash_ring.get_nodes(key, 2)  # Try up to 2 nodes
        for node in nodes:
            if key in self.cache_nodes[node]:
                return self.cache_nodes[node][key]
        return None
    
    def add_node(self, node: str) -> None:
        """Add a new cache node"""
        print(f"Adding node {node}")
        self.cache_nodes[node] = {}
        
        # Store keys that need to move to new node
        keys_to_move = {}
        
        # Check all existing keys to see if they should move
        for existing_node, cache in self.cache_nodes.items():
            if existing_node == node:
                continue
                
            for key in list(cache.keys()):
                # Before adding the new node, check where this key was
                old_responsible_node = self.hash_ring.get_node(key)
                
                # Add the new node
                if node not in [n for n in self.hash_ring.ring.values()]:
                    self.hash_ring.add_node(node)
                
                # Check where this key should be now
                new_responsible_node = self.hash_ring.get_node(key)
                
                # If the key should move to the new node
                if new_responsible_node == node and old_responsible_node != node:
                    keys_to_move[key] = cache[key]
                    del cache[key]
        
        # Move the keys to the new node
        for key, value in keys_to_move.items():
            self.cache_nodes[node][key] = value
            print(f"Moved {key}={value} to new node {node}")
    
    def remove_node(self, node: str) -> None:
        """Remove a cache node"""
        if node not in self.cache_nodes:
            return
        
        print(f"Removing node {node}")
        
        # Move all keys from the removed node to their new locations
        keys_to_redistribute = self.cache_nodes[node].copy()
        del self.cache_nodes[node]
        self.hash_ring.remove_node(node)
        
        for key, value in keys_to_redistribute.items():
            new_node = self.hash_ring.get_node(key)
            if new_node:
                self.cache_nodes[new_node][key] = value
                print(f"Moved {key}={value} from removed node to {new_node}")
    
    def print_distribution(self) -> None:
        """Print current key distribution"""
        print("\nCurrent key distribution:")
        for node, cache in self.cache_nodes.items():
            print(f"  {node}: {list(cache.keys())}")
        
        stats = self.hash_ring.get_stats()
        print(f"\nHash ring stats: {stats}")

# Demo
if __name__ == "__main__":
    # Create distributed cache with 3 nodes
    cache = DistributedCache(['node1', 'node2', 'node3'])
    
    # Add some data
    test_keys = ['user:1', 'user:2', 'user:3', 'user:4', 'user:5', 'user:6']
    for key in test_keys:
        cache.put(key, f"data_for_{key}")
    
    cache.print_distribution()
    
    print("\n" + "="*50)
    print("Adding node4...")
    cache.add_node('node4')
    cache.print_distribution()
    
    print("\n" + "="*50)
    print("Removing node2...")
    cache.remove_node('node2')
    cache.print_distribution()
    
    # Test retrieval
    print("\n" + "="*50)
    print("Testing retrieval:")
    for key in test_keys:
        value = cache.get(key)
        print(f"get({key}) = {value}")
```

### Benefits of Consistent Hashing

1. **Minimal Reorganization**: Only K/N keys move when adding/removing nodes (K=total keys, N=total nodes)
2. **Load Distribution**: Virtual nodes ensure even distribution
3. **Fault Tolerance**: Easy to handle node failures
4. **Scalability**: Easy to add/remove nodes

### Real-World Applications

- **Amazon DynamoDB**: Partitioning data across nodes
- **Apache Cassandra**: Data distribution in clusters
- **Memcached**: Client-side sharding
- **Load Balancers**: Consistent server selection
- **CDNs**: Content distribution across edge servers

---

## Health Checks and Monitoring

### Health Check Patterns

#### 1. Simple Health Check
```python
from flask import Flask, jsonify
import psutil
import time

app = Flask(__name__)

@app.route('/health')
def health_check():
    """Basic health check endpoint"""
    return jsonify({
        'status': 'healthy',
        'timestamp': time.time(),
        'service': 'user-service',
        'version': '1.0.0'
    }), 200

@app.route('/health/detailed')
def detailed_health_check():
    """Detailed health check with dependencies"""
    
    health_status = {
        'status': 'healthy',
        'timestamp': time.time(),
        'service': 'user-service',
        'version': '1.0.0',
        'checks': {}
    }
    
    # Database check
    try:
        # Simulate database ping
        db_start = time.time()
        # database.ping()  # Your database ping here
        db_duration = time.time() - db_start
        
        health_status['checks']['database'] = {
            'status': 'healthy',
            'response_time_ms': int(db_duration * 1000)
        }
    except Exception as e:
        health_status['checks']['database'] = {
            'status': 'unhealthy',
            'error': str(e)
        }
        health_status['status'] = 'degraded'
    
    # Redis check
    try:
        redis_start = time.time()
        # redis_client.ping()  # Your Redis ping here
        redis_duration = time.time() - redis_start
        
        health_status['checks']['redis'] = {
            'status': 'healthy',
            'response_time_ms': int(redis_duration * 1000)
        }
    except Exception as e:
        health_status['checks']['redis'] = {
            'status': 'unhealthy', 
            'error': str(e)
        }
        health_status['status'] = 'degraded'
    
    # System resources check
    cpu_percent = psutil.cpu_percent(interval=1)
    memory_percent = psutil.virtual_memory().percent
    disk_percent = psutil.disk_usage('/').percent
    
    health_status['checks']['system'] = {
        'status': 'healthy',
        'cpu_percent': cpu_percent,
        'memory_percent': memory_percent,
        'disk_percent': disk_percent
    }
    
    # Mark as unhealthy if resources are too high
    if cpu_percent > 90 or memory_percent > 90 or disk_percent > 90:
        health_status['checks']['system']['status'] = 'unhealthy'
        health_status['status'] = 'unhealthy'
    
    # Determine overall status
    if health_status['status'] != 'healthy':
        return jsonify(health_status), 503  # Service Unavailable
    else:
        return jsonify(health_status), 200

@app.route('/ready')
def readiness_check():
    """Kubernetes-style readiness check"""
    # Check if service is ready to receive traffic
    # This might include checking if initialization is complete,
    # dependencies are available, etc.
    
    ready_checks = {
        'database_initialized': True,  # Check if DB schema is ready
        'cache_warmed': True,         # Check if cache is warmed up
        'configuration_loaded': True   # Check if config is loaded
    }
    
    all_ready = all(ready_checks.values())
    
    return jsonify({
        'ready': all_ready,
        'checks': ready_checks
    }), 200 if all_ready else 503

@app.route('/live')
def liveness_check():
    """Kubernetes-style liveness check"""
    # Simple check - if this endpoint responds, the service is alive
    return jsonify({'live': True}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

#### 2. Circuit Breaker Health Check
```python
class HealthCheckCircuitBreaker:
    """Circuit breaker specifically for health checks"""
    
    def __init__(self, dependency_name, max_failures=3, timeout=30):
        self.dependency_name = dependency_name
        self.max_failures = max_failures
        self.timeout = timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.is_open = False
    
    def check_health(self, health_check_func):
        """Wrapper for health check functions"""
        
        # If circuit is open, check if we should try again
        if self.is_open:
            if (time.time() - self.last_failure_time) > self.timeout:
                self.is_open = False  # Try again
                self.failure_count = 0
            else:
                return {
                    'status': 'unhealthy',
                    'error': f'{self.dependency_name} circuit breaker is OPEN'
                }
        
        try:
            result = health_check_func()
            self.failure_count = 0  # Reset on success
            return result
            
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            
            if self.failure_count >= self.max_failures:
                self.is_open = True
            
            return {
                'status': 'unhealthy',
                'error': str(e),
                'failure_count': self.failure_count
            }

# Usage
db_health_cb = HealthCheckCircuitBreaker('database', max_failures=2, timeout=60)
redis_health_cb = HealthCheckCircuitBreaker('redis', max_failures=3, timeout=30)

def check_database():
    # Your database health check
    # database.execute("SELECT 1")
    return {'status': 'healthy', 'response_time_ms': 50}

def check_redis():
    # Your Redis health check  
    # redis_client.ping()
    return {'status': 'healthy', 'response_time_ms': 10}

@app.route('/health/protected')
def protected_health_check():
    """Health check with circuit breakers"""
    
    health_status = {
        'status': 'healthy',
        'timestamp': time.time(),
        'checks': {
            'database': db_health_cb.check_health(check_database),
            'redis': redis_health_cb.check_health(check_redis)
        }
    }
    
    # Determine overall status
    unhealthy_checks = [
        check for check in health_status['checks'].values()
        if check['status'] != 'healthy'
    ]
    
    if unhealthy_checks:
        health_status['status'] = 'degraded' if len(unhealthy_checks) < len(health_status['checks']) else 'unhealthy'
        return jsonify(health_status), 503
    
    return jsonify(health_status), 200
```

---

## Retry Strategies

### Retry Patterns

#### 1. Simple Retry
```python
import time
import random

def simple_retry(func, max_attempts=3, delay=1):
    """Simple retry with fixed delay"""
    
    for attempt in range(max_attempts):
        try:
            return func()
        except Exception as e:
            if attempt == max_attempts - 1:  # Last attempt
                raise e
            
            print(f"Attempt {attempt + 1} failed: {e}. Retrying in {delay}s...")
            time.sleep(delay)
    
# Usage
def unreliable_function():
    if random.random() < 0.7:  # 70% failure rate
        raise Exception("Random failure")
    return "Success!"

result = simple_retry(unreliable_function, max_attempts=5, delay=2)
```

#### 2. Exponential Backoff
```python
def exponential_backoff_retry(func, max_attempts=5, base_delay=1, max_delay=60, backoff_factor=2):
    """Retry with exponential backoff"""
    
    for attempt in range(max_attempts):
        try:
            return func()
        except Exception as e:
            if attempt == max_attempts - 1:  # Last attempt
                raise e
            
            # Calculate delay: base_delay * (backoff_factor ^ attempt)
            delay = min(base_delay * (backoff_factor ** attempt), max_delay)
            
            print(f"Attempt {attempt + 1} failed: {e}. Retrying in {delay}s...")
            time.sleep(delay)

# Usage
result = exponential_backoff_retry(
    unreliable_function,
    max_attempts=6,
    base_delay=1,
    max_delay=32,
    backoff_factor=2
)
# Delays: 1s, 2s, 4s, 8s, 16s, 32s
```

#### 3. Exponential Backoff with Jitter
```python
def exponential_backoff_with_jitter(func, max_attempts=5, base_delay=1, max_delay=60, backoff_factor=2):
    """Retry with exponential backoff and jitter to avoid thundering herd"""
    
    for attempt in range(max_attempts):
        try:
            return func()
        except Exception as e:
            if attempt == max_attempts - 1:
                raise e
            
            # Calculate base delay
            base_wait = min(base_delay * (backoff_factor ** attempt), max_delay)
            
            # Add jitter (random factor between 0 and 1)
            jitter = random.uniform(0, 1)
            delay = base_wait * jitter
            
            print(f"Attempt {attempt + 1} failed: {e}. Retrying in {delay:.2f}s...")
            time.sleep(delay)

# More advanced jitter strategies
def full_jitter(base_delay, attempt, backoff_factor=2, max_delay=60):
    """Full jitter: delay = random(0, exponential_backoff)"""
    max_wait = min(base_delay * (backoff_factor ** attempt), max_delay)
    return random.uniform(0, max_wait)

def equal_jitter(base_delay, attempt, backoff_factor=2, max_delay=60):
    """Equal jitter: delay = base/2 + random(0, base/2)"""
    max_wait = min(base_delay * (backoff_factor ** attempt), max_delay)
    return max_wait / 2 + random.uniform(0, max_wait / 2)

def decorrelated_jitter(base_delay, prev_delay, max_delay=60):
    """Decorrelated jitter: delay = random(base, prev_delay * 3)"""
    return min(random.uniform(base_delay, prev_delay * 3), max_delay)
```

#### 4. Smart Retry with Circuit Breaker
```python
class RetryWithCircuitBreaker:
    def __init__(self, max_attempts=3, base_delay=1, circuit_breaker=None):
        self.max_attempts = max_attempts
        self.base_delay = base_delay
        self.circuit_breaker = circuit_breaker
    
    def retry(self, func, *args, **kwargs):
        """Retry with circuit breaker integration"""
        
        for attempt in range(self.max_attempts):
            try:
                if self.circuit_breaker:
                    return self.circuit_breaker.call(func, *args, **kwargs)
                else:
                    return func(*args, **kwargs)
                    
            except CircuitBreakerError:
                # Circuit breaker is open, don't retry
                raise
                
            except Exception as e:
                if attempt == self.max_attempts - 1:
                    raise e
                
                # Exponential backoff with jitter
                delay = full_jitter(self.base_delay, attempt)
                time.sleep(delay)

# Usage with specific retry conditions
def should_retry_exception(exception):
    """Determine if an exception should trigger a retry"""
    
    # Don't retry on authentication errors
    if isinstance(exception, AuthenticationError):
        return False
    
    # Don't retry on validation errors
    if isinstance(exception, ValidationError):
        return False
    
    # Retry on network/timeout errors
    if isinstance(exception, (ConnectionError, TimeoutError)):
        return True
    
    # Retry on 5xx HTTP errors but not 4xx
    if hasattr(exception, 'status_code'):
        return 500 <= exception.status_code < 600
    
    return True  # Default: retry unknown exceptions

def smart_retry(func, max_attempts=3, should_retry=should_retry_exception):
    """Retry only on specific exceptions"""
    
    for attempt in range(max_attempts):
        try:
            return func()
        except Exception as e:
            if attempt == max_attempts - 1 or not should_retry(e):
                raise e
            
            delay = full_jitter(1, attempt)
            time.sleep(delay)
```

---

## Rate Limiting

### Rate Limiting Algorithms

#### 1. Token Bucket
```python
import time
import threading

class TokenBucket:
    """Token bucket rate limiter"""
    
    def __init__(self, capacity, refill_rate):
        """
        Args:
            capacity: Maximum number of tokens the bucket can hold
            refill_rate: Number of tokens added per second
        """
        self.capacity = capacity
        self.tokens = capacity
        self.refill_rate = refill_rate
        self.last_refill = time.time()
        self._lock = threading.Lock()
    
    def consume(self, tokens=1):
        """Try to consume tokens from the bucket"""
        
        with self._lock:
            now = time.time()
            
            # Add tokens based on time elapsed
            tokens_to_add = (now - self.last_refill) * self.refill_rate
            self.tokens = min(self.capacity, self.tokens + tokens_to_add)
            self.last_refill = now
            
            if self.tokens >= tokens:
                self.tokens -= tokens
                return True
            else:
                return False
    
    def get_tokens(self):
        """Get current number of tokens (for monitoring)"""
        with self._lock:
            now = time.time()
            tokens_to_add = (now - self.last_refill) * self.refill_rate
            return min(self.capacity, self.tokens + tokens_to_add)

# Usage example - API rate limiting
class APIRateLimiter:
    def __init__(self):
        # Different limits for different users/endpoints
        self.user_buckets = {}
        self.endpoint_buckets = {}
        
        # Global rate limit
        self.global_bucket = TokenBucket(capacity=1000, refill_rate=100)  # 100 req/sec
    
    def is_allowed(self, user_id, endpoint, tokens_needed=1):
        """Check if request is allowed"""
        
        # Check global limit first
        if not self.global_bucket.consume(tokens_needed):
            return False, "Global rate limit exceeded"
        
        # Check user-specific limit
        if user_id not in self.user_buckets:
            self.user_buckets[user_id] = TokenBucket(capacity=100, refill_rate=10)  # 10 req/sec per user
        
        if not self.user_buckets[user_id].consume(tokens_needed):
            return False, f"User {user_id} rate limit exceeded"
        
        # Check endpoint-specific limit
        if endpoint not in self.endpoint_buckets:
            # Different limits for different endpoints
            if endpoint.startswith('/api/upload'):
                bucket = TokenBucket(capacity=10, refill_rate=1)  # 1 upload/sec
            else:
                bucket = TokenBucket(capacity=50, refill_rate=5)   # 5 req/sec for other endpoints
            
            self.endpoint_buckets[endpoint] = bucket
        
        if not self.endpoint_buckets[endpoint].consume(tokens_needed):
            return False, f"Endpoint {endpoint} rate limit exceeded"
        
        return True, "Request allowed"

# Flask integration
from flask import Flask, request, jsonify

app = Flask(__name__)
rate_limiter = APIRateLimiter()

def rate_limit_middleware():
    """Middleware to check rate limits"""
    
    user_id = request.headers.get('X-User-ID', 'anonymous')
    endpoint = request.path
    
    allowed, message = rate_limiter.is_allowed(user_id, endpoint)
    
    if not allowed:
        return jsonify({'error': message}), 429  # Too Many Requests
    
    return None  # Continue processing

@app.before_request
def before_request():
    result = rate_limit_middleware()
    if result:
        return result
```

#### 2. Sliding Window Log
```python
from collections import defaultdict, deque
import time

class SlidingWindowLog:
    """Sliding window log rate limiter"""
    
    def __init__(self, max_requests, window_size_seconds):
        self.max_requests = max_requests
        self.window_size = window_size_seconds
        self.request_logs = defaultdict(deque)  # user_id -> deque of timestamps
        self._lock = threading.Lock()
    
    def is_allowed(self, user_id):
        """Check if request is allowed for user"""
        
        with self._lock:
            now = time.time()
            user_log = self.request_logs[user_id]
            
            # Remove old requests outside the window
            while user_log and user_log[0] < (now - self.window_size):
                user_log.popleft()
            
            # Check if under limit
            if len(user_log) < self.max_requests:
                user_log.append(now)
                return True, len(user_log)
            else:
                return False, len(user_log)

# Usage
rate_limiter = SlidingWindowLog(max_requests=100, window_size_seconds=60)  # 100 req/minute

def check_rate_limit(user_id):
    allowed, current_count = rate_limiter.is_allowed(user_id)
    if not allowed:
        return jsonify({
            'error': 'Rate limit exceeded',
            'current_requests': current_count,
            'max_requests': rate_limiter.max_requests,
            'window_seconds': rate_limiter.window_size
        }), 429
    
    return None
```

#### 3. Fixed Window Counter
```python
import time
import threading
from collections import defaultdict

class FixedWindowCounter:
    """Fixed window counter rate limiter"""
    
    def __init__(self, max_requests, window_size_seconds):
        self.max_requests = max_requests
        self.window_size = window_size_seconds
        self.counters = defaultdict(lambda: {'count': 0, 'window_start': 0})
        self._lock = threading.Lock()
    
    def is_allowed(self, user_id):
        """Check if request is allowed"""
        
        with self._lock:
            now = time.time()
            user_data = self.counters[user_id]
            
            # Calculate current window
            current_window = int(now // self.window_size)
            
            # Reset counter if we're in a new window
            if current_window > user_data['window_start']:
                user_data['count'] = 0
                user_data['window_start'] = current_window
            
            # Check limit
            if user_data['count'] < self.max_requests:
                user_data['count'] += 1
                return True, user_data['count']
            else:
                return False, user_data['count']

# Distributed rate limiting with Redis
import redis

class DistributedRateLimiter:
    """Redis-based distributed rate limiter"""
    
    def __init__(self, redis_client, max_requests, window_size_seconds):
        self.redis = redis_client
        self.max_requests = max_requests
        self.window_size = window_size_seconds
    
    def is_allowed(self, user_id):
        """Distributed rate limiting using Redis"""
        
        now = int(time.time())
        window = now // self.window_size
        key = f"rate_limit:{user_id}:{window}"
        
        pipe = self.redis.pipeline()
        pipe.incr(key)
        pipe.expire(key, self.window_size)
        results = pipe.execute()
        
        current_count = results[0]
        
        return current_count <= self.max_requests, current_count

# Redis Lua script for atomic rate limiting
RATE_LIMIT_SCRIPT = """
local key = KEYS[1]
local max_requests = tonumber(ARGV[1])
local window_size = tonumber(ARGV[2])

local current = redis.call('GET', key)
if current == false then
    current = 0
else
    current = tonumber(current)
end

if current < max_requests then
    local new_count = redis.call('INCR', key)
    if new_count == 1 then
        redis.call('EXPIRE', key, window_size)
    end
    return {1, new_count}  -- allowed, current count
else
    return {0, current}   -- not allowed, current count
end
"""

class AtomicRedisRateLimiter:
    def __init__(self, redis_client, max_requests, window_size_seconds):
        self.redis = redis_client
        self.max_requests = max_requests
        self.window_size = window_size_seconds
        self.script = self.redis.register_script(RATE_LIMIT_SCRIPT)
    
    def is_allowed(self, user_id):
        now = int(time.time())
        window = now // self.window_size
        key = f"rate_limit:{user_id}:{window}"
        
        result = self.script(keys=[key], args=[self.max_requests, self.window_size])
        allowed = bool(result[0])
        current_count = result[1]
        
        return allowed, current_count
```

---

## Interview Questions

### Basic Level

**Q1: What is the purpose of a load balancer and what are the main types?**

**Answer:**
Load balancers distribute incoming requests across multiple servers to:
- Prevent server overload
- Improve system availability
- Increase overall throughput
- Enable horizontal scaling

**Main Types:**
- **Layer 4 (Transport)**: Routes based on IP and port, faster but less intelligent
- **Layer 7 (Application)**: Routes based on content (URLs, headers), slower but more flexible

**Q2: Explain the circuit breaker pattern and when you would use it.**

**Answer:**
Circuit breaker prevents cascading failures by monitoring service calls and "opening" when failures exceed a threshold.

**States:**
- **Closed**: Normal operation, requests pass through
- **Open**: Failures exceeded threshold, requests fail fast
- **Half-Open**: Testing if service has recovered

**Use when:**
- Calling external services
- Service failures could cascade
- Want to fail fast instead of waiting for timeouts
- Need to give failing services time to recover

**Q3: What is consistent hashing and why is it useful?**

**Answer:**
Consistent hashing distributes data across nodes using a hash ring, where both keys and nodes are hashed onto the same ring.

**Benefits:**
- **Minimal reorganization**: Only K/N keys move when nodes change
- **Load distribution**: Virtual nodes ensure even distribution  
- **Scalability**: Easy to add/remove nodes
- **Fault tolerance**: Handles node failures gracefully

**Used in:** Distributed caches, databases (Cassandra), CDNs, load balancers

### Intermediate Level

**Q4: Design a rate limiting system for an API that needs to support different limits for different user tiers.**

**Answer Framework:**
1. **Algorithm Choice**: Token bucket for smooth rate limiting
2. **Architecture**:
   ```
   Request → Rate Limiter → API Service
             ↓
           Redis (distributed state)
   ```

3. **Implementation**:
   ```python
   # Different buckets per user tier
   limits = {
       'free': {'capacity': 100, 'refill_rate': 1},      # 100 req/hour
       'premium': {'capacity': 1000, 'refill_rate': 10}, # 1000 req/hour  
       'enterprise': {'capacity': 10000, 'refill_rate': 100}
   }
   ```

4. **Key Design Decisions**:
   - Redis for distributed state
   - Lua scripts for atomic operations
   - Multiple rate limit dimensions (per user, per endpoint, global)
   - Graceful degradation if rate limiter fails

**Q5: How would you implement health checks for a microservices system?**

**Answer Framework:**
1. **Health Check Types**:
   - **Liveness**: Is the service alive?
   - **Readiness**: Can the service handle traffic?
   - **Startup**: Is the service fully initialized?

2. **Implementation Levels**:
   ```python
   /health        # Simple ping
   /health/deep   # Check dependencies (DB, cache, etc.)
   /health/deps   # Just dependency status
   ```

3. **Circuit Breakers for Dependencies**:
   - Prevent health check failures from cascading
   - Mark dependencies as unhealthy if circuit is open

4. **Monitoring Integration**:
   - Service discovery registration/deregistration
   - Load balancer configuration
   - Alerting on health check failures

### Advanced Level

**Q6: Design a global load balancing system for a CDN with geographic distribution.**

**Answer Framework:**
1. **Multi-Level Load Balancing**:
   ```
   DNS-based Geographic Routing
           ↓
   Regional Load Balancers  
           ↓
   Local Server Selection
   ```

2. **Geographic Routing**:
   - GeoDNS for coarse-grained routing
   - Anycast IP for automatic geographic routing
   - Health-aware DNS responses

3. **Load Balancing Algorithms**:
   - **Global**: Geographic proximity + server health
   - **Regional**: Consistent hashing for cache locality
   - **Local**: Least connections with health checks

4. **Failover Strategy**:
   ```python
   def select_server(user_location):
       # Try primary region
       servers = get_regional_servers(user_location)
       if healthy_servers_available(servers):
           return consistent_hash_select(servers)
       
       # Failover to adjacent regions
       for fallback_region in get_fallback_regions(user_location):
           servers = get_regional_servers(fallback_region)
           if healthy_servers_available(servers):
               return consistent_hash_select(servers)
       
       # Last resort: any healthy server globally
       return get_any_healthy_server()
   ```

**Q7: How would you design a resilient system that can handle partial failures gracefully?**

**Answer Framework:**
1. **Failure Isolation**:
   - Circuit breakers between services
   - Bulkhead pattern for resource isolation
   - Timeout and retry policies

2. **Graceful Degradation**:
   ```python
   def get_user_recommendations(user_id):
       try:
           # Try ML recommendation service
           return ml_service.get_recommendations(user_id)
       except ServiceError:
           try:
               # Fallback to simple collaborative filtering
               return simple_recommendations(user_id)
           except:
               # Last resort: popular items
               return get_popular_items()
   ```

3. **Data Redundancy**:
   - Multi-region data replication
   - Consistent hashing for automatic failover
   - Read replicas for load distribution

4. **Monitoring and Alerting**:
   - Health checks at multiple levels
   - Circuit breaker state monitoring
   - SLA/SLO tracking and alerting

5. **Testing Resilience**:
   - Chaos engineering (Chaos Monkey)
   - Failure injection testing
   - Load testing with simulated failures

---

<function_calls>
<invoke name="mark_todo_as_done">
<parameter name="todo_ids">["46ca0ce2-80d7-4cd6-9eb5-729157146984", "4b50e40a-c203-46df-8fa8-d2de39f41227"]