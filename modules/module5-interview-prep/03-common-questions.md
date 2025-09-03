# Common System Design Interview Questions

## Question Categories

### Beginner Level Questions (Good for Practice)
1. **Design a URL Shortener (like bit.ly)**
2. **Design a Chat System**
3. **Design a Parking Lot System** 
4. **Design a Library Management System**
5. **Design a Rate Limiter**

### Intermediate Level Questions (Most Common)
1. **Design Twitter**
2. **Design Instagram** 
3. **Design Uber**
4. **Design WhatsApp**
5. **Design a Notification System**
6. **Design a Web Crawler**
7. **Design a File Storage System**
8. **Design a Video Streaming Service**

### Advanced Level Questions (Senior Roles)
1. **Design Google Search**
2. **Design Netflix**
3. **Design Amazon**
4. **Design a Global Payment System**
5. **Design a Real-time Analytics Platform**
6. **Design a Distributed Cache**
7. **Design a Database System**

## Detailed Question Breakdowns

### 1. Design Twitter (Most Popular Question)

#### Requirements Clarification
```python
class TwitterDesignQuestion:
    def __init__(self):
        self.clarification_questions = [
            "What are the core features? (Tweet, follow, timeline, search?)",
            "How many users? (100M DAU assumed)",
            "What's the read/write ratio? (300:1 assumed)",
            "Do we need real-time features? (Yes, for timeline updates)",
            "Global deployment needed? (Yes)",
            "Any character limits? (280 characters)",
            "Media support needed? (Yes, images and videos)"
        ]
        
        self.requirements = {
            'functional': [
                'Post tweets (280 characters + media)',
                'Follow/unfollow users',
                'View home timeline',
                'View user profile and tweets',
                'Search tweets and users',
                'Like and retweet posts'
            ],
            'non_functional': [
                '100M daily active users',
                '300:1 read to write ratio',
                'Timeline load < 200ms',
                '99.9% availability',
                'Global deployment'
            ]
        }
    
    def capacity_estimation(self):
        """Back-of-envelope calculations for Twitter"""
        
        # Traffic estimation
        dau = 100_000_000
        tweets_per_user_per_day = 2
        timeline_requests_per_user_per_day = 50
        
        # Write traffic
        tweets_per_day = dau * tweets_per_user_per_day
        tweets_per_second = tweets_per_day / (24 * 3600)  # ~2.3K TPS
        peak_tweets_per_second = tweets_per_second * 3    # ~7K TPS
        
        # Read traffic  
        timeline_requests_per_day = dau * timeline_requests_per_user_per_day
        timeline_rps = timeline_requests_per_day / (24 * 3600)  # ~58K RPS
        peak_timeline_rps = timeline_rps * 3                     # ~174K RPS
        
        # Storage estimation
        tweet_size = 200  # bytes
        media_size = 1024 * 1024  # 1MB average
        tweets_with_media_ratio = 0.1  # 10% tweets have media
        
        daily_storage = (
            tweets_per_day * tweet_size +
            tweets_per_day * tweets_with_media_ratio * media_size
        )
        
        return {
            'traffic': {
                'peak_write_tps': peak_tweets_per_second,
                'peak_read_rps': peak_timeline_rps
            },
            'storage': {
                'daily_storage_gb': daily_storage / (1024**3),
                'yearly_storage_tb': daily_storage * 365 / (1024**4)
            }
        }
    
    def system_design(self):
        """High-level system design"""
        
        return {
            'architecture': {
                'client_tier': 'Mobile apps, web clients',
                'load_balancer': 'Distribute traffic across regions',
                'api_gateway': 'Authentication, rate limiting, routing',
                'application_tier': [
                    'Tweet Service (create, read tweets)',
                    'Timeline Service (fan-out, timeline generation)', 
                    'User Service (profiles, relationships)',
                    'Search Service (tweet and user search)',
                    'Media Service (image/video processing)'
                ],
                'data_tier': [
                    'MySQL (user data, relationships)',
                    'Cassandra (tweets, timelines)', 
                    'Redis (caching)',
                    'Elasticsearch (search)',
                    'S3 (media storage)'
                ]
            },
            
            'key_algorithms': {
                'timeline_generation': 'Hybrid push/pull fan-out',
                'tweet_id_generation': 'Snowflake algorithm',
                'search_ranking': 'TF-IDF with recency boost'
            }
        }
```

### 2. Design Instagram

#### Approach and Key Considerations
```python
class InstagramDesignQuestion:
    def __init__(self):
        self.key_differences_from_twitter = [
            'Photo/video focused (larger media files)',
            'Higher storage requirements',
            'Image processing pipeline needed',
            'Different engagement patterns (likes, comments on photos)'
        ]
    
    def requirements_analysis(self):
        """Instagram-specific requirements"""
        
        return {
            'functional_requirements': [
                'Upload photos and videos',
                'Follow users and view their content',
                'News feed of photos/videos',
                'Like and comment on posts',
                'Search users and hashtags',
                'Stories feature (24-hour expiry)',
                'Direct messaging'
            ],
            
            'non_functional_requirements': [
                '500M daily active users',
                '95M photos uploaded daily',
                'Read-heavy workload (100:1 ratio)',
                'Global CDN for media delivery',
                'Image processing pipeline',
                'Mobile-first design'
            ],
            
            'unique_challenges': [
                'Large media file storage and delivery',
                'Image processing and multiple resolutions',
                'High bandwidth requirements',
                'Mobile data usage optimization'
            ]
        }
    
    def media_processing_pipeline(self):
        """Design media processing pipeline"""
        
        return {
            'upload_flow': [
                '1. Client uploads original image/video',
                '2. Store original in S3',
                '3. Queue processing job',
                '4. Generate multiple resolutions',
                '5. Apply filters and optimizations',
                '6. Store processed versions',
                '7. Update CDN cache',
                '8. Notify user of completion'
            ],
            
            'processing_requirements': [
                'Multiple image sizes (thumbnail, medium, full)',
                'Different formats (WebP, JPEG, PNG)',
                'Video transcoding (multiple bitrates)',
                'Metadata extraction (EXIF data)',
                'Content moderation (AI-based)'
            ],
            
            'optimization_strategies': [
                'Async processing to avoid blocking upload',
                'Progressive image loading',
                'CDN for global delivery',
                'Lazy loading for feeds',
                'Image compression optimization'
            ]
        }
```

### 3. Design Uber

#### Real-time Location System Focus
```python
class UberDesignQuestion:
    def __init__(self):
        self.unique_challenges = [
            'Real-time location tracking',
            'Efficient driver-rider matching',
            'Dynamic pricing (surge)',
            'Route optimization',
            'Payment processing',
            'Two-sided marketplace'
        ]
    
    def location_system_design(self):
        """Focus on location tracking system"""
        
        return {
            'location_updates': {
                'frequency': 'Every 4-8 seconds while active',
                'storage': 'Redis for real-time, Cassandra for history',
                'indexing': 'Geohash for spatial queries',
                'optimization': 'Batch updates to reduce server load'
            },
            
            'matching_algorithm': {
                'proximity_search': 'Geohash-based driver discovery',
                'optimization_factors': [
                    'Distance to pickup',
                    'Driver rating',
                    'Estimated arrival time',
                    'Driver utilization'
                ],
                'global_optimization': 'Assignment problem solver'
            },
            
            'real_time_tracking': {
                'technology': 'WebSockets for live updates',
                'fallback': 'HTTP polling if WebSocket fails',
                'optimization': 'Location compression for bandwidth'
            }
        }
    
    def surge_pricing_system(self):
        """Design dynamic pricing system"""
        
        return {
            'supply_demand_analysis': {
                'supply_tracking': 'Count available drivers in area',
                'demand_tracking': 'Count ride requests in area',
                'ratio_calculation': 'Supply/demand ratio determines surge'
            },
            
            'pricing_algorithm': {
                'base_calculation': 'Distance × time × base rate',
                'surge_multiplier': 'Based on supply/demand ratio',
                'external_factors': 'Weather, events, time of day',
                'smoothing': 'Gradual price changes to avoid shock'
            },
            
            'implementation': {
                'real_time_updates': 'Update prices every 1-2 minutes',
                'geographic_granularity': 'City block level pricing',
                'transparency': 'Show surge multiplier to users'
            }
        }
```

### 4. Design a Chat System

#### Real-time Messaging Focus
```python
class ChatSystemDesignQuestion:
    def __init__(self):
        self.chat_types = {
            'one_on_one': 'Direct messaging between two users',
            'group_chat': 'Multiple users in same conversation',
            'broadcast': 'One-to-many messaging (channels)'
        }
    
    def message_delivery_system(self):
        """Design reliable message delivery"""
        
        return {
            'delivery_guarantees': {
                'at_least_once': 'Messages not lost but may duplicate',
                'exactly_once': 'Complex but prevents duplicates',
                'best_effort': 'Simple but may lose messages'
            },
            
            'online_delivery': {
                'technology': 'WebSockets for real-time delivery',
                'fallback': 'HTTP long polling',
                'acknowledgments': 'Delivery and read receipts'
            },
            
            'offline_delivery': {
                'storage': 'Queue messages for offline users',
                'push_notifications': 'Alert users of new messages',
                'sync_on_reconnect': 'Deliver queued messages when user comes online'
            },
            
            'group_message_optimization': {
                'fan_out': 'Send to all group members',
                'optimization': 'Batch delivery for large groups',
                'ordering': 'Maintain message order per group'
            }
        }
    
    def presence_system(self):
        """Design user presence and status system"""
        
        return {
            'presence_states': ['online', 'away', 'busy', 'offline'],
            'heartbeat_mechanism': 'Periodic ping to maintain presence',
            'presence_distribution': 'Notify contacts of status changes',
            'last_seen_tracking': 'Track when user was last active'
        }
```

## Question-Specific Strategies

### 1. Search System Questions
```python
class SearchSystemStrategy:
    """Strategy for search-related questions (Google Search, Elasticsearch, etc.)"""
    
    def __init__(self):
        self.key_components = [
            'Web crawler',
            'Indexing system', 
            'Query processing',
            'Ranking algorithm',
            'Result serving'
        ]
    
    def design_approach(self):
        """Systematic approach for search systems"""
        
        return {
            'crawling_and_indexing': {
                'crawling_strategy': [
                    'Breadth-first vs depth-first crawling',
                    'Politeness policies (robots.txt)',
                    'Duplicate content detection',
                    'Freshness vs coverage trade-off'
                ],
                'indexing_pipeline': [
                    'Content extraction and parsing',
                    'Inverted index construction',
                    'Distributed indexing across shards',
                    'Index optimization and compression'
                ]
            },
            
            'query_processing': {
                'query_parsing': 'Tokenization, stemming, spell correction',
                'query_optimization': 'Query rewriting and expansion',
                'intent_detection': 'Understanding user search intent'
            },
            
            'ranking_and_serving': {
                'ranking_signals': 'Relevance, authority, freshness, personalization',
                'machine_learning': 'Learning-to-rank algorithms',
                'result_serving': 'Fast retrieval from distributed index'
            }
        }
```

### 2. Streaming System Questions
```python
class StreamingSystemStrategy:
    """Strategy for streaming-related questions (Netflix, YouTube, etc.)"""
    
    def __init__(self):
        self.key_challenges = [
            'Global content delivery',
            'Adaptive bitrate streaming',
            'Content encoding pipeline',
            'Recommendation system',
            'Analytics and monitoring'
        ]
    
    def design_approach(self):
        """Systematic approach for streaming systems"""
        
        return {
            'content_delivery': {
                'cdn_strategy': [
                    'Global CDN deployment',
                    'Edge server placement',
                    'Content pre-positioning',
                    'Bandwidth optimization'
                ],
                'adaptive_streaming': [
                    'Multiple bitrate encoding',
                    'HLS/DASH protocol support',
                    'Client-side adaptation logic',
                    'Quality measurement and adjustment'
                ]
            },
            
            'content_processing': {
                'encoding_pipeline': [
                    'Video transcoding to multiple formats',
                    'Thumbnail generation',
                    'Metadata extraction',
                    'Quality control validation'
                ],
                'storage_strategy': [
                    'Hot/warm/cold storage tiers',
                    'Geographic distribution',
                    'Backup and disaster recovery'
                ]
            },
            
            'personalization': {
                'recommendation_engine': [
                    'Collaborative filtering',
                    'Content-based filtering',
                    'Deep learning models',
                    'Real-time personalization'
                ],
                'a_b_testing': [
                    'Recommendation algorithm testing',
                    'UI/UX optimization',
                    'Performance impact measurement'
                ]
            }
        }
```

### 3. E-commerce System Questions
```python
class EcommerceSystemStrategy:
    """Strategy for e-commerce questions (Amazon, eBay, etc.)"""
    
    def __init__(self):
        self.key_components = [
            'Product catalog',
            'Inventory management',
            'Order processing',
            'Payment system',
            'Recommendation engine',
            'Search and discovery'
        ]
    
    def design_approach(self):
        """Systematic approach for e-commerce systems"""
        
        return {
            'catalog_and_search': {
                'product_catalog': [
                    'Product information management',
                    'Category hierarchy',
                    'Inventory tracking',
                    'Pricing management'
                ],
                'search_functionality': [
                    'Full-text search with filters',
                    'Faceted search navigation',
                    'Auto-complete and suggestions',
                    'Search result ranking'
                ]
            },
            
            'order_processing': {
                'order_workflow': [
                    'Cart management',
                    'Checkout process',
                    'Payment processing',
                    'Order fulfillment',
                    'Shipping and tracking'
                ],
                'inventory_management': [
                    'Real-time inventory updates',
                    'Reservation system',
                    'Distributed inventory across warehouses',
                    'Backorder handling'
                ]
            },
            
            'personalization': {
                'recommendation_types': [
                    'Product recommendations',
                    'Cross-selling and up-selling',
                    'Personalized search results',
                    'Dynamic pricing'
                ],
                'data_sources': [
                    'Purchase history',
                    'Browsing behavior',
                    'User preferences',
                    'Similar user patterns'
                ]
            }
        }
```

## Sample Interview Responses

### 1. URL Shortener - Complete Response
```python
class URLShortenerCompleteResponse:
    """Complete response for URL shortener question"""
    
    def requirements_phase(self):
        """Requirements clarification (5 minutes)"""
        
        return {
            'clarifying_questions': [
                "How many URLs do we expect to shorten per month?",
                "What's the read/write ratio for URL access?",
                "How long should URLs remain valid?",
                "Do we need custom aliases?",
                "Do we need analytics/click tracking?",
                "What's the expected scale?"
            ],
            
            'final_requirements': {
                'functional': [
                    'Shorten long URLs to short URLs',
                    'Redirect short URLs to original URLs',
                    'Custom aliases (optional)',
                    'Basic analytics (click count)'
                ],
                'non_functional': [
                    '100M URLs shortened per month',
                    '100:1 read to write ratio',
                    'URL redirection < 100ms',
                    '99.9% availability',
                    'URLs valid for 10 years'
                ]
            }
        }
    
    def capacity_estimation_phase(self):
        """Capacity estimation (5 minutes)"""
        
        return {
            'traffic_calculation': {
                'write_traffic': {
                    'urls_per_month': 100_000_000,
                    'urls_per_second': 100_000_000 / (30 * 24 * 3600),  # ~38 URLs/sec
                    'peak_write_rps': 38 * 3  # ~114 URLs/sec
                },
                'read_traffic': {
                    'redirects_per_month': 100_000_000 * 100,  # 100:1 ratio
                    'redirects_per_second': 10_000_000_000 / (30 * 24 * 3600),  # ~3.8K RPS
                    'peak_read_rps': 3800 * 3  # ~11.4K RPS
                }
            },
            
            'storage_calculation': {
                'url_record_size': 500,  # bytes (original URL + short URL + metadata)
                'total_urls_5_years': 100_000_000 * 12 * 5,  # 6B URLs
                'storage_requirement': 6_000_000_000 * 500 / (1024**3),  # ~2.8 TB
                'with_replication': 2.8 * 3  # ~8.4 TB with 3x replication
            }
        }
    
    def architecture_design_phase(self):
        """High-level architecture (10 minutes)"""
        
        return {
            'components': [
                'Load Balancer (distribute traffic)',
                'Application Servers (URL shortening logic)',
                'Database (URL mappings)',
                'Cache (hot URLs)',
                'Analytics Service (click tracking)'
            ],
            
            'data_flow': {
                'url_shortening': 'Client → LB → App Server → Database → Response',
                'url_redirection': 'Client → LB → App Server → Cache/Database → Redirect'
            },
            
            'technology_choices': {
                'database': 'NoSQL (DynamoDB/Cassandra) for simple key-value operations',
                'cache': 'Redis for hot URL mappings',
                'load_balancer': 'Application load balancer with health checks'
            }
        }
    
    def detailed_design_phase(self):
        """Detailed design (20 minutes)"""
        
        return {
            'url_encoding_algorithm': {
                'approach': 'Base62 encoding of auto-incrementing ID',
                'implementation': '''
                def encode_url_id(url_id):
                    chars = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
                    base = len(chars)
                    encoded = ""
                    
                    while url_id > 0:
                        encoded = chars[url_id % base] + encoded
                        url_id = url_id // base
                    
                    return encoded
                ''',
                'alternative': 'Hash-based approach with collision handling'
            },
            
            'database_schema': {
                'url_mappings': {
                    'short_url': 'string (PRIMARY KEY)',
                    'original_url': 'string',
                    'created_at': 'timestamp',
                    'expires_at': 'timestamp',
                    'click_count': 'integer'
                }
            },
            
            'caching_strategy': {
                'cache_hot_urls': '20% of URLs generate 80% of traffic',
                'cache_size': 'Store top 1M URLs in cache',
                'ttl': '1 hour for cached URLs',
                'invalidation': 'Remove expired URLs'
            }
        }
    
    def optimization_phase(self):
        """Optimization and scaling (5 minutes)"""
        
        return {
            'performance_optimizations': [
                'Cache popular URLs in Redis',
                'Use CDN for global distribution', 
                'Database read replicas for read scaling',
                'Connection pooling for database connections'
            ],
            
            'scaling_strategies': [
                'Horizontal scaling of app servers',
                'Database sharding by URL hash',
                'Auto-scaling based on traffic patterns',
                'Regional deployments for global users'
            ],
            
            'monitoring_and_alerting': [
                'Monitor URL creation rate',
                'Track redirection latency',
                'Alert on high error rates',
                'Monitor cache hit ratios'
            ]
        }
```

### 2. Design a Notification System - Complete Response
```python
class NotificationSystemCompleteResponse:
    """Complete response for notification system question"""
    
    def requirements_analysis(self):
        """Notification system requirements"""
        
        return {
            'functional_requirements': [
                'Send notifications via multiple channels (push, email, SMS)',
                'Support different notification types (promotional, transactional, alerts)',
                'User preferences for notification channels',
                'Template management for notifications',
                'Scheduling and delayed notifications',
                'Notification history and tracking'
            ],
            
            'non_functional_requirements': [
                '10M users, 1B notifications per day',
                'Delivery latency < 1 second for critical notifications',
                '99.9% delivery success rate',
                'Support for global users across timezones',
                'Rate limiting to prevent spam'
            ],
            
            'notification_channels': {
                'push_notifications': 'Mobile app notifications',
                'email': 'SMTP-based email delivery',
                'sms': 'SMS via third-party providers',
                'in_app': 'Notifications within application',
                'webhooks': 'API callbacks to external systems'
            }
        }
    
    def architecture_design(self):
        """Notification system architecture"""
        
        return {
            'core_services': [
                'Notification API Service',
                'Template Service',
                'User Preference Service',
                'Delivery Service',
                'Analytics Service'
            ],
            
            'message_flow': {
                'notification_creation': 'Service → Notification API → Validation → Queue',
                'delivery_processing': 'Queue → Delivery Service → Channel Providers → User',
                'status_tracking': 'Delivery Status → Analytics Service → Reporting'
            },
            
            'queue_architecture': {
                'high_priority_queue': 'Critical notifications (security alerts)',
                'normal_priority_queue': 'Regular notifications (social updates)',
                'bulk_queue': 'Marketing notifications',
                'retry_queue': 'Failed notifications for retry'
            }
        }
    
    def delivery_optimization(self):
        """Optimize notification delivery"""
        
        return {
            'channel_optimization': {
                'push_notifications': [
                    'Batch delivery to reduce API calls',
                    'Device token management and cleanup',
                    'Fallback to other channels if push fails'
                ],
                'email_delivery': [
                    'SMTP connection pooling',
                    'Email template caching',
                    'Bounce and complaint handling'
                ],
                'sms_delivery': [
                    'Multiple SMS provider integration',
                    'Cost optimization by provider selection',
                    'International SMS routing'
                ]
            },
            
            'user_experience_optimization': [
                'Respect user notification preferences',
                'Intelligent frequency capping',
                'Time zone aware delivery',
                'A/B testing for notification content'
            ],
            
            'reliability_patterns': [
                'Circuit breaker for external providers',
                'Retry with exponential backoff',
                'Dead letter queue for failed notifications',
                'Multi-provider redundancy'
            ]
        }
```

## Question Practice Framework

### 1. Structured Practice Approach
```python
class QuestionPracticeFramework:
    def __init__(self):
        self.practice_methodology = {
            'preparation_phase': [
                'Read question carefully',
                'Identify question category (social, streaming, e-commerce, etc.)',
                'Recall similar systems you know',
                'Set timer for realistic practice'
            ],
            
            'execution_phase': [
                'Follow RADIO framework strictly',
                'Think out loud throughout',
                'Draw diagrams as you explain',
                'Check time every 10 minutes'
            ],
            
            'review_phase': [
                'Compare with sample solutions',
                'Identify areas for improvement',
                'Note new concepts learned',
                'Practice weak areas again'
            ]
        }
    
    def self_evaluation_checklist(self):
        """Checklist for evaluating your practice sessions"""
        
        return {
            'requirements_gathering': [
                '✓ Asked relevant clarifying questions',
                '✓ Defined clear functional requirements',
                '✓ Identified non-functional requirements',
                '✓ Set appropriate scope boundaries'
            ],
            
            'architecture_design': [
                '✓ Identified all major components',
                '✓ Showed clear data flow',
                '✓ Made reasonable technology choices',
                '✓ Considered failure scenarios'
            ],
            
            'database_design': [
                '✓ Chose appropriate database type',
                '✓ Designed efficient schema',
                '✓ Planned for scalability',
                '✓ Considered consistency requirements'
            ],
            
            'interface_design': [
                '✓ Designed clean REST APIs',
                '✓ Considered error handling',
                '✓ Planned for versioning',
                '✓ Included authentication/authorization'
            ],
            
            'optimization': [
                '✓ Addressed scalability concerns',
                '✓ Designed caching strategy',
                '✓ Planned monitoring approach',
                '✓ Discussed trade-offs explicitly'
            ]
        }
```

### 2. Common Follow-up Questions
```python
class CommonFollowUpQuestions:
    def __init__(self):
        self.follow_up_categories = {
            'deep_dive_technical': [
                "How would you implement the caching layer?",
                "What happens when a database shard goes down?",
                "How do you ensure data consistency across services?",
                "How would you handle hot spots in your system?"
            ],
            
            'scalability_challenges': [
                "How would your system handle 10x more traffic?",
                "What bottlenecks do you anticipate at scale?",
                "How would you migrate from this architecture to microservices?",
                "How do you handle geographic distribution?"
            ],
            
            'failure_scenarios': [
                "What happens if your cache goes down?",
                "How do you handle network partitions?",
                "What's your disaster recovery strategy?",
                "How do you ensure zero-downtime deployments?"
            ],
            
            'monitoring_and_operations': [
                "What metrics would you monitor?",
                "How would you debug performance issues?",
                "What would your alerting strategy be?",
                "How do you handle capacity planning?"
            ]
        }
    
    def prepare_follow_up_responses(self):
        """Framework for handling follow-up questions"""
        
        return {
            'listen_carefully': 'Make sure you understand the specific question',
            'clarify_if_needed': 'Ask for clarification if question is ambiguous',
            'structure_response': 'Use a framework to organize your answer',
            'be_specific': 'Provide concrete examples and numbers',
            'discuss_trade_offs': 'Explain pros and cons of your approach',
            'relate_to_experience': 'Draw from real-world experience when possible'
        }
```

## Question-Specific Tips

### Tips by Question Type
```python
class QuestionSpecificTips:
    def __init__(self):
        self.tips_by_question_type = {
            'social_media_systems': {
                'key_focus_areas': [
                    'Fan-out strategies for timeline generation',
                    'Handling celebrity users differently',
                    'Real-time features (notifications, messaging)',
                    'Content moderation and safety'
                ],
                'common_pitfalls': [
                    'Not considering celebrity user scale',
                    'Ignoring content moderation requirements',
                    'Oversimplifying real-time requirements'
                ]
            },
            
            'messaging_systems': {
                'key_focus_areas': [
                    'Message delivery guarantees',
                    'Online vs offline message handling',
                    'Group messaging scalability',
                    'End-to-end encryption considerations'
                ],
                'common_pitfalls': [
                    'Not considering message ordering',
                    'Ignoring offline message storage',
                    'Oversimplifying group messaging'
                ]
            },
            
            'location_based_systems': {
                'key_focus_areas': [
                    'Geospatial indexing (geohash, quadtree)',
                    'Real-time location updates',
                    'Proximity search algorithms',
                    'Map data integration'
                ],
                'common_pitfalls': [
                    'Not considering geospatial indexing',
                    'Ignoring real-time update frequency',
                    'Oversimplifying proximity calculations'
                ]
            }
        }
```

## Exercise Problems

1. Practice the complete URL shortener question in 45 minutes using RADIO framework
2. Design a notification system with focus on multi-channel delivery
3. Practice follow-up questions for Twitter design (caching, scaling, failures)
4. Design a ride-sharing system with emphasis on real-time location tracking

## Key Takeaways

- **Know the classics**: Twitter, Instagram, Uber are extremely common
- **Practice with timer**: Time management is crucial
- **Prepare for follow-ups**: Deep dives test your technical depth  
- **Understand variations**: Same question can be asked differently
- **Study real systems**: Understanding how real companies solve problems helps
- **Practice explaining**: Communication is as important as technical knowledge
- **Learn from mistakes**: Each practice session should improve your approach
- **Adapt to interviewer**: Some prefer breadth, others prefer depth

## Next Steps

Move to: **04-communication-strategies.md**