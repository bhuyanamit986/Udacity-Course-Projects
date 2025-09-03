# Beginner Exercises

## Exercise 1: Design a URL Shortener (like bit.ly)

### Problem Statement
Design a web service that shortens URLs and redirects users to the original URL when they access the shortened version.

### Time Limit: 45 minutes

### Requirements Clarification Guide
```python
class URLShortenerRequirements:
    def clarification_questions(self):
        return [
            "How many URLs do we expect to shorten per month?",
            "What's the read/write ratio?", 
            "How long should URLs remain valid?",
            "Do we need custom aliases?",
            "Do we need analytics?",
            "What's the expected scale?"
        ]
    
    def sample_requirements(self):
        return {
            'functional': [
                'Shorten long URLs',
                'Redirect short URLs to original',
                'Optional custom aliases',
                'Basic click analytics'
            ],
            'non_functional': [
                '100M URLs shortened per month',
                '100:1 read to write ratio',
                'Redirection < 100ms',
                '99.9% availability'
            ]
        }
```

### Sample Solution Approach
```python
class URLShortenerSolution:
    def capacity_estimation(self):
        """Back-of-envelope calculations"""
        
        # Traffic
        urls_per_month = 100_000_000
        urls_per_second = urls_per_month / (30 * 24 * 3600)  # ~38 TPS
        redirects_per_second = urls_per_second * 100  # ~3800 RPS
        
        # Storage  
        url_size = 500  # bytes per URL record
        total_storage_5_years = urls_per_month * 12 * 5 * url_size
        storage_gb = total_storage_5_years / (1024**3)  # ~2.8 TB
        
        return {
            'write_tps': urls_per_second,
            'read_rps': redirects_per_second,
            'storage_gb_5_years': storage_gb
        }
    
    def high_level_design(self):
        """Major system components"""
        
        return {
            'components': [
                'Load Balancer',
                'Web Servers', 
                'Application Logic',
                'Database',
                'Cache'
            ],
            'data_flow': 'Client → LB → Web Server → Cache/DB → Response'
        }
    
    def database_design(self):
        """Database schema and choice"""
        
        return {
            'database_choice': 'NoSQL (DynamoDB/Cassandra)',
            'reasoning': 'Simple key-value operations, high read volume',
            'schema': {
                'url_mappings': {
                    'short_url': 'string (partition key)',
                    'original_url': 'string',
                    'created_at': 'timestamp',
                    'expires_at': 'timestamp',
                    'click_count': 'number'
                }
            },
            'indexing': 'Primary key on short_url for O(1) lookups'
        }
    
    def url_encoding_algorithm(self):
        """Algorithm for generating short URLs"""
        
        return {
            'approach_1_counter_based': {
                'method': 'Auto-incrementing counter + Base62 encoding',
                'pros': ['Short URLs', 'No collisions', 'Simple'],
                'cons': ['Predictable', 'Single point of failure for counter']
            },
            'approach_2_hash_based': {
                'method': 'Hash original URL + handle collisions',
                'pros': ['Distributed generation', 'No coordination needed'],
                'cons': ['Longer URLs', 'Collision handling complexity']
            },
            'recommended': 'Counter-based with distributed counter service'
        }
```

### Common Mistakes
- Starting with complex features before covering basics
- Not considering the read-heavy nature of URL redirects
- Ignoring caching for popular URLs
- Over-engineering the encoding algorithm
- Not discussing expiration and cleanup

### Extension Challenges
1. Add user authentication and personal URL management
2. Implement detailed analytics (geographic, time-based)
3. Add rate limiting to prevent abuse
4. Design bulk URL shortening API
5. Add real-time analytics dashboard

---

## Exercise 2: Design a Chat System

### Problem Statement  
Design a real-time chat system that supports one-on-one and group messaging.

### Time Limit: 50 minutes

### Requirements Clarification
```python
class ChatSystemRequirements:
    def clarification_questions(self):
        return [
            "One-on-one chat only or group chat too?",
            "How many users do we expect?",
            "Do we need message history?",
            "Do we need file sharing?",
            "What about presence (online/offline status)?",
            "Do we need push notifications?"
        ]
    
    def sample_requirements(self):
        return {
            'functional': [
                'Send and receive messages in real-time',
                'One-on-one and group chat (up to 100 users)',
                'Message delivery status (sent, delivered, read)',
                'User presence (online/offline)',
                'Message history',
                'File sharing (images, documents)'
            ],
            'non_functional': [
                '50M daily active users',
                'Message delivery < 1 second',
                '99.9% availability',
                'Support for mobile and web clients'
            ]
        }
```

### Sample Solution Approach
```python
class ChatSystemSolution:
    def architecture_design(self):
        """Chat system architecture"""
        
        return {
            'real_time_layer': [
                'WebSocket servers for real-time communication',
                'Connection management service',
                'Message routing service'
            ],
            'application_layer': [
                'User service (authentication, profiles)',
                'Chat service (message processing)',
                'Group service (group management)',
                'Notification service (push notifications)'
            ],
            'data_layer': [
                'User database (PostgreSQL)',
                'Message database (Cassandra)', 
                'File storage (S3)',
                'Cache (Redis)'
            ]
        }
    
    def message_delivery_design(self):
        """Message delivery system"""
        
        return {
            'online_delivery': {
                'technology': 'WebSockets',
                'fallback': 'HTTP long polling',
                'acknowledgments': 'Delivery receipts'
            },
            'offline_delivery': {
                'storage': 'Message queue for offline users',
                'push_notifications': 'Alert users of new messages',
                'sync_on_reconnect': 'Deliver queued messages'
            },
            'group_messaging': {
                'fan_out': 'Send to all group members',
                'optimization': 'Batch delivery for large groups'
            }
        }
```

### Common Mistakes
- Not considering offline message delivery
- Ignoring WebSocket connection management at scale
- Oversimplifying group message delivery
- Not addressing message ordering guarantees
- Missing presence system design

---

## Exercise 3: Design a Parking Lot System

### Problem Statement
Design a parking lot management system that tracks available spots and handles entry/exit.

### Time Limit: 40 minutes

### Sample Solution Approach
```python
class ParkingLotSystemSolution:
    def requirements_analysis(self):
        return {
            'functional': [
                'Track available parking spots',
                'Handle vehicle entry and exit',
                'Calculate parking fees',
                'Support different vehicle types',
                'Generate parking tickets',
                'Payment processing'
            ],
            'non_functional': [
                'Real-time spot availability',
                'Handle 1000 vehicles per hour',
                'System uptime 99.9%',
                'Fast entry/exit processing'
            ]
        }
    
    def system_design(self):
        return {
            'physical_components': [
                'Entry/exit gates with sensors',
                'Parking spot sensors',
                'Display boards',
                'Payment kiosks'
            ],
            'software_components': [
                'Parking management service',
                'Payment service',
                'Notification service',
                'Analytics service'
            ],
            'database_design': {
                'parking_lots': 'lot_id, name, total_spots, location',
                'parking_spots': 'spot_id, lot_id, spot_type, is_occupied',
                'vehicles': 'license_plate, vehicle_type, entry_time, spot_id',
                'transactions': 'transaction_id, vehicle_id, amount, timestamp'
            }
        }
```

---

## Exercise 4: Design a Rate Limiter

### Problem Statement
Design a rate limiting service that can limit requests based on user, IP, or API key.

### Time Limit: 35 minutes

### Sample Solution Approach
```python
class RateLimiterSolution:
    def algorithm_options(self):
        """Different rate limiting algorithms"""
        
        return {
            'token_bucket': {
                'description': 'Tokens added at fixed rate, consumed per request',
                'pros': ['Handles bursts', 'Smooth rate limiting'],
                'cons': ['Complex implementation', 'Memory per user'],
                'use_case': 'API rate limiting with burst allowance'
            },
            'sliding_window_log': {
                'description': 'Keep log of request timestamps',
                'pros': ['Very accurate', 'Flexible time windows'],
                'cons': ['High memory usage', 'Complex cleanup'],
                'use_case': 'Precise rate limiting requirements'
            },
            'fixed_window_counter': {
                'description': 'Counter resets at fixed intervals',
                'pros': ['Simple', 'Low memory usage'],
                'cons': ['Burst at window boundaries'],
                'use_case': 'Simple rate limiting with acceptable burst'
            }
        }
    
    def distributed_rate_limiter_design(self):
        """Design for distributed rate limiting"""
        
        return {
            'centralized_approach': {
                'implementation': 'Redis with atomic operations',
                'pros': ['Accurate', 'Consistent across servers'],
                'cons': ['Single point of failure', 'Network latency']
            },
            'distributed_approach': {
                'implementation': 'Local counters with periodic sync',
                'pros': ['No single point of failure', 'Low latency'],
                'cons': ['Less accurate', 'Complex synchronization']
            },
            'hybrid_approach': {
                'implementation': 'Local cache + periodic Redis sync',
                'pros': ['Balance of accuracy and performance'],
                'cons': ['Implementation complexity']
            }
        }
```

---

## Exercise 5: Design a Library Management System

### Problem Statement
Design a system to manage library operations including book catalog, member management, and borrowing/returning books.

### Time Limit: 40 minutes

### Sample Solution Approach
```python
class LibraryManagementSolution:
    def system_requirements(self):
        return {
            'functional': [
                'Manage book catalog',
                'Member registration and management',
                'Book borrowing and returning',
                'Search books by title, author, ISBN',
                'Track due dates and overdue books',
                'Fine calculation and payment',
                'Reservation system for popular books'
            ],
            'non_functional': [
                '10K active members',
                '100K books in catalog',
                'Fast search response < 100ms',
                'System availability 99.9%'
            ]
        }
    
    def database_design(self):
        return {
            'books': {
                'book_id': 'PRIMARY KEY',
                'isbn': 'UNIQUE',
                'title': 'string',
                'author': 'string',
                'category': 'string',
                'total_copies': 'integer',
                'available_copies': 'integer'
            },
            'members': {
                'member_id': 'PRIMARY KEY',
                'name': 'string',
                'email': 'UNIQUE',
                'phone': 'string',
                'membership_date': 'date',
                'status': 'active/suspended'
            },
            'borrowings': {
                'borrowing_id': 'PRIMARY KEY',
                'book_id': 'FOREIGN KEY',
                'member_id': 'FOREIGN KEY', 
                'borrowed_date': 'date',
                'due_date': 'date',
                'returned_date': 'date (nullable)',
                'fine_amount': 'decimal'
            }
        }
```

## Practice Guidelines

### How to Practice Effectively
```python
class PracticeGuidelines:
    def __init__(self):
        self.practice_methodology = {
            'preparation': [
                'Set up timer for realistic practice',
                'Have whiteboard or drawing tool ready',
                'Review relevant concepts before starting',
                'Clear your mind and focus'
            ],
            
            'during_practice': [
                'Follow RADIO framework strictly',
                'Think out loud as if interviewer is present',
                'Draw diagrams to support explanations',
                'Check time every 10 minutes',
                'Don\'t get stuck on perfect solutions'
            ],
            
            'after_practice': [
                'Review your solution critically',
                'Compare with sample solutions',
                'Identify areas for improvement',
                'Note new concepts learned',
                'Plan follow-up study topics'
            ]
        }
    
    def self_evaluation_criteria(self):
        """Criteria for evaluating your practice sessions"""
        
        return {
            'requirements_gathering': {
                'excellent': 'Asked all relevant questions, clear requirements',
                'good': 'Asked most important questions, mostly clear requirements',
                'needs_improvement': 'Missed key questions, unclear requirements'
            },
            
            'system_design': {
                'excellent': 'Clean architecture, appropriate technology choices',
                'good': 'Reasonable architecture, justified choices',
                'needs_improvement': 'Unclear architecture, poor technology choices'
            },
            
            'scalability': {
                'excellent': 'Comprehensive scaling strategy, identified bottlenecks',
                'good': 'Basic scaling considerations, some bottlenecks identified',
                'needs_improvement': 'Limited scaling discussion, missed bottlenecks'
            },
            
            'communication': {
                'excellent': 'Clear explanations, good use of diagrams',
                'good': 'Mostly clear, adequate diagrams',
                'needs_improvement': 'Unclear explanations, poor diagrams'
            }
        }
```

## Progressive Difficulty

### Exercise Progression Strategy
```python
class ExerciseProgression:
    def __init__(self):
        self.progression_path = {
            'week_1_foundations': [
                'URL Shortener (simple key-value system)',
                'Chat System (real-time messaging basics)',
                'Parking Lot (state management system)'
            ],
            
            'week_2_web_systems': [
                'Web Crawler (distributed processing)',
                'Rate Limiter (distributed algorithms)',
                'Library Management (CRUD operations with business logic)'
            ],
            
            'week_3_social_systems': [
                'Twitter (social media fundamentals)',
                'Instagram (media-heavy social system)',
                'LinkedIn (professional networking)'
            ],
            
            'week_4_complex_systems': [
                'Uber (real-time location systems)',
                'Netflix (content delivery and recommendations)',
                'Amazon (e-commerce platform)'
            ]
        }
    
    def skill_development_focus(self):
        """Skills developed at each level"""
        
        return {
            'beginner_skills': [
                'Basic system components',
                'Simple database design',
                'Load balancing concepts',
                'Basic caching strategies'
            ],
            
            'intermediate_skills': [
                'Microservices architecture',
                'Message queues and async processing',
                'Database scaling patterns',
                'API design principles'
            ],
            
            'advanced_skills': [
                'Distributed system patterns',
                'Consensus algorithms',
                'Complex trade-off analysis',
                'Global system design'
            ]
        }
```

## Exercise Solutions and Explanations

### URL Shortener - Detailed Solution
```python
class URLShortenerDetailedSolution:
    def complete_solution(self):
        """Complete solution with explanations"""
        
        return {
            'encoding_service': {
                'counter_based_approach': '''
                class URLEncoder:
                    def __init__(self):
                        self.counter = DistributedCounter()
                        self.base62_chars = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
                    
                    def encode_url(self, original_url):
                        # Get next unique ID
                        url_id = self.counter.get_next_id()
                        
                        # Encode to base62
                        short_code = self.base62_encode(url_id)
                        
                        # Store mapping
                        self.store_mapping(short_code, original_url)
                        
                        return f"https://short.ly/{short_code}"
                ''',
                'explanation': 'Counter-based approach ensures short URLs and no collisions'
            },
            
            'caching_strategy': {
                'hot_url_caching': '''
                class URLCache:
                    def __init__(self, redis_client):
                        self.cache = redis_client
                    
                    def get_original_url(self, short_code):
                        # Try cache first
                        original_url = self.cache.get(f"url:{short_code}")
                        
                        if original_url:
                            # Update access count
                            self.cache.incr(f"count:{short_code}")
                            return original_url
                        
                        # Cache miss - fetch from database
                        original_url = self.database.get_original_url(short_code)
                        
                        if original_url:
                            # Cache for 1 hour
                            self.cache.setex(f"url:{short_code}", 3600, original_url)
                        
                        return original_url
                ''',
                'explanation': 'Cache popular URLs to reduce database load'
            },
            
            'scaling_strategy': {
                'database_scaling': 'Shard by hash of short_code',
                'application_scaling': 'Stateless servers behind load balancer',
                'global_scaling': 'Regional deployments with CDN'
            }
        }
```

### Chat System - Detailed Solution
```python
class ChatSystemDetailedSolution:
    def websocket_architecture(self):
        """WebSocket-based real-time messaging"""
        
        return {
            'connection_management': '''
            class WebSocketConnectionManager:
                def __init__(self):
                    self.connections = {}  # user_id -> websocket
                    self.user_servers = {}  # user_id -> server_id
                
                def add_connection(self, user_id, websocket):
                    self.connections[user_id] = websocket
                    self.user_servers[user_id] = self.get_server_id()
                
                def send_message_to_user(self, user_id, message):
                    if user_id in self.connections:
                        # User connected to this server
                        websocket = self.connections[user_id]
                        websocket.send(json.dumps(message))
                    else:
                        # User connected to different server
                        target_server = self.user_servers.get(user_id)
                        if target_server:
                            self.forward_to_server(target_server, user_id, message)
                        else:
                            # User offline - queue message
                            self.queue_offline_message(user_id, message)
            ''',
            
            'message_routing': '''
            class MessageRouter:
                def route_message(self, sender_id, recipient_id, message):
                    # Create message record
                    message_record = {
                        'message_id': str(uuid.uuid4()),
                        'sender_id': sender_id,
                        'recipient_id': recipient_id,
                        'content': message['content'],
                        'timestamp': time.time()
                    }
                    
                    # Store message
                    self.message_storage.store_message(message_record)
                    
                    # Attempt delivery
                    self.connection_manager.send_message_to_user(
                        recipient_id, 
                        message_record
                    )
                    
                    return message_record['message_id']
            '''
        }
```

## Self-Practice Framework

### Practice Session Structure
```python
class PracticeSessionFramework:
    def __init__(self, question, time_limit_minutes):
        self.question = question
        self.time_limit = time_limit_minutes
        self.session_log = []
    
    def conduct_practice_session(self):
        """Structure for self-practice session"""
        
        session_structure = {
            'setup_phase': {
                'duration_minutes': 2,
                'activities': [
                    'Read question carefully',
                    'Set timer',
                    'Prepare whiteboard/drawing tool'
                ]
            },
            
            'requirements_phase': {
                'duration_minutes': self.time_limit * 0.15,  # 15% of time
                'activities': [
                    'Ask clarifying questions (to yourself)',
                    'Define functional requirements',
                    'Identify non-functional requirements',
                    'Set scope boundaries'
                ]
            },
            
            'design_phase': {
                'duration_minutes': self.time_limit * 0.70,  # 70% of time
                'activities': [
                    'Capacity estimation',
                    'High-level architecture',
                    'Database design',
                    'API design',
                    'Detailed component design'
                ]
            },
            
            'review_phase': {
                'duration_minutes': self.time_limit * 0.15,  # 15% of time
                'activities': [
                    'Review design for completeness',
                    'Identify potential issues',
                    'Consider scaling and optimization',
                    'Prepare for follow-up questions'
                ]
            }
        }
        
        return session_structure
    
    def post_session_evaluation(self):
        """Evaluate practice session"""
        
        evaluation_questions = [
            "Did I clarify requirements adequately?",
            "Is my architecture diagram clear and complete?",
            "Did I justify my technology choices?",
            "Did I consider scalability and failure scenarios?",
            "Did I manage time effectively?",
            "What would I do differently next time?"
        ]
        
        return evaluation_questions
```

## Common Beginner Mistakes and Solutions

### Mistake Analysis Framework
```python
class BeginnerMistakeAnalysis:
    def __init__(self):
        self.common_mistakes = {
            'requirements_phase': {
                'jumping_to_solution': {
                    'mistake': 'Starting design without clarifying requirements',
                    'solution': 'Always spend time understanding the problem first',
                    'example': 'Assuming Twitter needs real-time DMs without asking'
                },
                'scope_creep': {
                    'mistake': 'Adding features not mentioned in requirements',
                    'solution': 'Stick to defined scope, mention extensions separately',
                    'example': 'Adding analytics to URL shortener without requirement'
                }
            },
            
            'architecture_phase': {
                'monolithic_thinking': {
                    'mistake': 'Designing everything as single service',
                    'solution': 'Consider service boundaries based on functionality',
                    'example': 'Single service for user management, posts, and messaging'
                },
                'missing_load_balancer': {
                    'mistake': 'Single server handling all traffic',
                    'solution': 'Always include load balancing for scalable systems',
                    'example': 'Direct client connections to application server'
                }
            },
            
            'database_phase': {
                'wrong_database_choice': {
                    'mistake': 'Choosing database without justification',
                    'solution': 'Explain database choice based on data patterns',
                    'example': 'Using NoSQL for complex relational data'
                },
                'no_indexing_consideration': {
                    'mistake': 'Not considering query performance',
                    'solution': 'Design indexes based on query patterns',
                    'example': 'No index on frequently queried columns'
                }
            }
        }
    
    def improvement_strategies(self):
        """Strategies to improve common weak areas"""
        
        return {
            'requirements_gathering': [
                'Practice asking clarifying questions',
                'Study real product requirements',
                'Learn to identify implicit requirements',
                'Practice scoping exercises'
            ],
            
            'architecture_design': [
                'Study common architecture patterns',
                'Practice drawing system diagrams',
                'Learn standard component interactions',
                'Understand when to use which patterns'
            ],
            
            'database_design': [
                'Practice SQL and NoSQL schema design',
                'Learn database scaling patterns',
                'Understand consistency models',
                'Practice capacity calculations'
            ],
            
            'communication': [
                'Practice explaining technical concepts simply',
                'Record yourself doing mock interviews',
                'Get feedback from experienced engineers',
                'Practice drawing clear diagrams quickly'
            ]
        }
```

## Exercise Variations

### Adding Complexity Gradually
```python
class ExerciseVariations:
    def url_shortener_variations(self):
        """Progressive complexity for URL shortener"""
        
        return {
            'basic_version': [
                'Simple URL shortening and redirection',
                'Basic database storage',
                'Single server deployment'
            ],
            
            'intermediate_version': [
                'Add user authentication',
                'Custom aliases support',
                'Basic analytics (click count)',
                'Caching for popular URLs'
            ],
            
            'advanced_version': [
                'Detailed analytics dashboard',
                'Bulk URL operations',
                'Rate limiting and abuse prevention',
                'Global deployment with CDN',
                'Real-time analytics'
            ]
        }
    
    def chat_system_variations(self):
        """Progressive complexity for chat system"""
        
        return {
            'basic_version': [
                'One-on-one messaging',
                'Simple WebSocket implementation',
                'Basic message storage'
            ],
            
            'intermediate_version': [
                'Group messaging',
                'Message delivery status',
                'User presence system',
                'File sharing capability'
            ],
            
            'advanced_version': [
                'End-to-end encryption',
                'Message search functionality',
                'Voice and video calling',
                'Message synchronization across devices',
                'Advanced group management'
            ]
        }
```

## Exercise Problems

### Practice Schedule
```python
class BeginnerPracticeSchedule:
    def week_1_schedule(self):
        """First week practice schedule"""
        
        return {
            'day_1': {
                'exercise': 'URL Shortener',
                'focus': 'Requirements and basic architecture',
                'time_limit': '45 minutes'
            },
            'day_2': {
                'exercise': 'URL Shortener (retry)',
                'focus': 'Improve based on day 1 feedback',
                'time_limit': '40 minutes'
            },
            'day_3': {
                'exercise': 'Chat System',
                'focus': 'Real-time systems introduction',
                'time_limit': '50 minutes'
            },
            'day_4': {
                'exercise': 'Parking Lot System',
                'focus': 'State management and sensors',
                'time_limit': '40 minutes'
            },
            'day_5': {
                'exercise': 'Rate Limiter',
                'focus': 'Algorithms and distributed systems',
                'time_limit': '35 minutes'
            },
            'weekend': {
                'activity': 'Review all exercises and study weak areas',
                'focus': 'Consolidate learning and identify improvement areas'
            }
        }
```

## Key Takeaways

- **Start simple**: Master basic systems before moving to complex ones
- **Time management**: Practice with realistic time constraints
- **Iterative improvement**: Each practice session should build on previous ones
- **Focus on fundamentals**: Strong basics enable tackling complex problems
- **Self-evaluation**: Honest assessment leads to faster improvement
- **Pattern recognition**: Similar patterns appear across different systems
- **Communication practice**: Explaining solutions is as important as designing them

## Next Steps

Once comfortable with beginner exercises, move to: **02-intermediate-exercises.md**