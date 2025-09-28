# Real-World Examples 🏢

Learn from how major tech companies solve complex system design challenges at massive scale.

## Table of Contents

- [Netflix Architecture](#netflix-architecture)
- [Uber's Real-Time Architecture](#ubers-real-time-architecture)
- [WhatsApp Messaging System](#whatsapp-messaging-system)
- [YouTube Video Delivery](#youtube-video-delivery)
- [Amazon's Microservices](#amazons-microservices)
- [Twitter's Timeline Architecture](#twitters-timeline-architecture)
- [Spotify's Music Streaming](#spotifys-music-streaming)
- [Engineering Blog Resources](#engineering-blog-resources)

---

## Netflix Architecture

### The Challenge
- 200+ million subscribers globally
- 1 billion hours watched monthly
- 15,000+ titles in multiple formats
- Global content delivery
- 99.99% availability requirement

### Key Architectural Decisions

#### Microservices Architecture
```
┌─────────────────────────────────────────────────┐
│                Netflix Platform                 │
├─────────────────────────────────────────────────┤
│ User Service │ Content Service │ Recommendation │
├─────────────────────────────────────────────────┤
│ Billing      │ Analytics       │ Encoding       │
├─────────────────────────────────────────────────┤
│ Discovery    │ Playback        │ Personalization│
└─────────────────────────────────────────────────┘
```

**Benefits:**
- Independent deployment and scaling
- Technology diversity (Java, Python, Node.js)
- Fault isolation
- Team autonomy

#### Content Delivery Network (CDN)
```
Netflix Open Connect CDN:

┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   US East   │    │   Europe    │    │  Asia Pac   │
│   Servers   │    │   Servers   │    │   Servers   │
└─────────────┘    └─────────────┘    └─────────────┘
       │                   │                   │
       └─────────┬─────────┴─────────┬─────────┘
                 │                   │
          ┌─────────────┐    ┌─────────────┐
          │Master Video │    │ Content Mgmt│
          │   Storage   │    │   Service   │
          └─────────────┘    └─────────────┘
```

**Optimizations:**
- 95% of content served from CDN edge
- Predictive caching based on viewing patterns
- Multiple encoding formats (4K, HD, SD)

#### Chaos Engineering
```python
# Netflix's Chaos Monkey concept
class ChaosMonkey:
    def __init__(self):
        self.targets = ['instances', 'availability_zones', 'regions']
    
    def create_chaos(self):
        target = random.choice(self.targets)
        if target == 'instances':
            self.terminate_random_instance()
        elif target == 'availability_zones':
            self.disable_availability_zone()
        
    def terminate_random_instance(self):
        instances = get_healthy_instances()
        victim = random.choice(instances)
        victim.terminate()
        log(f"Chaos Monkey terminated {victim.id}")
```

**Key Lessons:**
- Build resilience by intentionally causing failures
- Test system behavior under adverse conditions
- Automate failure recovery procedures

---

## Uber's Real-Time Architecture

### The Challenge
- Real-time ride matching
- Location tracking for millions of drivers/riders
- Dynamic pricing calculations
- Global expansion across 900+ cities

### Architecture Components

#### Real-Time Location Processing
```
Driver/Rider Apps
        ↓
   API Gateway
        ↓
┌─────────────────┐
│  Location API   │
└─────────────────┘
        ↓
┌─────────────────┐    ┌─────────────────┐
│  Kafka Stream   │───▶│  Geospatial     │
│  Processing     │    │  Database       │
└─────────────────┘    │  (Redis)        │
                       └─────────────────┘
```

#### Supply-Demand Matching Algorithm
```python
class RideMatchingEngine:
    def __init__(self):
        self.drivers = SpatialIndex()  # R-tree or similar
        self.riders = Queue()
    
    def match_ride(self, ride_request):
        # Find drivers within radius
        candidate_drivers = self.drivers.find_nearby(
            ride_request.pickup_location,
            radius=2000  # meters
        )
        
        # Score drivers based on multiple factors
        scored_drivers = []
        for driver in candidate_drivers:
            score = self.calculate_score(driver, ride_request)
            scored_drivers.append((driver, score))
        
        # Select best match
        best_match = max(scored_drivers, key=lambda x: x[1])
        return best_match[0]
    
    def calculate_score(self, driver, request):
        # Distance factor
        distance = calculate_distance(
            driver.location, 
            request.pickup_location
        )
        
        # Driver rating
        rating_score = driver.rating / 5.0
        
        # ETA factor
        eta_score = 1.0 / (driver.eta_to_pickup + 1)
        
        return rating_score * 0.4 + eta_score * 0.6
```

#### Surge Pricing System
```
Real-time Events:
- Ride requests
- Driver availability
- Traffic conditions
- Weather data
- Special events

        ↓
┌─────────────────┐
│ Pricing Engine  │
└─────────────────┘
        ↓
Dynamic Price = Base Price × Multiplier

Multiplier factors:
- Supply/Demand ratio
- Time of day
- Location popularity
- Historical patterns
```

---

## WhatsApp Messaging System

### The Challenge
- 2+ billion users globally
- 100+ billion messages daily
- End-to-end encryption
- Minimal server infrastructure
- 99.9% uptime

### Key Design Principles

#### Erlang/OTP Architecture
```erlang
% WhatsApp's approach to handling millions of connections
-module(whatsapp_connection).
-behaviour(gen_server).

% Each user connection is a lightweight Erlang process
start_user_session(UserId) ->
    gen_server:start_link({local, UserId}, ?MODULE, [UserId], []).

% Message routing between users
route_message(FromUser, ToUser, Message) ->
    case is_user_online(ToUser) of
        true ->
            % Direct delivery
            gen_server:cast(ToUser, {new_message, FromUser, Message});
        false ->
            % Store for offline delivery
            offline_storage:store_message(ToUser, FromUser, Message)
    end.
```

**Advantages:**
- Lightweight processes (millions per server)
- Built-in fault tolerance
- Hot code swapping
- Pattern matching for message routing

#### Message Storage Strategy
```
Message Flow:
1. Client sends message
2. Server receives and assigns ID
3. ACK sent to sender immediately
4. Message stored temporarily (24-48 hours)
5. Message delivered to recipient
6. Delivery confirmation received
7. Message deleted from server storage

Storage Architecture:
┌─────────────┐
│   Client    │
└─────┬───────┘
      │ Send Message
      ▼
┌─────────────┐    ┌─────────────┐
│   Server    │───▶│   Temp      │
│             │    │  Storage    │
└─────┬───────┘    └─────────────┘
      │ Forward
      ▼
┌─────────────┐
│ Recipient   │
└─────────────┘
```

#### Scaling Strategies
```
Server Architecture:
- 50 engineers for 900 million users
- FreeBSD + Erlang/OTP
- Custom XMPP protocol
- Minimal databases (mostly in-memory)
- Connection pooling and load balancing

Per-server capacity:
- 2-3 million concurrent connections
- 50+ servers globally
- Geographic routing
```

---

## YouTube Video Delivery

### The Challenge
- 2 billion logged-in users monthly
- 500 hours uploaded every minute
- Multiple video qualities (360p to 8K)
- Global content delivery
- Live streaming capabilities

### Video Processing Pipeline

```
Upload Flow:
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Client    │───▶│Upload Server│───▶│  Storage    │
│  (Creator)  │    │             │    │  (Blob)     │
└─────────────┘    └─────────────┘    └─────────────┘
                                            │
                                            ▼
                   ┌─────────────┐    ┌─────────────┐
                   │  Encoding   │◄───│   Queue     │
                   │  Pipeline   │    │  Service    │
                   └─────────────┘    └─────────────┘
                          │
                          ▼
                   ┌─────────────┐    ┌─────────────┐
                   │     CDN     │───▶│  Metadata   │
                   │ Distribution│    │  Database   │
                   └─────────────┘    └─────────────┘
```

#### Adaptive Bitrate Streaming
```python
class VideoQualitySelector:
    def __init__(self):
        self.quality_levels = [
            {'resolution': '144p', 'bitrate': 80},
            {'resolution': '240p', 'bitrate': 150},
            {'resolution': '360p', 'bitrate': 300},
            {'resolution': '480p', 'bitrate': 500},
            {'resolution': '720p', 'bitrate': 1000},
            {'resolution': '1080p', 'bitrate': 2000},
            {'resolution': '1440p', 'bitrate': 4000},
            {'resolution': '2160p', 'bitrate': 8000}
        ]
    
    def select_quality(self, bandwidth, device_capability):
        # Start with highest quality device can handle
        max_quality = self.get_max_device_quality(device_capability)
        
        # Adjust based on available bandwidth
        suitable_qualities = [
            q for q in self.quality_levels 
            if q['bitrate'] <= bandwidth * 0.8  # 80% of bandwidth
            and q['resolution'] <= max_quality
        ]
        
        return max(suitable_qualities, key=lambda x: x['bitrate'])
```

#### Content Delivery Optimization
```
Global CDN Strategy:
- 1000+ points of presence
- Edge caching based on popularity
- Regional content optimization
- Predictive pre-caching

Caching Hierarchy:
┌─────────────┐
│ Edge Cache  │ ← Popular content (1-2 days)
├─────────────┤
│Regional Hub │ ← Trending content (1 week)
├─────────────┤
│Origin Server│ ← All content (permanent)
└─────────────┘
```

---

## Amazon's Microservices

### The Challenge
- Millions of products
- Global marketplace
- High availability requirements
- Rapid feature development
- Multiple business units

### Service-Oriented Architecture Evolution

#### Two-Pizza Team Rule
```
Traditional Monolith:
┌─────────────────────────────────┐
│         Amazon.com              │
│                                │
│ - Product catalog               │
│ - User management               │
│ - Order processing              │
│ - Payment processing            │
│ - Inventory management          │
│ - Shipping                      │
│ - Reviews                       │
└─────────────────────────────────┘

Microservices Architecture:
┌─────────┐ ┌─────────┐ ┌─────────┐
│Product  │ │User Mgmt│ │Orders   │
│Catalog  │ │Service  │ │Service  │
└─────────┘ └─────────┘ └─────────┘
┌─────────┐ ┌─────────┐ ┌─────────┐
│Payment  │ │Inventory│ │Shipping │
│Service  │ │Service  │ │Service  │
└─────────┘ └─────────┘ └─────────┘
```

#### Amazon's API-First Approach
```python
# All internal communication through APIs
class AmazonService:
    def __init__(self, service_name):
        self.service_name = service_name
        self.api_client = AmazonAPIClient()
    
    def call_service(self, target_service, endpoint, data):
        # All service calls go through well-defined APIs
        response = self.api_client.post(
            url=f"https://internal.{target_service}.amazon.com/{endpoint}",
            headers={'Authorization': self.get_service_token()},
            json=data
        )
        return response
    
    def get_product_details(self, product_id):
        # Even internal calls use the same APIs as external ones
        return self.call_service(
            'product-catalog', 
            f'products/{product_id}', 
            {}
        )
```

#### Database Per Service Pattern
```
Service Isolation:
┌─────────────┐    ┌─────────────┐
│   Product   │    │    User     │
│   Service   │    │   Service   │
│      │      │    │      │      │
│      ▼      │    │      ▼      │
│  Product DB │    │   User DB   │
└─────────────┘    └─────────────┘

Benefits:
- Independent scaling
- Technology choice per service
- Fault isolation
- Team ownership
```

---

## Twitter's Timeline Architecture

### The Challenge
- 330+ million monthly active users
- 500 million tweets per day
- Real-time timeline generation
- Celebrity users with millions of followers
- Global distribution

### Timeline Generation Strategies

#### Fan-Out Approaches

**Fan-Out on Write (Push Model):**
```python
def post_tweet(user_id, tweet):
    # Store the tweet
    tweet_id = store_tweet(user_id, tweet)
    
    # Get all followers
    followers = get_followers(user_id)
    
    # Push to all follower timelines
    for follower_id in followers:
        add_to_timeline(follower_id, tweet_id)
    
    # Trigger real-time notifications
    notify_followers(followers, tweet_id)
```

**Fan-Out on Read (Pull Model):**
```python
def get_timeline(user_id):
    # Get users this person follows
    following = get_following(user_id)
    
    # Merge recent tweets from all followed users
    timeline_tweets = []
    for followed_user in following:
        recent_tweets = get_recent_tweets(followed_user, limit=20)
        timeline_tweets.extend(recent_tweets)
    
    # Sort by timestamp and return top N
    return sorted(timeline_tweets, key=lambda x: x.timestamp)[-50:]
```

#### Hybrid Approach for Scale
```python
class TwitterTimelineService:
    def __init__(self):
        self.celebrity_threshold = 1000000  # 1M followers
    
    def post_tweet(self, user_id, tweet):
        tweet_id = self.store_tweet(user_id, tweet)
        followers = self.get_followers(user_id)
        
        if len(followers) > self.celebrity_threshold:
            # Celebrity: don't fan-out, use pull model
            self.mark_as_celebrity_tweet(tweet_id)
        else:
            # Regular user: fan-out to followers
            self.fan_out_to_followers(tweet_id, followers)
    
    def get_timeline(self, user_id):
        # Get pre-computed timeline (from fan-out)
        timeline = self.get_precomputed_timeline(user_id)
        
        # Merge with celebrity tweets (pull)
        celebrity_tweets = self.get_celebrity_tweets(user_id)
        
        return self.merge_timelines(timeline, celebrity_tweets)
```

#### Real-Time Updates
```
WebSocket Architecture:
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Client    │◄──▶│   Gateway   │◄──▶│ Timeline    │
│  (Mobile/   │    │   Server    │    │  Service    │
│   Web)      │    │             │    │             │
└─────────────┘    └─────────────┘    └─────────────┘
                           │                  │
                           ▼                  ▼
                   ┌─────────────┐    ┌─────────────┐
                   │Connection   │    │   Tweet     │
                   │   Pool      │    │ Processing  │
                   └─────────────┘    └─────────────┘
```

---

## Spotify's Music Streaming

### The Challenge
- 400+ million users
- 70+ million songs
- Real-time streaming
- Personalized recommendations
- Offline capabilities

### Microservices Architecture

#### Domain-Driven Design
```
Spotify Services:
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│   User      │ │  Playlist   │ │   Search    │
│  Service    │ │  Service    │ │  Service    │
└─────────────┘ └─────────────┘ └─────────────┘
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│Playback     │ │Recommendation│ │   Social    │
│Service      │ │  Service     │ │  Service    │
└─────────────┘ └─────────────┘ └─────────────┘
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│   Audio     │ │   Metadata  │ │   Content   │
│ Delivery    │ │   Service   │ │   Service   │
└─────────────┘ └─────────────┘ └─────────────┘
```

#### Music Recommendation Engine
```python
class SpotifyRecommendationEngine:
    def __init__(self):
        self.collaborative_filtering = CollaborativeFiltering()
        self.content_based = ContentBasedFiltering()
        self.neural_network = DeepLearningModel()
    
    def get_recommendations(self, user_id, context):
        # Collaborative filtering (users with similar taste)
        cf_recommendations = self.collaborative_filtering.recommend(user_id)
        
        # Content-based (similar songs/artists)
        cb_recommendations = self.content_based.recommend(
            user_id, 
            context['recent_songs']
        )
        
        # Deep learning (complex patterns)
        dl_recommendations = self.neural_network.predict(
            user_features=self.get_user_features(user_id),
            context_features=self.extract_context_features(context)
        )
        
        # Ensemble method to combine approaches
        return self.ensemble_ranker([
            cf_recommendations,
            cb_recommendations, 
            dl_recommendations
        ])
```

#### Audio Streaming Architecture
```
Streaming Flow:
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Client    │───▶│    CDN      │───▶│   Origin    │
│             │    │   (Edge)    │    │   Storage   │
└─────────────┘    └─────────────┘    └─────────────┘

Quality Adaptation:
- Multiple bitrates (96kbps, 160kbps, 320kbps)
- Adaptive streaming based on bandwidth
- Offline caching for Premium users
- Progressive download for immediate playback
```

---

## Engineering Blog Resources

### Must-Read Engineering Blogs

#### High Scalability Blogs
- **Netflix Tech Blog**: https://netflixtechblog.com/
- **Uber Engineering**: https://eng.uber.com/
- **Airbnb Engineering**: https://medium.com/airbnb-engineering
- **LinkedIn Engineering**: https://engineering.linkedin.com/
- **Pinterest Engineering**: https://medium.com/pinterest-engineering

#### Database and Storage
- **Amazon AWS Architecture**: https://aws.amazon.com/architecture/
- **Google Cloud Architecture**: https://cloud.google.com/architecture
- **MongoDB Engineering**: https://engineering.mongodb.com/
- **Cassandra at Scale**: DataStax Blog

#### Messaging and Real-Time Systems
- **Kafka at LinkedIn**: Confluent Blog
- **WhatsApp Engineering**: Facebook Engineering Blog
- **Discord Engineering**: https://blog.discord.com/
- **Slack Engineering**: https://slack.engineering/

### Key Papers and Resources

#### Foundational Papers
1. **Google MapReduce** (2004)
2. **Amazon Dynamo** (2007)  
3. **Google Bigtable** (2006)
4. **Facebook TAO** (2013)
5. **Netflix Chaos Engineering** (2016)

#### System Design Resources
- **System Design Primer**: GitHub repository
- **Designing Data-Intensive Applications**: Book by Martin Kleppmann
- **High Scalability**: Website with case studies
- **AWS Well-Architected Framework**: Amazon's best practices
- **Google SRE Books**: Site Reliability Engineering practices

### Learning from Failures

#### Notable Outages and Lessons
1. **AWS S3 Outage 2017**: Importance of region isolation
2. **Facebook Outage 2021**: DNS and BGP dependencies
3. **GitHub Outage 2018**: Database failover complexities
4. **CloudFlare Outage 2019**: Regular expression catastrophe

**Key Takeaways:**
- Build redundancy at every level
- Test failure scenarios regularly
- Monitor dependencies carefully
- Have rollback procedures ready
- Post-mortems are learning opportunities

---

**Conclusion**: These real-world examples demonstrate that successful system design is about making informed trade-offs, understanding your specific constraints, and learning from others' experiences. Each company's architecture reflects their unique challenges, scale, and business requirements.