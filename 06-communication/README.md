# Communication Protocols 📡

Understanding different ways systems and services communicate with each other in distributed architectures.

## Table of Contents

- [REST APIs](#rest-apis)
- [GraphQL](#graphql)
- [gRPC](#grpc)
- [WebSockets](#websockets)
- [Server-Sent Events (SSE)](#server-sent-events-sse)
- [Long Polling](#long-polling)
- [Message Queues vs Direct Communication](#message-queues-vs-direct-communication)
- [Protocol Selection Guide](#protocol-selection-guide)
- [Interview Questions](#interview-questions)

---

## REST APIs

### RESTful Design Principles

REST (Representational State Transfer) is an architectural style for distributed systems that uses HTTP methods and standard conventions.

#### Core Principles

1. **Stateless**: Each request contains all information needed to understand it
2. **Resource-Based**: Everything is a resource identified by URLs
3. **HTTP Methods**: Use standard HTTP verbs (GET, POST, PUT, DELETE)
4. **Representation**: Resources can have multiple representations (JSON, XML)

```python
# RESTful API example
from flask import Flask, request, jsonify

app = Flask(__name__)

# Resource: Users
@app.route('/api/v1/users', methods=['GET'])
def get_users():
    """Get all users - Collection endpoint"""
    page = request.args.get('page', 1, type=int)
    limit = request.args.get('limit', 10, type=int)
    
    users = User.query.paginate(page=page, per_page=limit)
    
    return jsonify({
        'users': [user.to_dict() for user in users.items],
        'pagination': {
            'page': page,
            'pages': users.pages,
            'total': users.total
        }
    })

@app.route('/api/v1/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    """Get specific user - Resource endpoint"""
    user = User.query.get_or_404(user_id)
    return jsonify(user.to_dict())

@app.route('/api/v1/users', methods=['POST'])
def create_user():
    """Create new user"""
    data = request.get_json()
    
    # Validation
    if not data or 'email' not in data:
        return jsonify({'error': 'Email is required'}), 400
    
    user = User(email=data['email'], name=data.get('name'))
    db.session.add(user)
    db.session.commit()
    
    return jsonify(user.to_dict()), 201

@app.route('/api/v1/users/<int:user_id>', methods=['PUT'])
def update_user(user_id):
    """Update user (full update)"""
    user = User.query.get_or_404(user_id)
    data = request.get_json()
    
    user.email = data.get('email', user.email)
    user.name = data.get('name', user.name)
    
    db.session.commit()
    return jsonify(user.to_dict())

@app.route('/api/v1/users/<int:user_id>', methods=['PATCH'])
def partial_update_user(user_id):
    """Partial update user"""
    user = User.query.get_or_404(user_id)
    data = request.get_json()
    
    for field in ['email', 'name']:
        if field in data:
            setattr(user, field, data[field])
    
    db.session.commit()
    return jsonify(user.to_dict())

@app.route('/api/v1/users/<int:user_id>', methods=['DELETE'])
def delete_user(user_id):
    """Delete user"""
    user = User.query.get_or_404(user_id)
    db.session.delete(user)
    db.session.commit()
    
    return '', 204
```

### REST URL Design Patterns

#### Resource Hierarchies
```python
# Good RESTful URL patterns
/api/v1/users                    # GET: List users, POST: Create user
/api/v1/users/{id}               # GET: Get user, PUT/PATCH: Update, DELETE: Delete
/api/v1/users/{id}/orders        # GET: User's orders, POST: Create order for user
/api/v1/users/{id}/orders/{order_id}  # Specific order for user

# Nested resources
/api/v1/companies/{id}/departments           # Company's departments
/api/v1/companies/{id}/departments/{dept_id}/employees  # Employees in department

# Query parameters for filtering, sorting, pagination
/api/v1/products?category=electronics&sort=price&page=2&limit=20
```

### HTTP Status Codes

```python
class APIResponse:
    """Standard HTTP status codes for REST APIs"""
    
    # Success responses
    SUCCESS = 200          # Generic success
    CREATED = 201          # Resource created
    ACCEPTED = 202         # Request accepted, processing async
    NO_CONTENT = 204       # Success but no content to return
    
    # Redirection
    NOT_MODIFIED = 304     # Resource hasn't changed (caching)
    
    # Client errors
    BAD_REQUEST = 400      # Invalid request format
    UNAUTHORIZED = 401     # Authentication required
    FORBIDDEN = 403        # User doesn't have permission
    NOT_FOUND = 404        # Resource doesn't exist
    METHOD_NOT_ALLOWED = 405  # HTTP method not supported
    CONFLICT = 409         # Resource conflict (duplicate email)
    UNPROCESSABLE_ENTITY = 422  # Validation failed
    RATE_LIMITED = 429     # Too many requests
    
    # Server errors
    INTERNAL_ERROR = 500   # Generic server error
    BAD_GATEWAY = 502      # Upstream service error
    SERVICE_UNAVAILABLE = 503  # Service temporarily down
    GATEWAY_TIMEOUT = 504  # Upstream service timeout

# Usage example
@app.route('/api/v1/users', methods=['POST'])
def create_user():
    try:
        data = request.get_json()
        
        if not data:
            return jsonify({'error': 'Request body required'}), APIResponse.BAD_REQUEST
        
        if User.query.filter_by(email=data['email']).first():
            return jsonify({'error': 'Email already exists'}), APIResponse.CONFLICT
        
        user = User(**data)
        db.session.add(user)
        db.session.commit()
        
        return jsonify(user.to_dict()), APIResponse.CREATED
        
    except ValidationError as e:
        return jsonify({'error': str(e)}), APIResponse.UNPROCESSABLE_ENTITY
    except Exception as e:
        logger.error(f"Error creating user: {e}")
        return jsonify({'error': 'Internal server error'}), APIResponse.INTERNAL_ERROR
```

### REST API Best Practices

#### 1. Versioning
```python
# URL versioning (most common)
/api/v1/users
/api/v2/users

# Header versioning
# Accept: application/vnd.myapi.v1+json

# Query parameter versioning
/api/users?version=1

# Implementation
@app.route('/api/v1/users')
def get_users_v1():
    # Version 1 implementation
    return jsonify([user.to_dict_v1() for user in users])

@app.route('/api/v2/users')
def get_users_v2():
    # Version 2 with additional fields
    return jsonify([user.to_dict_v2() for user in users])
```

#### 2. Error Handling
```python
class APIError:
    def __init__(self, message, code=None, details=None):
        self.message = message
        self.code = code
        self.details = details or {}
    
    def to_dict(self):
        return {
            'error': {
                'message': self.message,
                'code': self.code,
                'details': self.details,
                'timestamp': datetime.utcnow().isoformat()
            }
        }

# Consistent error responses
{
    "error": {
        "message": "Validation failed",
        "code": "VALIDATION_ERROR",
        "details": {
            "email": ["Email is required"],
            "password": ["Password must be at least 8 characters"]
        },
        "timestamp": "2024-01-01T12:00:00Z"
    }
}
```

#### 3. Pagination
```python
class PaginatedResponse:
    def __init__(self, items, page, per_page, total):
        self.items = items
        self.page = page
        self.per_page = per_page
        self.total = total
    
    def to_dict(self):
        return {
            'data': [item.to_dict() for item in self.items],
            'pagination': {
                'page': self.page,
                'per_page': self.per_page,
                'total': self.total,
                'pages': (self.total + self.per_page - 1) // self.per_page,
                'has_next': self.page * self.per_page < self.total,
                'has_prev': self.page > 1
            }
        }

# Cursor-based pagination for large datasets
@app.route('/api/v1/posts')
def get_posts():
    cursor = request.args.get('cursor')  # timestamp or ID
    limit = request.args.get('limit', 20, type=int)
    
    if cursor:
        posts = Post.query.filter(Post.created_at < cursor).order_by(Post.created_at.desc()).limit(limit)
    else:
        posts = Post.query.order_by(Post.created_at.desc()).limit(limit)
    
    next_cursor = posts[-1].created_at.isoformat() if posts else None
    
    return jsonify({
        'data': [post.to_dict() for post in posts],
        'next_cursor': next_cursor
    })
```

### REST Limitations

1. **Over-fetching**: Getting more data than needed
2. **Under-fetching**: Multiple requests needed for related data
3. **Rigid Structure**: Fixed response format
4. **No Real-time**: Request-response only

```python
# Over-fetching example
# Client only needs user name and email
# But gets all user data including address, preferences, etc.
user_data = requests.get('/api/v1/users/123').json()
name = user_data['name']  # Only using name field

# Under-fetching example  
# Need user info + their recent orders
user = requests.get('/api/v1/users/123').json()          # Request 1
orders = requests.get('/api/v1/users/123/orders').json() # Request 2
```

---

## GraphQL

### What is GraphQL?

GraphQL is a query language and runtime for APIs that allows clients to request exactly the data they need.

```graphql
# GraphQL Schema Definition
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
  createdAt: DateTime!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  comments: [Comment!]!
  createdAt: DateTime!
}

type Comment {
  id: ID!
  content: String!
  author: User!
  post: Post!
  createdAt: DateTime!
}

type Query {
  user(id: ID!): User
  users(limit: Int, offset: Int): [User!]!
  post(id: ID!): Post
  posts(authorId: ID, limit: Int): [Post!]!
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
  createPost(input: CreatePostInput!): Post!
}

input CreateUserInput {
  name: String!
  email: String!
}

input UpdateUserInput {
  name: String
  email: String
}

input CreatePostInput {
  title: String!
  content: String!
  authorId: ID!
}
```

### GraphQL Queries

```graphql
# Basic query - get specific fields only
query GetUser {
  user(id: "123") {
    id
    name
    email
  }
}

# Nested query - get user with their posts
query GetUserWithPosts {
  user(id: "123") {
    id
    name
    email
    posts {
      id
      title
      createdAt
    }
  }
}

# Multiple queries in one request
query GetUserAndPosts {
  user(id: "123") {
    id
    name
    email
  }
  
  posts(authorId: "123", limit: 5) {
    id
    title
    content
    comments {
      id
      content
      author {
        name
      }
    }
  }
}

# Query with variables
query GetUserPosts($userId: ID!, $postLimit: Int!) {
  user(id: $userId) {
    name
    posts(limit: $postLimit) {
      title
      createdAt
    }
  }
}

# Variables:
{
  "userId": "123",
  "postLimit": 10
}
```

### GraphQL Implementation

```python
import graphene
from graphene import ObjectType, String, Field, List, ID, Int, Mutation
from graphene_sqlalchemy import SQLAlchemyObjectType

# Convert SQLAlchemy models to GraphQL types
class UserType(SQLAlchemyObjectType):
    class Meta:
        model = User

class PostType(SQLAlchemyObjectType):
    class Meta:
        model = Post

class CommentType(SQLAlchemyObjectType):
    class Meta:
        model = Comment

# Query resolvers
class Query(ObjectType):
    user = Field(UserType, id=ID(required=True))
    users = List(UserType, limit=Int(), offset=Int())
    post = Field(PostType, id=ID(required=True))
    posts = List(PostType, author_id=ID(), limit=Int())
    
    def resolve_user(self, info, id):
        return User.query.get(id)
    
    def resolve_users(self, info, limit=10, offset=0):
        return User.query.offset(offset).limit(limit).all()
    
    def resolve_post(self, info, id):
        return Post.query.get(id)
    
    def resolve_posts(self, info, author_id=None, limit=10):
        query = Post.query
        if author_id:
            query = query.filter_by(author_id=author_id)
        return query.limit(limit).all()

# Mutations
class CreateUser(Mutation):
    class Arguments:
        name = String(required=True)
        email = String(required=True)
    
    user = Field(UserType)
    
    def mutate(self, info, name, email):
        user = User(name=name, email=email)
        db.session.add(user)
        db.session.commit()
        return CreateUser(user=user)

class UpdateUser(Mutation):
    class Arguments:
        id = ID(required=True)
        name = String()
        email = String()
    
    user = Field(UserType)
    
    def mutate(self, info, id, name=None, email=None):
        user = User.query.get(id)
        if not user:
            raise Exception('User not found')
        
        if name:
            user.name = name
        if email:
            user.email = email
        
        db.session.commit()
        return UpdateUser(user=user)

class Mutation(ObjectType):
    create_user = CreateUser.Field()
    update_user = UpdateUser.Field()

# Create schema
schema = graphene.Schema(query=Query, mutation=Mutation)

# Flask integration
from flask_graphql import GraphQLView

app.add_url_rule(
    '/graphql',
    view_func=GraphQLView.as_view('graphql', schema=schema, graphiql=True)
)
```

### Advanced GraphQL Features

#### 1. DataLoader (N+1 Problem Solution)
```python
from promise import Promise
from promise.dataloader import DataLoader

class UserLoader(DataLoader):
    def batch_load_fn(self, user_ids):
        # Load multiple users in one DB query instead of N queries
        users = User.query.filter(User.id.in_(user_ids)).all()
        user_map = {user.id: user for user in users}
        
        # Return users in same order as requested IDs
        return Promise.resolve([user_map.get(user_id) for user_id in user_ids])

class PostLoader(DataLoader):
    def batch_load_fn(self, user_ids):
        # Load posts for multiple users efficiently
        posts = Post.query.filter(Post.author_id.in_(user_ids)).all()
        
        # Group posts by author_id
        posts_by_user = {}
        for post in posts:
            if post.author_id not in posts_by_user:
                posts_by_user[post.author_id] = []
            posts_by_user[post.author_id].append(post)
        
        return Promise.resolve([posts_by_user.get(user_id, []) for user_id in user_ids])

# Usage in resolver
class UserType(SQLAlchemyObjectType):
    posts = List(PostType)
    
    def resolve_posts(self, info):
        # Use DataLoader instead of direct query
        return info.context['post_loader'].load(self.id)

# Context setup
def get_context():
    return {
        'user_loader': UserLoader(),
        'post_loader': PostLoader()
    }
```

#### 2. Subscriptions (Real-time)
```python
import graphene
from graphene import ObjectType, String, Field, Subscription

class PostSubscription(Subscription):
    post_added = Field(PostType, author_id=ID())
    
    def subscribe_post_added(self, info, author_id=None):
        # Return async generator for real-time updates
        return self.subscribe_to_post_stream(author_id)
    
    async def subscribe_to_post_stream(self, author_id):
        # Implementation with Redis pub/sub or WebSocket
        channel = f"posts:{author_id}" if author_id else "posts:all"
        
        async for message in redis_pubsub.listen(channel):
            post_data = json.loads(message)
            yield Post.query.get(post_data['id'])

class Subscription(ObjectType):
    post_added = PostSubscription.Field()

# WebSocket implementation
schema = graphene.Schema(query=Query, mutation=Mutation, subscription=Subscription)
```

### GraphQL Benefits and Limitations

#### Benefits
1. **Precise Data Fetching**: Get exactly what you need
2. **Single Request**: Fetch related data in one query
3. **Strong Type System**: Schema-first development
4. **Introspection**: Self-documenting APIs
5. **Real-time**: Subscriptions support

#### Limitations
1. **Learning Curve**: More complex than REST
2. **Caching**: HTTP caching doesn't work well
3. **File Uploads**: Not natively supported
4. **Query Complexity**: Can create expensive queries
5. **Over-complexity**: Overkill for simple APIs

---

## gRPC

### What is gRPC?

gRPC is a high-performance, open-source RPC framework that uses HTTP/2 and Protocol Buffers for communication.

```protobuf
// user_service.proto
syntax = "proto3";

package user;

// Service definition
service UserService {
  rpc GetUser(GetUserRequest) returns (UserResponse);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  rpc CreateUser(CreateUserRequest) returns (UserResponse);
  rpc UpdateUser(UpdateUserRequest) returns (UserResponse);
  rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);
  
  // Streaming RPC
  rpc StreamUsers(StreamUsersRequest) returns (stream UserResponse);
  rpc BulkCreateUsers(stream CreateUserRequest) returns (BulkCreateUsersResponse);
}

// Message definitions
message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
  int64 created_at = 4;
  repeated string tags = 5;
  Address address = 6;
}

message Address {
  string street = 1;
  string city = 2;
  string country = 3;
  string postal_code = 4;
}

message GetUserRequest {
  int64 id = 1;
}

message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;
  string filter = 3;
}

message ListUsersResponse {
  repeated User users = 1;
  string next_page_token = 2;
  int32 total_count = 3;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  Address address = 3;
  repeated string tags = 4;
}

message UpdateUserRequest {
  int64 id = 1;
  string name = 2;
  string email = 3;
  Address address = 4;
  repeated string tags = 5;
}

message UserResponse {
  User user = 1;
  bool success = 2;
  string error_message = 3;
}

message DeleteUserRequest {
  int64 id = 1;
}

message DeleteUserResponse {
  bool success = 1;
  string error_message = 2;
}

message StreamUsersRequest {
  string filter = 1;
}

message BulkCreateUsersResponse {
  int32 created_count = 1;
  repeated string errors = 2;
}
```

### gRPC Server Implementation

```python
import grpc
from concurrent import futures
import user_service_pb2
import user_service_pb2_grpc

class UserServicer(user_service_pb2_grpc.UserServiceServicer):
    def GetUser(self, request, context):
        try:
            user = User.query.get(request.id)
            if not user:
                context.set_code(grpc.StatusCode.NOT_FOUND)
                context.set_details('User not found')
                return user_service_pb2.UserResponse()
            
            return user_service_pb2.UserResponse(
                user=self._user_to_proto(user),
                success=True
            )
        except Exception as e:
            context.set_code(grpc.StatusCode.INTERNAL)
            context.set_details(str(e))
            return user_service_pb2.UserResponse(success=False, error_message=str(e))
    
    def ListUsers(self, request, context):
        try:
            page_size = request.page_size or 10
            offset = 0
            
            if request.page_token:
                offset = int(request.page_token)
            
            query = User.query
            if request.filter:
                query = query.filter(User.name.contains(request.filter))
            
            users = query.offset(offset).limit(page_size + 1).all()
            has_next = len(users) > page_size
            
            if has_next:
                users = users[:-1]  # Remove extra user
                next_token = str(offset + page_size)
            else:
                next_token = ""
            
            return user_service_pb2.ListUsersResponse(
                users=[self._user_to_proto(user) for user in users],
                next_page_token=next_token,
                total_count=User.query.count()
            )
        except Exception as e:
            context.set_code(grpc.StatusCode.INTERNAL)
            context.set_details(str(e))
            return user_service_pb2.ListUsersResponse()
    
    def CreateUser(self, request, context):
        try:
            # Validation
            if not request.name or not request.email:
                context.set_code(grpc.StatusCode.INVALID_ARGUMENT)
                context.set_details('Name and email are required')
                return user_service_pb2.UserResponse()
            
            user = User(name=request.name, email=request.email)
            
            if request.address:
                user.address = Address(
                    street=request.address.street,
                    city=request.address.city,
                    country=request.address.country,
                    postal_code=request.address.postal_code
                )
            
            user.tags = list(request.tags)
            
            db.session.add(user)
            db.session.commit()
            
            return user_service_pb2.UserResponse(
                user=self._user_to_proto(user),
                success=True
            )
        except Exception as e:
            db.session.rollback()
            context.set_code(grpc.StatusCode.INTERNAL)
            context.set_details(str(e))
            return user_service_pb2.UserResponse(success=False, error_message=str(e))
    
    def StreamUsers(self, request, context):
        """Server streaming - send users one by one"""
        try:
            query = User.query
            if request.filter:
                query = query.filter(User.name.contains(request.filter))
            
            for user in query.all():
                yield user_service_pb2.UserResponse(
                    user=self._user_to_proto(user),
                    success=True
                )
        except Exception as e:
            context.set_code(grpc.StatusCode.INTERNAL)
            context.set_details(str(e))
    
    def BulkCreateUsers(self, request_iterator, context):
        """Client streaming - receive multiple users"""
        created_count = 0
        errors = []
        
        try:
            for request in request_iterator:
                try:
                    user = User(name=request.name, email=request.email)
                    db.session.add(user)
                    created_count += 1
                except Exception as e:
                    errors.append(f"Error creating user {request.name}: {str(e)}")
            
            db.session.commit()
            
            return user_service_pb2.BulkCreateUsersResponse(
                created_count=created_count,
                errors=errors
            )
        except Exception as e:
            db.session.rollback()
            context.set_code(grpc.StatusCode.INTERNAL)
            context.set_details(str(e))
            return user_service_pb2.BulkCreateUsersResponse(errors=[str(e)])
    
    def _user_to_proto(self, user):
        proto_user = user_service_pb2.User(
            id=user.id,
            name=user.name,
            email=user.email,
            created_at=int(user.created_at.timestamp()),
            tags=user.tags or []
        )
        
        if user.address:
            proto_user.address.CopyFrom(user_service_pb2.Address(
                street=user.address.street,
                city=user.address.city,
                country=user.address.country,
                postal_code=user.address.postal_code
            ))
        
        return proto_user

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    user_service_pb2_grpc.add_UserServiceServicer_to_server(UserServicer(), server)
    
    listen_addr = '[::]:50051'
    server.add_insecure_port(listen_addr)
    
    print(f"gRPC server starting on {listen_addr}")
    server.start()
    server.wait_for_termination()

if __name__ == '__main__':
    serve()
```

### gRPC Client Implementation

```python
import grpc
import user_service_pb2
import user_service_pb2_grpc

class UserClient:
    def __init__(self, server_address='localhost:50051'):
        self.channel = grpc.insecure_channel(server_address)
        self.stub = user_service_pb2_grpc.UserServiceStub(self.channel)
    
    def get_user(self, user_id):
        request = user_service_pb2.GetUserRequest(id=user_id)
        
        try:
            response = self.stub.GetUser(request, timeout=10)
            if response.success:
                return response.user
            else:
                print(f"Error: {response.error_message}")
                return None
        except grpc.RpcError as e:
            print(f"RPC failed: {e.code()} - {e.details()}")
            return None
    
    def list_users(self, page_size=10, filter_text=""):
        request = user_service_pb2.ListUsersRequest(
            page_size=page_size,
            filter=filter_text
        )
        
        try:
            response = self.stub.ListUsers(request)
            return response.users, response.next_page_token
        except grpc.RpcError as e:
            print(f"RPC failed: {e.code()} - {e.details()}")
            return [], ""
    
    def create_user(self, name, email, address=None, tags=None):
        request = user_service_pb2.CreateUserRequest(
            name=name,
            email=email,
            tags=tags or []
        )
        
        if address:
            request.address.CopyFrom(user_service_pb2.Address(**address))
        
        try:
            response = self.stub.CreateUser(request)
            if response.success:
                return response.user
            else:
                print(f"Error: {response.error_message}")
                return None
        except grpc.RpcError as e:
            print(f"RPC failed: {e.code()} - {e.details()}")
            return None
    
    def stream_users(self, filter_text=""):
        request = user_service_pb2.StreamUsersRequest(filter=filter_text)
        
        try:
            for response in self.stub.StreamUsers(request):
                if response.success:
                    yield response.user
                else:
                    print(f"Error: {response.error_message}")
        except grpc.RpcError as e:
            print(f"RPC failed: {e.code()} - {e.details()}")
    
    def bulk_create_users(self, users_data):
        def request_generator():
            for user_data in users_data:
                yield user_service_pb2.CreateUserRequest(**user_data)
        
        try:
            response = self.stub.BulkCreateUsers(request_generator())
            return response.created_count, response.errors
        except grpc.RpcError as e:
            print(f"RPC failed: {e.code()} - {e.details()}")
            return 0, [str(e)]
    
    def close(self):
        self.channel.close()

# Usage example
if __name__ == '__main__':
    client = UserClient()
    
    # Create user
    user = client.create_user(
        name="John Doe",
        email="john@example.com",
        address={
            "street": "123 Main St",
            "city": "Anytown",
            "country": "USA",
            "postal_code": "12345"
        },
        tags=["customer", "premium"]
    )
    
    if user:
        print(f"Created user: {user.name} ({user.id})")
        
        # Get user
        retrieved_user = client.get_user(user.id)
        print(f"Retrieved user: {retrieved_user.name}")
        
        # Stream users
        print("Streaming users:")
        for user in client.stream_users():
            print(f"- {user.name} ({user.email})")
    
    client.close()
```

### gRPC Benefits and Use Cases

#### Benefits
1. **High Performance**: Binary protocol, HTTP/2 multiplexing
2. **Type Safety**: Strong typing with Protocol Buffers
3. **Streaming**: Bi-directional streaming support
4. **Language Agnostic**: Code generation for many languages
5. **Built-in Features**: Authentication, load balancing, health checking

#### Use Cases
- **Microservices Communication**: High-performance service-to-service communication
- **Real-time Applications**: Streaming data, live updates
- **Mobile Applications**: Efficient binary protocol, battery-friendly
- **IoT Systems**: Compact message format, streaming support

---

## WebSockets

### What are WebSockets?

WebSockets provide full-duplex communication over a single TCP connection, enabling real-time bi-directional communication.

```python
from flask_socketio import SocketIO, emit, join_room, leave_room
from flask import Flask

app = Flask(__name__)
app.config['SECRET_KEY'] = 'secret!'
socketio = SocketIO(app, cors_allowed_origins="*")

# Connected users tracking
connected_users = {}
chat_rooms = {}

@socketio.on('connect')
def on_connect(auth):
    print(f'Client {request.sid} connected')
    
    # Authentication
    if auth and 'token' in auth:
        user = verify_token(auth['token'])
        if user:
            connected_users[request.sid] = {
                'user_id': user.id,
                'username': user.name,
                'connected_at': datetime.utcnow()
            }
            emit('connected', {'message': f'Welcome {user.name}!'})
        else:
            return False  # Reject connection
    else:
        return False

@socketio.on('disconnect')
def on_disconnect():
    print(f'Client {request.sid} disconnected')
    
    if request.sid in connected_users:
        user_info = connected_users[request.sid]
        
        # Leave all rooms
        for room_id in chat_rooms:
            if request.sid in chat_rooms[room_id]:
                chat_rooms[room_id].remove(request.sid)
                emit('user_left', {
                    'username': user_info['username'],
                    'message': f"{user_info['username']} left the room"
                }, room=room_id)
        
        del connected_users[request.sid]

@socketio.on('join_chat')
def on_join_chat(data):
    room_id = data['room_id']
    username = connected_users.get(request.sid, {}).get('username', 'Anonymous')
    
    join_room(room_id)
    
    if room_id not in chat_rooms:
        chat_rooms[room_id] = set()
    chat_rooms[room_id].add(request.sid)
    
    emit('user_joined', {
        'username': username,
        'message': f'{username} joined the room',
        'room_id': room_id
    }, room=room_id)

@socketio.on('leave_chat')
def on_leave_chat(data):
    room_id = data['room_id']
    username = connected_users.get(request.sid, {}).get('username', 'Anonymous')
    
    leave_room(room_id)
    
    if room_id in chat_rooms and request.sid in chat_rooms[room_id]:
        chat_rooms[room_id].remove(request.sid)
    
    emit('user_left', {
        'username': username,
        'message': f'{username} left the room',
        'room_id': room_id
    }, room=room_id)

@socketio.on('send_message')
def on_send_message(data):
    if request.sid not in connected_users:
        emit('error', {'message': 'Not authenticated'})
        return
    
    user_info = connected_users[request.sid]
    room_id = data.get('room_id')
    message = data.get('message', '').strip()
    
    if not message:
        emit('error', {'message': 'Message cannot be empty'})
        return
    
    # Save message to database
    chat_message = ChatMessage(
        user_id=user_info['user_id'],
        room_id=room_id,
        message=message,
        timestamp=datetime.utcnow()
    )
    db.session.add(chat_message)
    db.session.commit()
    
    # Broadcast to room
    emit('new_message', {
        'id': chat_message.id,
        'username': user_info['username'],
        'message': message,
        'timestamp': chat_message.timestamp.isoformat(),
        'room_id': room_id
    }, room=room_id)

@socketio.on('typing')
def on_typing(data):
    if request.sid not in connected_users:
        return
    
    user_info = connected_users[request.sid]
    room_id = data.get('room_id')
    is_typing = data.get('is_typing', False)
    
    emit('user_typing', {
        'username': user_info['username'],
        'is_typing': is_typing
    }, room=room_id, include_self=False)

# REST endpoint to get chat history
@app.route('/api/chat/<room_id>/messages')
def get_chat_messages(room_id):
    page = request.args.get('page', 1, type=int)
    per_page = request.args.get('per_page', 50, type=int)
    
    messages = ChatMessage.query.filter_by(room_id=room_id)\
        .order_by(ChatMessage.timestamp.desc())\
        .paginate(page=page, per_page=per_page)
    
    return jsonify({
        'messages': [{
            'id': msg.id,
            'username': msg.user.name,
            'message': msg.message,
            'timestamp': msg.timestamp.isoformat()
        } for msg in reversed(messages.items)],  # Reverse to get chronological order
        'has_next': messages.has_next,
        'has_prev': messages.has_prev,
        'page': page,
        'pages': messages.pages
    })

if __name__ == '__main__':
    socketio.run(app, debug=True, port=5000)
```

### WebSocket Client Implementation

```javascript
// JavaScript WebSocket client
class ChatClient {
    constructor(serverUrl, token) {
        this.serverUrl = serverUrl;
        this.token = token;
        this.socket = null;
        this.currentRoom = null;
        this.messageHandlers = {};
    }
    
    connect() {
        this.socket = io(this.serverUrl, {
            auth: {
                token: this.token
            }
        });
        
        this.socket.on('connect', () => {
            console.log('Connected to server');
            this.onConnect();
        });
        
        this.socket.on('disconnect', () => {
            console.log('Disconnected from server');
            this.onDisconnect();
        });
        
        this.socket.on('connected', (data) => {
            console.log('Authentication successful:', data.message);
        });
        
        this.socket.on('new_message', (data) => {
            this.handleMessage(data);
        });
        
        this.socket.on('user_joined', (data) => {
            this.handleUserJoined(data);
        });
        
        this.socket.on('user_left', (data) => {
            this.handleUserLeft(data);
        });
        
        this.socket.on('user_typing', (data) => {
            this.handleUserTyping(data);
        });
        
        this.socket.on('error', (data) => {
            console.error('Socket error:', data.message);
        });
    }
    
    joinRoom(roomId) {
        if (this.currentRoom) {
            this.leaveRoom(this.currentRoom);
        }
        
        this.currentRoom = roomId;
        this.socket.emit('join_chat', { room_id: roomId });
    }
    
    leaveRoom(roomId) {
        this.socket.emit('leave_chat', { room_id: roomId });
        if (this.currentRoom === roomId) {
            this.currentRoom = null;
        }
    }
    
    sendMessage(message) {
        if (!this.currentRoom) {
            console.error('Not in a room');
            return;
        }
        
        this.socket.emit('send_message', {
            room_id: this.currentRoom,
            message: message
        });
    }
    
    setTyping(isTyping) {
        if (!this.currentRoom) return;
        
        this.socket.emit('typing', {
            room_id: this.currentRoom,
            is_typing: isTyping
        });
    }
    
    // Event handlers
    onConnect() {
        // Override in subclass
    }
    
    onDisconnect() {
        // Override in subclass
    }
    
    handleMessage(data) {
        // Override in subclass
        console.log('New message:', data);
    }
    
    handleUserJoined(data) {
        // Override in subclass
        console.log('User joined:', data);
    }
    
    handleUserLeft(data) {
        // Override in subclass  
        console.log('User left:', data);
    }
    
    handleUserTyping(data) {
        // Override in subclass
        console.log('User typing:', data);
    }
    
    disconnect() {
        if (this.socket) {
            this.socket.disconnect();
        }
    }
}

// Usage
const chatClient = new ChatClient('http://localhost:5000', 'your-jwt-token');
chatClient.connect();
chatClient.joinRoom('general');
chatClient.sendMessage('Hello everyone!');
```

### WebSocket Use Cases

1. **Real-time Chat**: Instant messaging applications
2. **Live Updates**: Stock prices, sports scores, news feeds
3. **Collaborative Tools**: Google Docs-like real-time editing
4. **Gaming**: Real-time multiplayer games
5. **IoT Dashboards**: Live sensor data visualization
6. **Trading Platforms**: Real-time market data

---

## Server-Sent Events (SSE)

### What are Server-Sent Events?

SSE allows a server to push data to a client over a single HTTP connection. It's simpler than WebSockets but only supports server-to-client communication.

```python
from flask import Flask, Response, request
import json
import time
import threading
from queue import Queue

app = Flask(__name__)

# Store client connections
sse_clients = {}

class SSEClient:
    def __init__(self, client_id):
        self.client_id = client_id
        self.queue = Queue()
        self.active = True
    
    def put(self, data):
        if self.active:
            self.queue.put(data)
    
    def close(self):
        self.active = False

@app.route('/events')
def events():
    def event_stream():
        client_id = request.args.get('client_id', f'client_{int(time.time())}')
        client = SSEClient(client_id)
        sse_clients[client_id] = client
        
        try:
            # Send initial connection event
            yield f"data: {json.dumps({'type': 'connected', 'client_id': client_id})}\n\n"
            
            while client.active:
                try:
                    # Wait for data with timeout
                    data = client.queue.get(timeout=30)
                    if data:
                        yield f"data: {json.dumps(data)}\n\n"
                except:
                    # Send heartbeat to keep connection alive
                    yield f"data: {json.dumps({'type': 'heartbeat', 'timestamp': time.time()})}\n\n"
        
        except GeneratorExit:
            # Client disconnected
            pass
        finally:
            client.close()
            if client_id in sse_clients:
                del sse_clients[client_id]
    
    return Response(event_stream(), mimetype='text/event-stream',
                   headers={
                       'Cache-Control': 'no-cache',
                       'Connection': 'keep-alive',
                       'Access-Control-Allow-Origin': '*'
                   })

# Broadcast message to all clients
def broadcast_message(message_type, data):
    message = {
        'type': message_type,
        'data': data,
        'timestamp': time.time()
    }
    
    for client in list(sse_clients.values()):
        client.put(message)

# Send message to specific client
def send_to_client(client_id, message_type, data):
    if client_id in sse_clients:
        message = {
            'type': message_type,
            'data': data,
            'timestamp': time.time()
        }
        sse_clients[client_id].put(message)

# REST endpoints to trigger events
@app.route('/api/broadcast', methods=['POST'])
def broadcast():
    data = request.get_json()
    broadcast_message(data.get('type', 'message'), data.get('data', {}))
    return jsonify({'status': 'sent', 'clients': len(sse_clients)})

@app.route('/api/notify/<client_id>', methods=['POST'])
def notify_client(client_id):
    data = request.get_json()
    send_to_client(client_id, data.get('type', 'notification'), data.get('data', {}))
    return jsonify({'status': 'sent'})

# Example: News feed updates
class NewsFeedService:
    def __init__(self):
        self.latest_articles = []
        self.start_feed_monitor()
    
    def start_feed_monitor(self):
        def monitor():
            while True:
                # Simulate checking for new articles
                new_articles = self.check_for_new_articles()
                if new_articles:
                    for article in new_articles:
                        broadcast_message('news_article', {
                            'title': article['title'],
                            'summary': article['summary'],
                            'url': article['url'],
                            'category': article['category']
                        })
                time.sleep(30)  # Check every 30 seconds
        
        thread = threading.Thread(target=monitor, daemon=True)
        thread.start()
    
    def check_for_new_articles(self):
        # Simulate fetching new articles
        # In real implementation, this would check RSS feeds, APIs, etc.
        import random
        if random.random() < 0.3:  # 30% chance of new article
            return [{
                'title': f'Breaking News {int(time.time())}',
                'summary': 'This is a sample news article summary...',
                'url': f'/articles/{int(time.time())}',
                'category': random.choice(['tech', 'sports', 'politics'])
            }]
        return []

# Initialize news feed service
news_service = NewsFeedService()

if __name__ == '__main__':
    app.run(debug=True, threaded=True)
```

### SSE Client Implementation

```javascript
// JavaScript SSE client
class SSEClient {
    constructor(url, options = {}) {
        this.url = url;
        this.options = options;
        this.eventSource = null;
        this.reconnectAttempts = 0;
        this.maxReconnectAttempts = options.maxReconnectAttempts || 5;
        this.reconnectInterval = options.reconnectInterval || 5000;
        this.handlers = {};
    }
    
    connect() {
        const url = new URL(this.url);
        if (this.options.clientId) {
            url.searchParams.set('client_id', this.options.clientId);
        }
        
        this.eventSource = new EventSource(url.toString());
        
        this.eventSource.onopen = (event) => {
            console.log('SSE connection opened');
            this.reconnectAttempts = 0;
            this.emit('connected', event);
        };
        
        this.eventSource.onmessage = (event) => {
            try {
                const data = JSON.parse(event.data);
                this.handleMessage(data);
            } catch (error) {
                console.error('Error parsing SSE message:', error);
            }
        };
        
        this.eventSource.onerror = (event) => {
            console.error('SSE error:', event);
            this.emit('error', event);
            
            if (this.eventSource.readyState === EventSource.CLOSED) {
                this.handleReconnect();
            }
        };
    }
    
    handleMessage(data) {
        const { type, data: messageData, timestamp } = data;
        
        switch (type) {
            case 'connected':
                this.emit('connected', messageData);
                break;
            case 'heartbeat':
                // Handle heartbeat silently
                break;
            case 'news_article':
                this.emit('news_article', messageData);
                break;
            case 'notification':
                this.emit('notification', messageData);
                break;
            default:
                this.emit('message', data);
        }
    }
    
    handleReconnect() {
        if (this.reconnectAttempts < this.maxReconnectAttempts) {
            this.reconnectAttempts++;
            console.log(`Attempting to reconnect... (${this.reconnectAttempts}/${this.maxReconnectAttempts})`);
            
            setTimeout(() => {
                this.connect();
            }, this.reconnectInterval);
        } else {
            console.error('Max reconnection attempts reached');
            this.emit('max_reconnects_reached');
        }
    }
    
    on(eventType, handler) {
        if (!this.handlers[eventType]) {
            this.handlers[eventType] = [];
        }
        this.handlers[eventType].push(handler);
    }
    
    emit(eventType, data) {
        if (this.handlers[eventType]) {
            this.handlers[eventType].forEach(handler => handler(data));
        }
    }
    
    close() {
        if (this.eventSource) {
            this.eventSource.close();
        }
    }
}

// Usage example
const sseClient = new SSEClient('/events', {
    clientId: 'user-123',
    maxReconnectAttempts: 10,
    reconnectInterval: 3000
});

sseClient.on('connected', (data) => {
    console.log('Connected to SSE stream:', data);
});

sseClient.on('news_article', (article) => {
    console.log('New article:', article.title);
    displayArticle(article);
});

sseClient.on('notification', (notification) => {
    console.log('Notification:', notification);
    showNotification(notification);
});

sseClient.connect();

function displayArticle(article) {
    const articleElement = document.createElement('div');
    articleElement.innerHTML = `
        <h3>${article.title}</h3>
        <p>${article.summary}</p>
        <span class="category">${article.category}</span>
    `;
    document.getElementById('news-feed').prepend(articleElement);
}

function showNotification(notification) {
    // Show browser notification or in-app notification
    if (Notification.permission === 'granted') {
        new Notification(notification.title, {
            body: notification.message,
            icon: '/icon.png'
        });
    }
}
```

### SSE vs WebSockets Comparison

| Feature | SSE | WebSockets |
|---------|-----|------------|
| **Direction** | Server → Client only | Bi-directional |
| **Protocol** | HTTP/HTTPS | WebSocket protocol |
| **Complexity** | Simple | More complex |
| **Reconnection** | Automatic | Manual implementation |
| **Firewall/Proxy** | Better support | Can be blocked |
| **Text Only** | Yes | Binary + Text |
| **Use Cases** | News feeds, notifications | Chat, gaming, collaboration |

---

## Long Polling

### What is Long Polling?

Long polling is a technique where the client makes a request to the server, and the server holds the connection open until it has data to send back.

```python
from flask import Flask, request, jsonify
import threading
import time
from collections import defaultdict, deque

app = Flask(__name__)

# Store polling connections
polling_connections = defaultdict(list)
message_queues = defaultdict(deque)

class PollConnection:
    def __init__(self, user_id, timestamp):
        self.user_id = user_id
        self.timestamp = timestamp
        self.event = threading.Event()
        self.response_data = None
    
    def wait_for_data(self, timeout=30):
        return self.event.wait(timeout)
    
    def send_data(self, data):
        self.response_data = data
        self.event.set()

@app.route('/api/poll/<user_id>')
def long_poll(user_id):
    timeout = request.args.get('timeout', 30, type=int)
    last_message_time = request.args.get('since', 0, type=float)
    
    # Check if there are immediate messages
    messages = []
    user_queue = message_queues[user_id]
    
    while user_queue and user_queue[0]['timestamp'] <= last_message_time:
        user_queue.popleft()  # Remove old messages
    
    if user_queue:
        # Return immediate messages
        messages = list(user_queue)
        user_queue.clear()
        return jsonify({
            'messages': messages,
            'timestamp': time.time(),
            'has_more': False
        })
    
    # No immediate messages, start long polling
    connection = PollConnection(user_id, time.time())
    polling_connections[user_id].append(connection)
    
    try:
        # Wait for data or timeout
        has_data = connection.wait_for_data(timeout)
        
        if has_data and connection.response_data:
            return jsonify(connection.response_data)
        else:
            # Timeout reached
            return jsonify({
                'messages': [],
                'timestamp': time.time(),
                'has_more': False,
                'timeout': True
            })
    
    finally:
        # Clean up connection
        if connection in polling_connections[user_id]:
            polling_connections[user_id].remove(connection)

@app.route('/api/send_message', methods=['POST'])
def send_message():
    data = request.get_json()
    recipient_id = data['recipient_id']
    message = data['message']
    sender_id = data['sender_id']
    
    message_data = {
        'id': f"msg_{int(time.time() * 1000)}",
        'sender_id': sender_id,
        'message': message,
        'timestamp': time.time()
    }
    
    # Add to message queue
    message_queues[recipient_id].append(message_data)
    
    # Notify any waiting connections
    for connection in polling_connections[recipient_id][:]:  # Copy list to avoid modification during iteration
        connection.send_data({
            'messages': [message_data],
            'timestamp': time.time(),
            'has_more': len(message_queues[recipient_id]) > 1
        })
    
    # Clear notified connections
    polling_connections[recipient_id].clear()
    
    return jsonify({'status': 'sent', 'message_id': message_data['id']})

# Example: Notification system with long polling
class NotificationService:
    def __init__(self):
        self.user_notifications = defaultdict(deque)
        self.polling_clients = defaultdict(list)
    
    def send_notification(self, user_id, notification_type, data):
        notification = {
            'id': f"notif_{int(time.time() * 1000)}",
            'type': notification_type,
            'data': data,
            'timestamp': time.time(),
            'read': False
        }
        
        self.user_notifications[user_id].append(notification)
        
        # Notify polling clients
        for connection in self.polling_clients[user_id][:]:
            connection.send_data({
                'notifications': [notification],
                'timestamp': time.time()
            })
        
        self.polling_clients[user_id].clear()
    
    def poll_notifications(self, user_id, since_timestamp, timeout=30):
        # Check for immediate notifications
        notifications = []
        user_queue = self.user_notifications[user_id]
        
        for notification in user_queue:
            if notification['timestamp'] > since_timestamp:
                notifications.append(notification)
        
        if notifications:
            return {
                'notifications': notifications,
                'timestamp': time.time()
            }
        
        # Start long polling
        connection = PollConnection(user_id, time.time())
        self.polling_clients[user_id].append(connection)
        
        try:
            has_data = connection.wait_for_data(timeout)
            if has_data and connection.response_data:
                return connection.response_data
            else:
                return {
                    'notifications': [],
                    'timestamp': time.time(),
                    'timeout': True
                }
        finally:
            if connection in self.polling_clients[user_id]:
                self.polling_clients[user_id].remove(connection)

notification_service = NotificationService()

@app.route('/api/notifications/<user_id>/poll')
def poll_notifications(user_id):
    since = request.args.get('since', 0, type=float)
    timeout = request.args.get('timeout', 30, type=int)
    
    result = notification_service.poll_notifications(user_id, since, timeout)
    return jsonify(result)

@app.route('/api/notifications/send', methods=['POST'])
def send_notification():
    data = request.get_json()
    user_id = data['user_id']
    notification_type = data['type']
    notification_data = data['data']
    
    notification_service.send_notification(user_id, notification_type, notification_data)
    
    return jsonify({'status': 'sent'})

if __name__ == '__main__':
    app.run(debug=True, threaded=True)
```

### Long Polling Client Implementation

```javascript
class LongPollingClient {
    constructor(baseUrl, userId) {
        this.baseUrl = baseUrl;
        this.userId = userId;
        this.isPolling = false;
        this.lastTimestamp = 0;
        this.pollTimeout = 30000; // 30 seconds
        this.reconnectDelay = 5000; // 5 seconds
        this.handlers = {};
    }
    
    startPolling() {
        if (this.isPolling) return;
        
        this.isPolling = true;
        this.poll();
    }
    
    async poll() {
        if (!this.isPolling) return;
        
        try {
            const params = new URLSearchParams({
                since: this.lastTimestamp,
                timeout: this.pollTimeout / 1000
            });
            
            const response = await fetch(
                `${this.baseUrl}/api/notifications/${this.userId}/poll?${params}`,
                {
                    method: 'GET',
                    headers: {
                        'Content-Type': 'application/json'
                    }
                }
            );
            
            if (!response.ok) {
                throw new Error(`HTTP ${response.status}`);
            }
            
            const data = await response.json();
            
            if (data.notifications && data.notifications.length > 0) {
                // Process notifications
                data.notifications.forEach(notification => {
                    this.handleNotification(notification);
                });
            }
            
            // Update timestamp
            if (data.timestamp) {
                this.lastTimestamp = data.timestamp;
            }
            
            // Continue polling immediately if not timeout
            if (this.isPolling) {
                if (data.timeout) {
                    // Small delay for timeout responses
                    setTimeout(() => this.poll(), 1000);
                } else {
                    this.poll();
                }
            }
            
        } catch (error) {
            console.error('Polling error:', error);
            this.emit('error', error);
            
            // Reconnect after delay
            if (this.isPolling) {
                setTimeout(() => this.poll(), this.reconnectDelay);
            }
        }
    }
    
    stopPolling() {
        this.isPolling = false;
    }
    
    handleNotification(notification) {
        console.log('Received notification:', notification);
        this.emit('notification', notification);
        
        // Emit specific notification type events
        this.emit(`notification:${notification.type}`, notification);
    }
    
    on(event, handler) {
        if (!this.handlers[event]) {
            this.handlers[event] = [];
        }
        this.handlers[event].push(handler);
    }
    
    emit(event, data) {
        if (this.handlers[event]) {
            this.handlers[event].forEach(handler => handler(data));
        }
    }
    
    // Send notification to another user
    async sendNotification(recipientId, type, data) {
        try {
            const response = await fetch(`${this.baseUrl}/api/notifications/send`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({
                    user_id: recipientId,
                    type: type,
                    data: data
                })
            });
            
            return await response.json();
        } catch (error) {
            console.error('Error sending notification:', error);
            throw error;
        }
    }
}

// Usage example
const pollingClient = new LongPollingClient('http://localhost:5000', 'user-123');

pollingClient.on('notification', (notification) => {
    console.log('New notification:', notification);
    displayNotification(notification);
});

pollingClient.on('notification:message', (notification) => {
    console.log('New message notification:', notification);
    updateMessageCounter();
});

pollingClient.on('error', (error) => {
    console.error('Polling client error:', error);
});

// Start polling
pollingClient.startPolling();

function displayNotification(notification) {
    const notificationElement = document.createElement('div');
    notificationElement.className = 'notification';
    notificationElement.innerHTML = `
        <div class="notification-type">${notification.type}</div>
        <div class="notification-message">${notification.data.message || ''}</div>
        <div class="notification-time">${new Date(notification.timestamp * 1000).toLocaleTimeString()}</div>
    `;
    
    document.getElementById('notifications').appendChild(notificationElement);
    
    // Auto-remove after 5 seconds
    setTimeout(() => {
        notificationElement.remove();
    }, 5000);
}

// Stop polling when page unloads
window.addEventListener('beforeunload', () => {
    pollingClient.stopPolling();
});
```

### Long Polling Benefits and Drawbacks

#### Benefits
1. **Simple Implementation**: Uses standard HTTP
2. **Firewall Friendly**: Works through most proxies and firewalls
3. **Real-time Feel**: Near real-time updates
4. **Compatible**: Works with existing HTTP infrastructure

#### Drawbacks
1. **Resource Intensive**: Holds server connections open
2. **Scaling Issues**: Limited by server connection limits
3. **Timeout Handling**: Requires reconnection logic
4. **Not True Real-time**: Still has some latency

---

## Protocol Selection Guide

### Decision Matrix

| Use Case | REST | GraphQL | gRPC | WebSockets | SSE | Long Polling |
|----------|------|---------|------|------------|-----|--------------|
| **CRUD Operations** | ✅ Perfect | ⚡ Good | ⚡ Good | ❌ Overkill | ❌ No | ❌ No |
| **Real-time Chat** | ❌ No | ❌ No | ⚡ Good | ✅ Perfect | ❌ One-way | ⚡ Possible |
| **Live Updates** | ❌ No | 🔄 Subscriptions | 🔄 Streaming | ✅ Perfect | ✅ Perfect | ⚡ Good |
| **Mobile Apps** | ✅ Good | ⚡ Efficient | ✅ Efficient | 🔋 Battery drain | ⚡ Good | 🔋 Battery drain |
| **Microservices** | ✅ Standard | ❌ Complex | ✅ Perfect | ❌ Overkill | ❌ No | ❌ No |
| **Public APIs** | ✅ Perfect | ✅ Perfect | ❌ Complex | ❌ No | ❌ No | ❌ No |
| **File Upload** | ✅ Good | ❌ Limited | ✅ Streaming | ❌ No | ❌ No | ❌ No |
| **Gaming** | ❌ Too slow | ❌ Too slow | ✅ Good | ✅ Perfect | ❌ One-way | ❌ Too slow |

### Selection Criteria

#### Choose REST when:
- Building CRUD APIs
- Need caching (HTTP caching)
- Wide client compatibility required
- Simple request-response pattern
- Public APIs

#### Choose GraphQL when:
- Clients need flexible data fetching
- Multiple client types with different needs
- Want to minimize over-fetching
- Have complex, related data models
- Developer experience is important

#### Choose gRPC when:
- Building microservices
- Performance is critical
- Type safety is important
- Bi-directional streaming needed
- Internal service communication

#### Choose WebSockets when:
- Need bi-directional real-time communication
- Building chat applications
- Real-time collaboration tools
- Gaming applications
- Low latency is critical

#### Choose SSE when:
- Server needs to push updates to clients
- One-way real-time communication
- Simple implementation required
- News feeds, notifications
- Live dashboards

#### Choose Long Polling when:
- Need real-time feel with HTTP
- WebSockets not supported
- Simple notification system
- Existing HTTP infrastructure
- Occasional updates

---

## Interview Questions

### Basic Level

**Q1: What are the main differences between REST and GraphQL?**

**Answer:**

| Aspect | REST | GraphQL |
|--------|------|---------|
| **Data Fetching** | Fixed endpoints, may over-fetch | Request exactly what you need |
| **Endpoints** | Multiple endpoints per resource | Single endpoint |
| **Caching** | HTTP caching works well | More complex caching |
| **Learning Curve** | Simple, well-known | Steeper learning curve |
| **Use Case** | CRUD operations, public APIs | Complex data relationships, flexible clients |

**Q2: When would you use WebSockets over HTTP?**

**Answer:**
Use WebSockets when you need:
- **Bi-directional communication**: Both client and server need to send messages
- **Low latency**: Real-time gaming, trading platforms
- **Persistent connection**: Chat applications, live collaboration
- **High frequency updates**: Live sports scores, stock tickers

Use HTTP when:
- **Simple request-response**: CRUD operations
- **Caching important**: Static or semi-static content
- **Stateless**: Each request independent
- **Wide compatibility**: Public APIs, web services

**Q3: Explain the difference between Server-Sent Events and Long Polling.**

**Answer:**
- **SSE**: Uses persistent HTTP connection with `text/event-stream`. Server pushes data when available. Built-in reconnection.
- **Long Polling**: Client makes HTTP request, server holds it open until data available or timeout. Client must reconnect after each response.

SSE is better for frequent updates, Long Polling for occasional notifications.

### Intermediate Level

**Q4: Design an API for a social media platform. Which communication protocols would you use and why?**

**Answer Framework:**
1. **REST for Core CRUD Operations**:
   ```
   GET /api/v1/posts - List posts
   POST /api/v1/posts - Create post
   PUT /api/v1/posts/{id} - Update post
   DELETE /api/v1/posts/{id} - Delete post
   ```

2. **GraphQL for Mobile/Web Clients**:
   - Flexible data fetching for different screen sizes
   - Single request for complex queries (user + posts + comments)

3. **WebSockets for Real-time Features**:
   - Live chat/messaging
   - Real-time notifications
   - Live comments on posts

4. **SSE for Activity Feeds**:
   - Push new posts to user's feed
   - Notification updates

**Q5: How would you implement real-time notifications for a web application?**

**Answer Framework:**
1. **Technology Choice**:
   - **WebSockets**: For users actively using the app
   - **SSE**: For background notifications
   - **Push Notifications**: For browser/mobile notifications when app closed

2. **Architecture**:
   ```
   User Action → API → Message Queue → Notification Service → Push to Connected Clients
   ```

3. **Implementation Considerations**:
   - User presence tracking
   - Message persistence for offline users
   - Rate limiting to prevent spam
   - Notification preferences

### Advanced Level

**Q6: Design a real-time multiplayer game communication system.**

**Answer Framework:**
1. **Protocol Selection**:
   - **WebSockets** for game state updates (bi-directional, low latency)
   - **UDP with custom protocol** for extremely low latency (if browser supports)
   - **HTTP/REST** for non-real-time operations (login, stats, leaderboards)

2. **Message Types**:
   ```javascript
   {
     type: 'player_move',
     player_id: 'player123',
     position: {x: 100, y: 200},
     timestamp: 1640995200000
   }
   ```

3. **Architecture**:
   ```
   Client ↔ Game Server ↔ Game State Manager ↔ Database
                ↓
   Other Clients (broadcast updates)
   ```

4. **Optimizations**:
   - **Message Batching**: Combine multiple updates
   - **Delta Compression**: Send only changes
   - **Client-side Prediction**: Reduce perceived latency
   - **Lag Compensation**: Handle network delays

**Q7: How would you handle API versioning in a microservices architecture?**

**Answer Framework:**
1. **Versioning Strategies**:
   - **URL Versioning**: `/api/v1/users`, `/api/v2/users`
   - **Header Versioning**: `Accept: application/vnd.api.v1+json`
   - **Semantic Versioning**: Major.Minor.Patch

2. **Microservices Considerations**:
   ```
   API Gateway → Route based on version → Appropriate Service Version
   ```

3. **Backward Compatibility**:
   - Support multiple versions simultaneously
   - Gradual migration plan
   - Deprecation notices
   - Version sunset timeline

4. **Implementation**:
   - **Service Registry**: Track service versions
   - **Contract Testing**: Ensure compatibility
   - **Blue-Green Deployment**: Safe version rollouts
   - **Feature Flags**: Gradual feature rollout

---

**Next:** Continue to [Resiliency and System Essentials](../07-resiliency/README.md) to learn about building robust, fault-tolerant systems.