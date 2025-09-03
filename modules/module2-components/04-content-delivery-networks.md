# Content Delivery Networks (CDN)

## What is a CDN?

A Content Delivery Network is a geographically distributed group of servers that work together to provide fast delivery of Internet content by caching content closer to users.

## CDN Architecture

### Basic CDN Structure
```
User in NYC → NYC Edge Server (50ms)
User in LA  → LA Edge Server (50ms)
User in London → London Edge Server (50ms)
                    ↓
                Origin Server (500ms without CDN)
```

### CDN Components

#### Edge Servers
- Geographically distributed cache servers
- Store copies of content
- Serve users from nearest location

#### Origin Server
- Original source of content
- Fallback when content not cached
- Handles cache misses

#### Points of Presence (PoPs)
- Physical locations of edge servers
- Strategic placement for optimal coverage

## CDN Benefits

### 1. Reduced Latency
```
Without CDN:
User (London) → Origin Server (California) = 150ms

With CDN:
User (London) → London Edge Server = 20ms
Latency Reduction: 86%
```

### 2. Reduced Origin Load
```python
class CDNLoadReduction:
    def calculate_origin_load_reduction(self, total_requests, cache_hit_ratio):
        """Calculate how much load CDN removes from origin"""
        
        origin_requests_without_cdn = total_requests
        origin_requests_with_cdn = total_requests * (1 - cache_hit_ratio)
        
        load_reduction = (origin_requests_without_cdn - origin_requests_with_cdn) / origin_requests_without_cdn
        
        return {
            'total_requests': total_requests,
            'cache_hit_ratio': cache_hit_ratio,
            'origin_requests_without_cdn': origin_requests_without_cdn,
            'origin_requests_with_cdn': origin_requests_with_cdn,
            'load_reduction_percentage': load_reduction * 100
        }

# Example: 95% cache hit ratio
# Origin load reduced by 95%
```

### 3. Improved Availability
- Multiple edge servers provide redundancy
- If one edge server fails, traffic routes to another
- Origin server protected from traffic spikes

### 4. DDoS Protection
- Distributed architecture absorbs attacks
- Traffic filtering at edge locations
- Rate limiting capabilities

## CDN Caching Strategies

### 1. Static Content Caching
```html
<!-- HTML with CDN URLs -->
<link rel="stylesheet" href="https://cdn.example.com/css/styles.css">
<script src="https://cdn.example.com/js/app.js"></script>
<img src="https://cdn.example.com/images/logo.png" alt="Logo">
```

**Cache Headers**:
```http
Cache-Control: public, max-age=31536000  # 1 year
ETag: "abc123"
Last-Modified: Wed, 21 Oct 2023 07:28:00 GMT
```

### 2. Dynamic Content Caching
```python
class DynamicContentCaching:
    def generate_api_response(self, request):
        # Add caching headers for API responses
        response_data = self.get_data(request)
        
        headers = {
            'Cache-Control': 'public, max-age=300',  # 5 minutes
            'ETag': self.generate_etag(response_data),
            'Vary': 'Accept-Encoding, User-Agent'
        }
        
        return response_data, headers
    
    def generate_etag(self, data):
        """Generate ETag based on content"""
        content_hash = hashlib.md5(json.dumps(data).encode()).hexdigest()
        return f'"{content_hash}"'
```

### 3. Edge Side Includes (ESI)
```html
<!-- Cacheable page template -->
<html>
<head><title>User Dashboard</title></head>
<body>
    <!-- Static content (cached for hours) -->
    <header>Welcome to our site</header>
    
    <!-- Dynamic content (cached for minutes) -->
    <esi:include src="/api/user/notifications" ttl="300"/>
    
    <!-- Personalized content (not cached) -->
    <esi:include src="/api/user/profile" ttl="0"/>
</body>
</html>
```

## CDN Configuration Examples

### CloudFront Configuration
```python
import boto3

class CloudFrontDistribution:
    def __init__(self):
        self.cloudfront = boto3.client('cloudfront')
    
    def create_distribution(self, origin_domain, cache_behaviors):
        distribution_config = {
            'CallerReference': str(uuid.uuid4()),
            'Origins': {
                'Quantity': 1,
                'Items': [{
                    'Id': 'origin1',
                    'DomainName': origin_domain,
                    'CustomOriginConfig': {
                        'HTTPPort': 80,
                        'HTTPSPort': 443,
                        'OriginProtocolPolicy': 'https-only'
                    }
                }]
            },
            'DefaultCacheBehavior': {
                'TargetOriginId': 'origin1',
                'ViewerProtocolPolicy': 'redirect-to-https',
                'TrustedSigners': {'Enabled': False, 'Quantity': 0},
                'ForwardedValues': {
                    'QueryString': False,
                    'Cookies': {'Forward': 'none'}
                },
                'MinTTL': 0,
                'DefaultTTL': 86400,  # 24 hours
                'MaxTTL': 31536000    # 1 year
            },
            'Enabled': True
        }
        
        return self.cloudfront.create_distribution(
            DistributionConfig=distribution_config
        )
```

### Nginx CDN Configuration
```nginx
# nginx.conf for edge server
http {
    proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=cdn_cache:100m 
                     max_size=10g inactive=60m use_temp_path=off;
    
    upstream origin {
        server origin.example.com:80;
        keepalive 32;
    }
    
    server {
        listen 80;
        server_name cdn.example.com;
        
        # Static content caching
        location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
            proxy_pass http://origin;
            proxy_cache cdn_cache;
            proxy_cache_valid 200 1y;
            proxy_cache_valid 404 1m;
            
            # Add cache headers
            add_header X-Cache-Status $upstream_cache_status;
            expires 1y;
        }
        
        # API response caching
        location /api/ {
            proxy_pass http://origin;
            proxy_cache cdn_cache;
            proxy_cache_valid 200 5m;
            proxy_cache_key "$scheme$request_method$host$request_uri$is_args$args";
            
            # Respect origin cache headers
            proxy_cache_revalidate on;
            proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
        }
    }
}
```

## CDN Optimization Techniques

### 1. Cache Warming
```python
class CDNCacheWarming:
    def __init__(self, cdn_endpoints):
        self.cdn_endpoints = cdn_endpoints
    
    def warm_popular_content(self, popular_urls):
        """Proactively cache popular content"""
        
        for endpoint in self.cdn_endpoints:
            for url in popular_urls:
                cdn_url = f"https://{endpoint}{url}"
                
                # Make request to warm cache
                try:
                    response = requests.get(cdn_url, timeout=10)
                    if response.status_code == 200:
                        print(f"Warmed cache for {cdn_url}")
                except Exception as e:
                    print(f"Failed to warm {cdn_url}: {e}")
    
    def warm_user_specific_content(self, user_id):
        """Warm cache for specific user's content"""
        user_urls = [
            f"/api/user/{user_id}/profile",
            f"/api/user/{user_id}/dashboard",
            f"/api/user/{user_id}/recommendations"
        ]
        
        self.warm_popular_content(user_urls)
```

### 2. Smart Purging
```python
class CDNPurging:
    def __init__(self, cdn_client):
        self.cdn_client = cdn_client
    
    def purge_by_tag(self, tag):
        """Purge all content with specific tag"""
        return self.cdn_client.purge_cache(tags=[tag])
    
    def purge_user_content(self, user_id):
        """Purge all content related to a user"""
        patterns = [
            f"/api/user/{user_id}/*",
            f"/images/user/{user_id}/*",
            f"/cache/user/{user_id}/*"
        ]
        
        for pattern in patterns:
            self.cdn_client.purge_cache(url_pattern=pattern)
    
    def selective_purge(self, content_type, last_modified):
        """Purge content based on type and modification time"""
        if content_type == 'user_profile':
            # Purge immediately for profile changes
            self.purge_by_tag('user_profile')
        elif content_type == 'product_catalog':
            # Batch purge for product changes
            self.schedule_purge('product_catalog', delay=300)
```

### 3. Compression and Optimization
```python
class CDNOptimization:
    def __init__(self):
        self.compression_types = {
            'text/html': 'gzip',
            'text/css': 'gzip',
            'application/javascript': 'gzip',
            'application/json': 'gzip',
            'image/svg+xml': 'gzip'
        }
    
    def optimize_content(self, content, content_type):
        """Apply optimizations based on content type"""
        
        if content_type in self.compression_types:
            # Apply compression
            compression = self.compression_types[content_type]
            content = self.compress_content(content, compression)
        
        if content_type.startswith('image/'):
            # Image optimization
            content = self.optimize_image(content)
        
        if content_type == 'text/html':
            # Minify HTML
            content = self.minify_html(content)
        
        return content
    
    def set_optimal_headers(self, content_type):
        """Set optimal caching headers"""
        
        if content_type.startswith('image/') or content_type in ['text/css', 'application/javascript']:
            # Long cache for static assets
            return {
                'Cache-Control': 'public, max-age=31536000, immutable',
                'Content-Encoding': 'gzip'
            }
        elif content_type == 'application/json':
            # Short cache for API responses
            return {
                'Cache-Control': 'public, max-age=300',
                'Content-Encoding': 'gzip'
            }
        else:
            # Default caching
            return {
                'Cache-Control': 'public, max-age=3600'
            }
```

## CDN for Different Content Types

### 1. Static Assets
```python
class StaticAssetCDN:
    def __init__(self, cdn_base_url):
        self.cdn_base_url = cdn_base_url
    
    def get_asset_url(self, asset_path, version=None):
        """Generate versioned CDN URLs for cache busting"""
        if version:
            return f"{self.cdn_base_url}/{asset_path}?v={version}"
        else:
            # Use file hash for versioning
            file_hash = self.get_file_hash(asset_path)
            return f"{self.cdn_base_url}/{asset_path}?v={file_hash}"
    
    def upload_asset(self, local_path, cdn_path):
        """Upload asset to CDN"""
        # Set optimal headers based on file type
        headers = self.get_headers_for_file_type(cdn_path)
        
        return self.cdn_client.upload_file(
            local_path, 
            cdn_path, 
            headers=headers
        )
```

### 2. API Response Caching
```python
class APICaching:
    def __init__(self, cache_ttl_map):
        self.cache_ttl_map = cache_ttl_map
    
    def get_cache_headers(self, endpoint, user_type):
        """Dynamic cache headers based on endpoint and user"""
        
        base_ttl = self.cache_ttl_map.get(endpoint, 300)  # Default 5 minutes
        
        if user_type == 'premium':
            # Longer cache for premium users
            ttl = base_ttl * 2
        else:
            ttl = base_ttl
        
        return {
            'Cache-Control': f'public, max-age={ttl}',
            'Vary': 'User-Type, Accept-Encoding'
        }
    
    def generate_cache_key(self, request):
        """Generate unique cache key for API request"""
        key_parts = [
            request.path,
            request.method,
            sorted(request.query_params.items()),
            request.headers.get('User-Type', 'standard')
        ]
        
        key_string = json.dumps(key_parts, sort_keys=True)
        return hashlib.md5(key_string.encode()).hexdigest()
```

### 3. Video Content Delivery
```python
class VideoCDN:
    def __init__(self, cdn_client):
        self.cdn_client = cdn_client
    
    def upload_video(self, video_file, video_id):
        """Upload video with multiple quality levels"""
        
        qualities = ['240p', '480p', '720p', '1080p']
        video_urls = {}
        
        for quality in qualities:
            # Transcode video to different quality
            transcoded_file = self.transcode_video(video_file, quality)
            
            # Upload to CDN
            cdn_path = f"videos/{video_id}/{quality}.mp4"
            cdn_url = self.cdn_client.upload(transcoded_file, cdn_path)
            
            video_urls[quality] = cdn_url
        
        return video_urls
    
    def generate_adaptive_playlist(self, video_id):
        """Generate HLS playlist for adaptive streaming"""
        
        playlist = "#EXTM3U\n#EXT-X-VERSION:3\n"
        
        qualities = [
            {'resolution': '1920x1080', 'bandwidth': 5000000, 'quality': '1080p'},
            {'resolution': '1280x720', 'bandwidth': 3000000, 'quality': '720p'},
            {'resolution': '854x480', 'bandwidth': 1500000, 'quality': '480p'},
            {'resolution': '426x240', 'bandwidth': 800000, 'quality': '240p'}
        ]
        
        for q in qualities:
            playlist += f"#EXT-X-STREAM-INF:BANDWIDTH={q['bandwidth']},RESOLUTION={q['resolution']}\n"
            playlist += f"https://cdn.example.com/videos/{video_id}/{q['quality']}.m3u8\n"
        
        return playlist
```

## CDN Implementation Strategies

### 1. Push CDN
```python
class PushCDN:
    """Content is pushed to CDN when it's created/updated"""
    
    def __init__(self, cdn_client):
        self.cdn_client = cdn_client
    
    def upload_content(self, content, path, cache_ttl=3600):
        """Upload content to all edge locations"""
        
        # Upload to CDN
        cdn_url = self.cdn_client.upload(content, path)
        
        # Set cache headers
        headers = {
            'Cache-Control': f'public, max-age={cache_ttl}',
            'Content-Type': self.get_content_type(path)
        }
        
        self.cdn_client.set_headers(path, headers)
        return cdn_url
    
    def invalidate_content(self, path):
        """Remove content from all edge locations"""
        return self.cdn_client.purge(path)
```

### 2. Pull CDN
```python
class PullCDN:
    """Content is pulled from origin when first requested"""
    
    def __init__(self, origin_server, cache_rules):
        self.origin_server = origin_server
        self.cache_rules = cache_rules
        self.edge_cache = {}
    
    def handle_request(self, request_path):
        """Handle request at edge server"""
        
        # Check if content is cached
        if request_path in self.edge_cache:
            cached_item = self.edge_cache[request_path]
            
            # Check if cache is still valid
            if time.time() < cached_item['expires_at']:
                return cached_item['content']
            else:
                # Cache expired, remove it
                del self.edge_cache[request_path]
        
        # Fetch from origin
        content = self.fetch_from_origin(request_path)
        
        # Cache the content
        cache_ttl = self.get_cache_ttl(request_path)
        if cache_ttl > 0:
            self.edge_cache[request_path] = {
                'content': content,
                'expires_at': time.time() + cache_ttl,
                'cached_at': time.time()
            }
        
        return content
    
    def get_cache_ttl(self, path):
        """Get cache TTL based on content type and rules"""
        for pattern, ttl in self.cache_rules.items():
            if re.match(pattern, path):
                return ttl
        return 3600  # Default 1 hour
```

## CDN Security

### 1. Signed URLs
```python
import hmac
import hashlib
from urllib.parse import urlencode

class CDNSecurity:
    def __init__(self, secret_key):
        self.secret_key = secret_key
    
    def generate_signed_url(self, base_url, expiration_time, allowed_ip=None):
        """Generate signed URL for secure content access"""
        
        expires = int(time.time() + expiration_time)
        
        # Create signature data
        signature_data = f"{base_url}{expires}"
        if allowed_ip:
            signature_data += allowed_ip
        
        # Generate signature
        signature = hmac.new(
            self.secret_key.encode(),
            signature_data.encode(),
            hashlib.sha256
        ).hexdigest()
        
        # Build signed URL
        params = {'expires': expires, 'signature': signature}
        if allowed_ip:
            params['ip'] = allowed_ip
        
        return f"{base_url}?{urlencode(params)}"
    
    def validate_signed_url(self, url, client_ip=None):
        """Validate signed URL"""
        
        # Parse URL and extract parameters
        parsed_url = urlparse(url)
        params = parse_qs(parsed_url.query)
        
        expires = int(params['expires'][0])
        provided_signature = params['signature'][0]
        
        # Check expiration
        if time.time() > expires:
            return False, "URL expired"
        
        # Check IP if provided
        if 'ip' in params and client_ip != params['ip'][0]:
            return False, "IP mismatch"
        
        # Verify signature
        base_url = f"{parsed_url.scheme}://{parsed_url.netloc}{parsed_url.path}"
        signature_data = f"{base_url}{expires}"
        if 'ip' in params:
            signature_data += params['ip'][0]
        
        expected_signature = hmac.new(
            self.secret_key.encode(),
            signature_data.encode(),
            hashlib.sha256
        ).hexdigest()
        
        if provided_signature != expected_signature:
            return False, "Invalid signature"
        
        return True, "Valid"
```

### 2. Geo-blocking
```python
class GeoBlocking:
    def __init__(self, allowed_countries):
        self.allowed_countries = allowed_countries
    
    def is_request_allowed(self, client_ip):
        """Check if request is from allowed country"""
        
        country = self.get_country_from_ip(client_ip)
        return country in self.allowed_countries
    
    def get_country_from_ip(self, ip):
        """Get country code from IP address"""
        # Use GeoIP database or service
        return geoip_service.get_country(ip)
```

## CDN Performance Optimization

### 1. Cache Hit Ratio Optimization
```python
class CacheHitOptimization:
    def analyze_cache_performance(self, access_logs):
        """Analyze logs to optimize cache performance"""
        
        hit_ratios = {}
        popular_content = {}
        
        for log_entry in access_logs:
            url = log_entry['url']
            cache_status = log_entry['cache_status']  # HIT, MISS, EXPIRED
            
            # Track hit ratios by URL pattern
            url_pattern = self.get_url_pattern(url)
            if url_pattern not in hit_ratios:
                hit_ratios[url_pattern] = {'hits': 0, 'total': 0}
            
            hit_ratios[url_pattern]['total'] += 1
            if cache_status == 'HIT':
                hit_ratios[url_pattern]['hits'] += 1
            
            # Track popular content
            if url not in popular_content:
                popular_content[url] = 0
            popular_content[url] += 1
        
        # Generate recommendations
        recommendations = []
        
        for pattern, stats in hit_ratios.items():
            hit_ratio = stats['hits'] / stats['total']
            
            if hit_ratio < 0.7:  # Low hit ratio
                recommendations.append({
                    'pattern': pattern,
                    'issue': 'Low cache hit ratio',
                    'suggestion': 'Increase cache TTL or implement cache warming'
                })
        
        return {
            'hit_ratios': hit_ratios,
            'popular_content': sorted(popular_content.items(), key=lambda x: x[1], reverse=True)[:50],
            'recommendations': recommendations
        }
```

### 2. Intelligent Routing
```python
class IntelligentRouting:
    def __init__(self):
        self.server_health = {}
        self.server_load = {}
        self.user_location_cache = {}
    
    def route_request(self, user_ip, content_type):
        """Route request to optimal server"""
        
        # Get user's geographic location
        user_location = self.get_user_location(user_ip)
        
        # Find nearby servers
        nearby_servers = self.get_nearby_servers(user_location)
        
        # Filter healthy servers
        healthy_servers = [
            server for server in nearby_servers
            if self.is_server_healthy(server)
        ]
        
        if not healthy_servers:
            # Fallback to any healthy server
            healthy_servers = [
                server for server in self.all_servers
                if self.is_server_healthy(server)
            ]
        
        # Select server with lowest load
        selected_server = min(
            healthy_servers,
            key=lambda s: self.server_load.get(s, 0)
        )
        
        return selected_server
    
    def update_server_metrics(self, server, response_time, error_occurred):
        """Update server health and load metrics"""
        
        if server not in self.server_health:
            self.server_health[server] = {'successes': 0, 'failures': 0}
        
        if error_occurred:
            self.server_health[server]['failures'] += 1
        else:
            self.server_health[server]['successes'] += 1
        
        # Update load (simplified as response time)
        self.server_load[server] = response_time
```

## CDN Monitoring and Analytics

### Key Metrics
```python
class CDNAnalytics:
    def __init__(self):
        self.metrics = {
            'total_requests': 0,
            'cache_hits': 0,
            'cache_misses': 0,
            'bandwidth_saved': 0,
            'response_times': [],
            'error_count': 0,
            'geographic_distribution': {}
        }
    
    def record_request(self, request_data):
        """Record metrics for each request"""
        
        self.metrics['total_requests'] += 1
        
        if request_data['cache_status'] == 'HIT':
            self.metrics['cache_hits'] += 1
            self.metrics['bandwidth_saved'] += request_data['content_size']
        else:
            self.metrics['cache_misses'] += 1
        
        if request_data['status_code'] >= 400:
            self.metrics['error_count'] += 1
        
        self.metrics['response_times'].append(request_data['response_time'])
        
        # Track geographic distribution
        country = request_data['country']
        if country not in self.metrics['geographic_distribution']:
            self.metrics['geographic_distribution'][country] = 0
        self.metrics['geographic_distribution'][country] += 1
    
    def get_analytics_report(self):
        """Generate analytics report"""
        
        total_requests = self.metrics['total_requests']
        if total_requests == 0:
            return {}
        
        cache_hit_ratio = self.metrics['cache_hits'] / total_requests
        error_rate = self.metrics['error_count'] / total_requests
        avg_response_time = sum(self.metrics['response_times']) / len(self.metrics['response_times'])
        
        return {
            'cache_hit_ratio': cache_hit_ratio,
            'error_rate': error_rate,
            'avg_response_time': avg_response_time,
            'bandwidth_saved_gb': self.metrics['bandwidth_saved'] / (1024**3),
            'top_countries': sorted(
                self.metrics['geographic_distribution'].items(),
                key=lambda x: x[1],
                reverse=True
            )[:10]
        }
```

## CDN Cost Optimization

### 1. Smart Caching Policies
```python
class CDNCostOptimization:
    def __init__(self, cost_per_gb, cost_per_request):
        self.cost_per_gb = cost_per_gb
        self.cost_per_request = cost_per_request
    
    def optimize_cache_policy(self, content_analytics):
        """Optimize caching policy based on cost-benefit analysis"""
        
        recommendations = []
        
        for content_type, stats in content_analytics.items():
            requests_per_day = stats['daily_requests']
            avg_size_mb = stats['avg_size_mb']
            current_hit_ratio = stats['hit_ratio']
            
            # Calculate costs
            current_origin_requests = requests_per_day * (1 - current_hit_ratio)
            current_bandwidth_cost = current_origin_requests * avg_size_mb * self.cost_per_gb / 1024
            
            # Simulate improved hit ratio
            improved_hit_ratio = min(current_hit_ratio + 0.2, 0.95)
            improved_origin_requests = requests_per_day * (1 - improved_hit_ratio)
            improved_bandwidth_cost = improved_origin_requests * avg_size_mb * self.cost_per_gb / 1024
            
            potential_savings = current_bandwidth_cost - improved_bandwidth_cost
            
            if potential_savings > 100:  # $100/day savings threshold
                recommendations.append({
                    'content_type': content_type,
                    'current_hit_ratio': current_hit_ratio,
                    'recommended_hit_ratio': improved_hit_ratio,
                    'potential_daily_savings': potential_savings,
                    'action': 'Increase cache TTL or implement cache warming'
                })
        
        return recommendations
```

## Exercise Problems

1. Design a CDN strategy for a global video streaming platform
2. How would you implement cache invalidation for a news website?
3. Design a CDN architecture for a social media platform with user-generated content
4. Compare the trade-offs between push and pull CDN strategies

## Real-World CDN Examples

### Netflix CDN Strategy
- **Open Connect**: Netflix's own CDN
- **ISP partnerships**: Servers inside ISP networks
- **Content pre-positioning**: Popular content cached before demand

### Cloudflare's Approach
- **Global network**: 200+ data centers
- **Anycast routing**: Single IP, multiple servers
- **Edge computing**: Serverless functions at edge

### Amazon CloudFront
- **Integration**: Seamless AWS integration
- **Lambda@Edge**: Custom logic at edge
- **Real-time metrics**: Detailed analytics

## Key Takeaways

- CDNs dramatically improve user experience through reduced latency
- Choose between push and pull strategies based on content characteristics
- Implement proper cache invalidation strategies
- Monitor cache hit ratios and optimize accordingly
- Consider security requirements for sensitive content
- Geographic distribution is crucial for global applications
- CDNs provide additional benefits beyond caching (DDoS protection, etc.)
- Cost optimization requires balancing cache efficiency with storage costs

## Next Steps

Move to: **05-microservices-architecture.md**