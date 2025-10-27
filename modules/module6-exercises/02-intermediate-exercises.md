# Intermediate Exercises

## Exercise 1: Design Twitter (Most Important Interview Question)

### Problem Statement
Design a social media platform where users can post short messages, follow other users, and view a timeline of posts from people they follow.

### Time Limit: 60 minutes

### Complete Solution Framework
```python
class TwitterDesignSolution:
    def requirements_clarification(self):
        """Systematic requirements gathering"""
        
        return {
            'clarifying_questions': [
                "What are the core features? (post, follow, timeline, search)",
                "How many users? (assume 100M DAU)",
                "What's the read/write ratio? (assume 300:1)",
                "Character limit for posts? (assume 280 chars)",
                "Do we need media support? (yes, images/videos)",
                "Real-time timeline updates? (yes)",
                "Global deployment? (yes)"
            ],
            
            'functional_requirements': [
                'Users can post tweets (280 chars + media)',
                'Users can follow/unfollow other users',
                'Users can view home timeline (posts from followed users)',
                'Users can view user profiles and their tweets',
                'Users can search tweets and users',
                'Users can like and retweet posts',
                'Users receive notifications for interactions'
            ],
            
            'non_functional_requirements': [
                '100M daily active users',
                '300:1 read to write ratio',
                'Home timeline load time < 200ms',
                '99.9% availability',
                'Global deployment with low latency'
            ]
        }
    
    def capacity_estimation(self):
        """Detailed capacity calculations"""
        
        # User metrics
        dau = 100_000_000
        mau = dau * 2  # 200M monthly active users
        
        # Tweet metrics
        tweets_per_user_per_day = 2
        total_tweets_per_day = dau * tweets_per_user_per_day  # 200M tweets/day
        tweets_per_second = total_tweets_per_day / (24 * 3600)  # ~2.3K TPS
        peak_tweets_per_second = tweets_per_second * 3  # ~7K TPS
        
        # Timeline metrics
        timeline_requests_per_user_per_day = 50
        total_timeline_requests = dau * timeline_requests_per_user_per_day
        timeline_rps = total_timeline_requests / (24 * 3600)  # ~58K RPS
        peak_timeline_rps = timeline_rps * 3  # ~174K RPS
        
        # Storage estimation
        tweet_metadata_size = 200  # bytes
        avg_tweet_content_size = 100  # bytes
        media_attachment_size = 1024 * 1024  # 1MB average
        tweets_with_media_ratio = 0.1  # 10% tweets have media
        
        daily_tweet_storage = (
            total_tweets_per_day * (tweet_metadata_size + avg_tweet_content_size) +
            total_tweets_per_day * tweets_with_media_ratio * media_attachment_size
        )
        
        return {
            'traffic': {
                'peak_write_tps': peak_tweets_per_second,
                'peak_read_rps': peak_timeline_rps,
                'read_write_ratio': 300
            },
            'storage': {
                'daily_storage_gb': daily_storage / (1024**3),
                'monthly_storage_tb': daily_storage * 30 / (1024**4),
                'yearly_storage_tb': daily_storage * 365 / (1024**4)
            }
        }
```

### Twitter Timeline Generation Deep Dive
```python
class TwitterTimelineGeneration:
    def fan_out_strategies(self):
        """Compare different fan-out strategies"""
        
        return {
            'push_model_fan_out_on_write': {
                'description': 'Pre-compute timelines when tweets are posted',
                'algorithm': '''
                def fan_out_tweet(tweet):
                    followers = get_followers(tweet.user_id)
                    
                    for follower_id in followers:
                        timeline_cache.add_tweet_to_timeline(follower_id, tweet)
                ''',
                'pros': ['Fast timeline reads', 'Simple timeline generation'],
                'cons': ['Slow writes for users with many followers', 'Storage overhead'],
                'best_for': 'Regular users with moderate follower count'
            },
            
            'pull_model_fan_out_on_read': {
                'description': 'Generate timeline when user requests it',
                'algorithm': '''
                def generate_timeline(user_id):
                    following = get_following(user_id)
                    timeline_tweets = []
                    
                    for followed_user_id in following:
                        recent_tweets = get_recent_tweets(followed_user_id, limit=10)
                        timeline_tweets.extend(recent_tweets)
                    
                    timeline_tweets.sort(key=lambda t: t.timestamp, reverse=True)
                    return timeline_tweets[:50]
                ''',
                'pros': ['Fast writes', 'No storage overhead', 'Handles celebrity users well'],
                'cons': ['Slow timeline reads', 'Complex timeline generation'],
                'best_for': 'Celebrity users with millions of followers'
            },
            
            'hybrid_approach': {
                'description': 'Combine push and pull based on user type',
                'algorithm': '''
                def hybrid_fan_out(tweet):
                    follower_count = get_follower_count(tweet.user_id)
                    
                    if follower_count > 1_000_000:  # Celebrity
                        # Don't fan out, let followers pull
                        mark_as_celebrity_tweet(tweet)
                    else:
                        # Regular fan out
                        fan_out_to_followers(tweet)
                
                def generate_timeline(user_id):
                    # Get pre-computed timeline (push model)
                    timeline = get_precomputed_timeline(user_id)
                    
                    # Add celebrity tweets (pull model)
                    celebrity_following = get_celebrity_following(user_id)
                    for celebrity_id in celebrity_following:
                        celebrity_tweets = get_recent_tweets(celebrity_id, limit=5)
                        timeline.extend(celebrity_tweets)
                    
                    timeline.sort(key=lambda t: t.timestamp, reverse=True)
                    return timeline
                ''',
                'pros': ['Optimized for different user types', 'Balanced performance'],
                'cons': ['Complex implementation', 'Multiple code paths'],
                'best_for': 'Large-scale social media platforms'
            }
        }
```

---

## Exercise 2: Design Instagram

### Problem Statement
Design a photo and video sharing social media platform with feed, stories, and messaging features.

### Time Limit: 60 minutes

### Instagram-Specific Considerations
```python
class InstagramDesignSolution:
    def unique_challenges(self):
        """Challenges specific to Instagram vs Twitter"""
        
        return {
            'media_heavy_workload': {
                'challenge': 'Large file uploads and storage',
                'solutions': [
                    'Async media processing pipeline',
                    'Multiple image resolutions',
                    'CDN for global media delivery',
                    'Progressive image loading'
                ]
            },
            
            'mobile_first_design': {
                'challenge': 'Optimized for mobile data usage',
                'solutions': [
                    'Image compression optimization',
                    'Adaptive quality based on connection',
                    'Lazy loading for feeds',
                    'Offline caching for viewed content'
                ]
            },
            
            'visual_content_discovery': {
                'challenge': 'Different engagement patterns than text',
                'solutions': [
                    'Image-based recommendation algorithms',
                    'Hashtag and location-based discovery',
                    'Computer vision for content analysis',
                    'Explore page algorithm'
                ]
            }
        }
    
    def media_processing_pipeline(self):
        """Design media processing pipeline"""
        
        return {
            'upload_flow': '''
            1. Client uploads image/video to CDN
            2. CDN stores original and triggers processing
            3. Media processing service generates:
               - Multiple resolutions (thumbnail, medium, full)
               - Different formats (WebP, JPEG for compatibility)
               - Compressed versions for mobile
            4. Processed media stored in CDN
            5. Database updated with media URLs
            6. User notified of completion
            ''',
            
            'processing_requirements': [
                'Image resizing (150x150, 320x320, 640x640, 1080x1080)',
                'Video transcoding (multiple bitrates)',
                'Format optimization (WebP for supported browsers)',
                'Content moderation (AI-based inappropriate content detection)',
                'Metadata extraction (EXIF data, location, etc.)'
            ],
            
            'storage_strategy': [
                'Original media in S3 (backup)',
                'Processed media in CDN',
                'Metadata in application database',
                'Hot media in edge caches'
            ]
        }
```

### Instagram Feed Algorithm
```python
class InstagramFeedAlgorithm:
    def feed_ranking_factors(self):
        """Factors that influence Instagram feed ranking"""
        
        return {
            'engagement_signals': [
                'Likes, comments, shares on post',
                'User\'s past engagement with poster',
                'Overall engagement rate of poster',
                'Recency of post'
            ],
            
            'relationship_signals': [
                'How often user interacts with poster',
                'Whether users are close friends',
                'Whether user searches for poster',
                'Direct message frequency'
            ],
            
            'content_signals': [
                'Post type (photo, video, carousel)',
                'Hashtags used',
                'Location tags',
                'Content quality (blur detection, etc.)'
            ],
            
            'user_behavior_signals': [
                'Time spent viewing similar content',
                'User\'s active hours',
                'Device type and connection speed',
                'Historical preferences'
            ]
        }
    
    def feed_generation_algorithm(self):
        """Algorithm for generating personalized feed"""
        
        return '''
        def generate_feed(user_id, page_size=20):
            # Get candidate posts
            following = get_following(user_id)
            candidate_posts = []
            
            for followed_user in following:
                recent_posts = get_recent_posts(followed_user, limit=10)
                candidate_posts.extend(recent_posts)
            
            # Score each post
            scored_posts = []
            for post in candidate_posts:
                score = calculate_post_score(user_id, post)
                scored_posts.append((post, score))
            
            # Sort by score and return top posts
            scored_posts.sort(key=lambda x: x[1], reverse=True)
            return [post for post, score in scored_posts[:page_size]]
        
        def calculate_post_score(user_id, post):
            # Engagement score
            engagement_score = (
                post.like_count * 1.0 +
                post.comment_count * 2.0 +
                post.share_count * 3.0
            )
            
            # Relationship score
            relationship_score = get_relationship_strength(user_id, post.user_id)
            
            # Recency score (decay over time)
            hours_old = (current_time - post.created_at) / 3600
            recency_score = 1.0 / (1.0 + hours_old * 0.1)
            
            # Content type score
            content_score = get_content_type_preference(user_id, post.type)
            
            # Combine scores
            total_score = (
                engagement_score * 0.3 +
                relationship_score * 0.4 +
                recency_score * 0.2 +
                content_score * 0.1
            )
            
            return total_score
        '''
```

---

## Exercise 3: Design Uber

### Problem Statement
Design a ride-sharing platform that connects drivers with passengers for on-demand transportation.

### Time Limit: 60 minutes

### Location System Deep Dive
```python
class UberLocationSystemSolution:
    def real_time_location_tracking(self):
        """Design real-time location tracking system"""
        
        return {
            'location_update_frequency': {
                'driver_states': {
                    'online_available': '8 seconds',
                    'on_trip': '4 seconds', 
                    'offline': 'no updates'
                },
                'optimization': 'Reduce frequency when not moving'
            },
            
            'geospatial_indexing': {
                'technology': 'Redis with geospatial commands',
                'implementation': '''
                # Store driver locations
                redis.geoadd("driver_locations", longitude, latitude, driver_id)
                
                # Find nearby drivers
                nearby = redis.georadius(
                    "driver_locations", 
                    pickup_lng, pickup_lat, 
                    radius_km, unit="km",
                    withcoord=True, withdist=True
                )
                ''',
                'backup_indexing': 'Geohash-based indexing for fallback'
            },
            
            'location_storage': {
                'real_time_storage': 'Redis for current locations',
                'historical_storage': 'Cassandra for trip history',
                'data_retention': '30 days for historical locations'
            }
        }
    
    def driver_matching_algorithm(self):
        """Algorithm for matching drivers to ride requests"""
        
        return '''
        def match_driver_to_ride(ride_request):
            # Find nearby available drivers
            nearby_drivers = find_drivers_within_radius(
                ride_request.pickup_lat,
                ride_request.pickup_lng,
                radius_km=10
            )
            
            # Filter available drivers
            available_drivers = [
                driver for driver in nearby_drivers
                if driver.status == "available" and 
                   driver.vehicle_type == ride_request.vehicle_type
            ]
            
            if not available_drivers:
                return None
            
            # Score drivers based on multiple factors
            scored_drivers = []
            for driver in available_drivers:
                score = calculate_driver_score(ride_request, driver)
                scored_drivers.append((driver, score))
            
            # Return best driver
            scored_drivers.sort(key=lambda x: x[1], reverse=True)
            return scored_drivers[0][0]
        
        def calculate_driver_score(ride_request, driver):
            # Distance factor (closer is better)
            distance_score = max(0, 10 - driver.distance_km)
            
            # Driver rating
            rating_score = driver.rating * 2
            
            # ETA factor
            eta_score = max(0, 10 - driver.eta_minutes / 2)
            
            # Driver utilization (balance workload)
            utilization_score = 10 - driver.trips_today
            
            return (distance_score * 0.4 + 
                   rating_score * 0.3 + 
                   eta_score * 0.2 + 
                   utilization_score * 0.1)
        '''
```

### Uber Surge Pricing System
```python
class UberSurgePricingSystem:
    def surge_calculation_algorithm(self):
        """Algorithm for calculating surge pricing"""
        
        return {
            'supply_demand_analysis': '''
            def calculate_surge_multiplier(location_area):
                # Get current supply and demand
                available_drivers = count_available_drivers_in_area(location_area)
                pending_requests = count_pending_ride_requests_in_area(location_area)
                
                # Calculate supply/demand ratio
                if pending_requests == 0:
                    return 1.0  # No surge when no demand
                
                supply_demand_ratio = available_drivers / pending_requests
                
                # Surge calculation based on ratio
                if supply_demand_ratio >= 1.5:
                    surge_multiplier = 1.0
                elif supply_demand_ratio >= 1.0:
                    surge_multiplier = 1.2
                elif supply_demand_ratio >= 0.5:
                    surge_multiplier = 1.5
                elif supply_demand_ratio >= 0.25:
                    surge_multiplier = 2.0
                else:
                    surge_multiplier = min(3.0, 4.0 / supply_demand_ratio)
                
                # Apply external factors
                surge_multiplier = apply_external_factors(surge_multiplier, location_area)
                
                # Smooth transitions to avoid price shock
                previous_surge = get_previous_surge(location_area)
                return smooth_surge_transition(previous_surge, surge_multiplier)
            ''',
            
            'external_factors': [
                'Weather conditions (rain/snow increases demand)',
                'Events (concerts, sports games)',
                'Time of day (rush hours)',
                'Day of week (weekends vs weekdays)',
                'Holidays and special occasions'
            ],
            
            'surge_smoothing': {
                'max_change_per_minute': 0.1,
                'reasoning': 'Prevent dramatic price swings that shock users'
            }
        }
```

---

## Exercise 4: Design Instagram

### Problem Statement
Design a photo and video sharing platform with social features like following, liking, and commenting.

### Time Limit: 60 minutes

### Media Processing Architecture
```python
class InstagramMediaProcessing:
    def upload_and_processing_pipeline(self):
        """Complete media upload and processing pipeline"""
        
        return {
            'client_side_optimization': [
                'Image compression before upload',
                'Progressive upload for large files',
                'Retry mechanism for failed uploads',
                'Background upload for better UX'
            ],
            
            'server_side_processing': '''
            class MediaProcessingPipeline:
                def process_uploaded_media(self, media_file, user_id):
                    # Store original file
                    original_url = self.store_original(media_file)
                    
                    # Queue processing job
                    job_id = self.queue_processing_job({
                        'media_file_url': original_url,
                        'user_id': user_id,
                        'processing_requirements': self.get_processing_requirements(media_file)
                    })
                    
                    return {
                        'upload_id': str(uuid.uuid4()),
                        'status': 'processing',
                        'job_id': job_id
                    }
                
                def process_media_async(self, job_data):
                    media_url = job_data['media_file_url']
                    
                    # Generate multiple resolutions
                    resolutions = self.generate_resolutions(media_url)
                    
                    # Apply filters and optimizations
                    optimized_versions = self.apply_optimizations(resolutions)
                    
                    # Store processed versions
                    processed_urls = self.store_processed_media(optimized_versions)
                    
                    # Update database
                    self.update_media_record(job_data['upload_id'], processed_urls)
                    
                    # Notify user of completion
                    self.notify_processing_complete(job_data['user_id'], job_data['upload_id'])
            ''',
            
            'processing_outputs': {
                'image_resolutions': ['150x150 (thumbnail)', '320x320 (small)', '640x640 (medium)', '1080x1080 (full)'],
                'video_qualities': ['240p', '480p', '720p', '1080p'],
                'formats': ['Original format', 'WebP (when supported)', 'JPEG fallback']
            }
        }
```

---

## Exercise 5: Design a Web Crawler

### Problem Statement
Design a web crawler that can crawl billions of web pages efficiently and keep the index fresh.

### Time Limit: 55 minutes

### Web Crawler Architecture
```python
class WebCrawlerSolution:
    def crawler_architecture(self):
        """Distributed web crawler architecture"""
        
        return {
            'crawler_components': [
                'URL Frontier (manages URLs to crawl)',
                'Fetcher Service (downloads web pages)',
                'Content Processor (extracts links and content)',
                'Duplicate Detector (avoids crawling same content)',
                'Politeness Manager (respects robots.txt)',
                'Storage Service (stores crawled content)'
            ],
            
            'distributed_crawling': '''
            class DistributedCrawler:
                def __init__(self):
                    self.url_frontier = URLFrontier()
                    self.fetcher_pool = FetcherPool(num_workers=1000)
                    self.content_processor = ContentProcessor()
                    self.duplicate_detector = DuplicateDetector()
                
                def crawl_continuously(self):
                    while True:
                        # Get batch of URLs to crawl
                        urls_to_crawl = self.url_frontier.get_next_batch(size=100)
                        
                        if not urls_to_crawl:
                            time.sleep(60)  # Wait for new URLs
                            continue
                        
                        # Distribute to worker pool
                        crawl_tasks = []
                        for url in urls_to_crawl:
                            task = self.fetcher_pool.submit_crawl_task(url)
                            crawl_tasks.append(task)
                        
                        # Process results
                        for task in crawl_tasks:
                            result = task.get_result()
                            self.process_crawl_result(result)
            ''',
            
            'politeness_policies': [
                'Respect robots.txt directives',
                'Implement crawl delay between requests to same domain',
                'Distribute load across different domains',
                'Handle rate limiting gracefully'
            ]
        }
    
    def url_frontier_design(self):
        """Design URL frontier for managing crawl queue"""
        
        return {
            'prioritization_strategy': '''
            class URLFrontier:
                def __init__(self):
                    self.priority_queues = {
                        'high': PriorityQueue(),    # News sites, frequently updated
                        'medium': PriorityQueue(),  # Regular websites
                        'low': PriorityQueue()      # Static content, archives
                    }
                    self.domain_queues = {}  # Separate queue per domain for politeness
                
                def add_url(self, url, priority='medium'):
                    domain = extract_domain(url)
                    
                    # Add to domain-specific queue for politeness
                    if domain not in self.domain_queues:
                        self.domain_queues[domain] = Queue()
                    
                    self.domain_queues[domain].put(url)
                    
                    # Add to priority queue
                    self.priority_queues[priority].put({
                        'url': url,
                        'domain': domain,
                        'added_at': time.time()
                    })
                
                def get_next_urls(self, batch_size=100):
                    urls = []
                    domain_last_crawl = {}  # Track last crawl time per domain
                    
                    # Try to get URLs from different domains
                    for priority in ['high', 'medium', 'low']:
                        while len(urls) < batch_size and not self.priority_queues[priority].empty():
                            url_item = self.priority_queues[priority].get()
                            domain = url_item['domain']
                            
                            # Check politeness delay
                            if self.can_crawl_domain(domain, domain_last_crawl):
                                urls.append(url_item['url'])
                                domain_last_crawl[domain] = time.time()
                    
                    return urls
            ''',
            
            'freshness_management': [
                'Track last crawl time for each URL',
                'Prioritize frequently updated content',
                'Implement exponential backoff for stable content',
                'Use sitemaps for crawl scheduling hints'
            ]
        }
```

---

## Exercise 6: Design a File Storage System (like Dropbox)

### Problem Statement
Design a cloud file storage system that allows users to upload, download, and sync files across multiple devices.

### Time Limit: 60 minutes

### File Storage System Architecture
```python
class FileStorageSystemSolution:
    def system_architecture(self):
        """Complete file storage system architecture"""
        
        return {
            'core_services': [
                'File Upload Service',
                'File Download Service', 
                'Metadata Service',
                'Sync Service',
                'Version Control Service',
                'Sharing Service'
            ],
            
            'storage_architecture': {
                'file_storage': 'Distributed object storage (S3-like)',
                'metadata_storage': 'SQL database for file metadata',
                'cache_storage': 'Redis for hot file metadata',
                'cdn': 'Global CDN for file downloads'
            },
            
            'sync_mechanism': '''
            class FileSyncService:
                def sync_file_changes(self, user_id, device_id):
                    # Get last sync timestamp for device
                    last_sync = self.get_last_sync_timestamp(user_id, device_id)
                    
                    # Get all changes since last sync
                    changes = self.get_file_changes_since(user_id, last_sync)
                    
                    sync_operations = []
                    
                    for change in changes:
                        if change.type == 'file_added':
                            sync_operations.append({
                                'operation': 'download',
                                'file_id': change.file_id,
                                'file_path': change.file_path
                            })
                        elif change.type == 'file_modified':
                            sync_operations.append({
                                'operation': 'update',
                                'file_id': change.file_id,
                                'new_version': change.version
                            })
                        elif change.type == 'file_deleted':
                            sync_operations.append({
                                'operation': 'delete',
                                'file_path': change.file_path
                            })
                    
                    # Update last sync timestamp
                    self.update_last_sync_timestamp(user_id, device_id, current_time)
                    
                    return sync_operations
            '''
        }
    
    def file_chunking_strategy(self):
        """Strategy for handling large files"""
        
        return {
            'chunking_benefits': [
                'Parallel upload/download',
                'Resume interrupted transfers',
                'Efficient delta sync',
                'Reduced memory usage'
            ],
            
            'chunking_algorithm': '''
            class FileChunker:
                def __init__(self, chunk_size_mb=4):
                    self.chunk_size = chunk_size_mb * 1024 * 1024
                
                def chunk_file(self, file_path):
                    chunks = []
                    
                    with open(file_path, 'rb') as f:
                        chunk_index = 0
                        
                        while True:
                            chunk_data = f.read(self.chunk_size)
                            if not chunk_data:
                                break
                            
                            chunk_hash = hashlib.sha256(chunk_data).hexdigest()
                            
                            chunks.append({
                                'chunk_index': chunk_index,
                                'chunk_hash': chunk_hash,
                                'chunk_size': len(chunk_data),
                                'chunk_data': chunk_data
                            })
                            
                            chunk_index += 1
                    
                    return chunks
                
                def upload_chunks(self, file_id, chunks):
                    upload_results = []
                    
                    # Upload chunks in parallel
                    with ThreadPoolExecutor(max_workers=10) as executor:
                        futures = []
                        
                        for chunk in chunks:
                            future = executor.submit(
                                self.upload_single_chunk,
                                file_id,
                                chunk
                            )
                            futures.append(future)
                        
                        # Collect results
                        for future in futures:
                            upload_results.append(future.result())
                    
                    return upload_results
            ''',
            
            'deduplication': [
                'Hash-based chunk deduplication',
                'Reference counting for shared chunks',
                'Garbage collection for unreferenced chunks'
            ]
        }
```

## Practice Session Guidelines

### Effective Practice Methodology
```python
class IntermediatePracticeGuidelines:
    def session_structure(self):
        """Structure for intermediate practice sessions"""
        
        return {
            'pre_session': {
                'duration_minutes': 5,
                'activities': [
                    'Review relevant concepts',
                    'Set up timer and drawing tools',
                    'Clear mental state and focus'
                ]
            },
            
            'practice_session': {
                'duration_minutes': 60,
                'phase_breakdown': {
                    'requirements': '10 minutes',
                    'capacity_estimation': '5 minutes',
                    'high_level_design': '15 minutes',
                    'detailed_design': '25 minutes',
                    'optimization_and_scaling': '5 minutes'
                }
            },
            
            'post_session': {
                'duration_minutes': 15,
                'activities': [
                    'Self-evaluation against rubric',
                    'Compare with sample solutions',
                    'Note areas for improvement',
                    'Plan follow-up study topics'
                ]
            }
        }
    
    def improvement_tracking(self):
        """Track improvement across practice sessions"""
        
        return {
            'metrics_to_track': [
                'Time to complete requirements phase',
                'Completeness of architecture diagram',
                'Quality of technology justifications',
                'Depth of scalability discussion',
                'Clarity of communication'
            ],
            
            'improvement_indicators': [
                'Faster requirements gathering',
                'More comprehensive initial design',
                'Better trade-off discussions',
                'More detailed scaling strategies',
                'Cleaner diagram organization'
            ]
        }
```

### Common Intermediate Mistakes
```python
class IntermediateMistakes:
    def __init__(self):
        self.mistakes_and_solutions = {
            'architecture_mistakes': {
                'monolithic_services': {
                    'mistake': 'Creating services that do too many things',
                    'solution': 'Apply single responsibility principle to services',
                    'example': 'Separate user service from tweet service'
                },
                'missing_async_processing': {
                    'mistake': 'Making everything synchronous',
                    'solution': 'Identify operations that can be async',
                    'example': 'Timeline fan-out should be async'
                },
                'ignoring_data_consistency': {
                    'mistake': 'Not considering consistency requirements',
                    'solution': 'Discuss consistency model for each component',
                    'example': 'Eventual consistency OK for social feeds'
                }
            },
            
            'scalability_mistakes': {
                'premature_microservices': {
                    'mistake': 'Starting with microservices for simple systems',
                    'solution': 'Start monolithic, evolve to microservices when needed',
                    'example': 'Don\'t use microservices for simple chat system'
                },
                'ignoring_hot_spots': {
                    'mistake': 'Not considering uneven data distribution',
                    'solution': 'Identify and plan for hot spots',
                    'example': 'Celebrity users in social media systems'
                },
                'inadequate_caching': {
                    'mistake': 'Simple caching strategy for complex access patterns',
                    'solution': 'Design multi-level caching with different TTLs',
                    'example': 'Different cache strategies for different data types'
                }
            }
        }
```

## Exercise Variations and Extensions

### Progressive Complexity
```python
class ExerciseVariations:
    def twitter_variations(self):
        """Twitter design with increasing complexity"""
        
        return {
            'basic_twitter': [
                'Simple tweeting and timeline',
                'Basic follow functionality',
                'Single region deployment'
            ],
            
            'intermediate_twitter': [
                'Add media support',
                'Implement search functionality',
                'Add notifications',
                'Basic scaling with read replicas'
            ],
            
            'advanced_twitter': [
                'Trending topics algorithm',
                'Advanced recommendation system',
                'Real-time analytics',
                'Global deployment with edge caching',
                'Content moderation system'
            ]
        }
    
    def instagram_variations(self):
        """Instagram design with increasing complexity"""
        
        return {
            'basic_instagram': [
                'Photo upload and sharing',
                'Basic feed generation',
                'Simple follow system'
            ],
            
            'intermediate_instagram': [
                'Stories feature (24-hour expiry)',
                'Direct messaging',
                'Hashtag system',
                'Basic recommendation algorithm'
            ],
            
            'advanced_instagram': [
                'Instagram Reels (short videos)',
                'Live streaming feature',
                'Shopping integration',
                'Advanced ML-based feed ranking',
                'Creator monetization tools'
            ]
        }
```

## Exercise Problems

### Week-by-Week Practice Plan
```python
class IntermediatePracticeSchedule:
    def weekly_schedule(self):
        return {
            'week_1_social_media': {
                'day_1': 'Twitter (basic version)',
                'day_2': 'Twitter (with optimizations)', 
                'day_3': 'Instagram (focus on media processing)',
                'day_4': 'Instagram (with stories and messaging)',
                'day_5': 'LinkedIn (professional networking variant)',
                'weekend': 'Review and study weak areas'
            },
            
            'week_2_real_time_systems': {
                'day_1': 'Chat System (WhatsApp-like)',
                'day_2': 'Uber (location-based system)',
                'day_3': 'Real-time Analytics Dashboard',
                'day_4': 'Live Streaming Platform',
                'day_5': 'Multiplayer Game Backend',
                'weekend': 'Deep dive into real-time system patterns'
            },
            
            'week_3_content_systems': {
                'day_1': 'Web Crawler',
                'day_2': 'Search Engine (basic)',
                'day_3': 'File Storage System (Dropbox)',
                'day_4': 'Video Streaming (Netflix-like)',
                'day_5': 'Content Recommendation System',
                'weekend': 'Study content delivery and processing patterns'
            },
            
            'week_4_e_commerce_systems': {
                'day_1': 'E-commerce Platform (Amazon-like)',
                'day_2': 'Payment Processing System',
                'day_3': 'Inventory Management System',
                'day_4': 'Order Fulfillment System',
                'day_5': 'Recommendation Engine for E-commerce',
                'weekend': 'Mock interviews with all learned systems'
            }
        }
```

## Key Takeaways

- **Build on fundamentals**: Intermediate exercises combine multiple basic concepts
- **Focus on real-world constraints**: Consider practical limitations and trade-offs
- **Practice time management**: Intermediate questions require efficient time allocation
- **Develop pattern recognition**: Similar patterns appear across different systems
- **Emphasize trade-offs**: Intermediate level requires deeper trade-off analysis
- **Consider failure scenarios**: Always discuss what happens when things go wrong
- **Practice communication**: Explaining complex systems clearly is crucial

## Next Steps

Once comfortable with intermediate exercises, move to: **03-advanced-exercises.md**