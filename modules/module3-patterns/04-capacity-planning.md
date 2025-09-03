# Capacity Planning and Estimation

## What is Capacity Planning?

Capacity planning is the process of determining the infrastructure resources needed to meet current and future demand while maintaining performance requirements.

## Estimation Fundamentals

### Back-of-the-Envelope Calculations

#### Key Numbers to Remember
```python
class SystemEstimationConstants:
    # Latency numbers
    L1_CACHE = 0.5      # nanoseconds
    L2_CACHE = 7        # nanoseconds
    RAM = 100           # nanoseconds
    SSD = 150_000       # nanoseconds (0.15ms)
    HDD = 10_000_000    # nanoseconds (10ms)
    NETWORK_WITHIN_DC = 500_000     # nanoseconds (0.5ms)
    NETWORK_CROSS_REGION = 150_000_000  # nanoseconds (150ms)
    
    # Storage numbers
    BYTE = 1
    KB = 1_024
    MB = 1_024 * KB
    GB = 1_024 * MB
    TB = 1_024 * GB
    PB = 1_024 * TB
    
    # Time numbers
    SECOND = 1
    MINUTE = 60 * SECOND
    HOUR = 60 * MINUTE
    DAY = 24 * HOUR
    MONTH = 30 * DAY
    YEAR = 365 * DAY
    
    # Network bandwidth
    ETHERNET_1GB = 125 * MB  # bytes per second
    ETHERNET_10GB = 1.25 * GB  # bytes per second
```

### Estimation Methodology
```python
class CapacityEstimator:
    def __init__(self):
        self.constants = SystemEstimationConstants()
    
    def estimate_storage_requirements(self, user_base, data_per_user, growth_rate, retention_period):
        """Estimate storage requirements"""
        
        # Current storage
        current_storage = user_base * data_per_user
        
        # Future storage with growth
        years = retention_period / self.constants.YEAR
        future_users = user_base * (1 + growth_rate) ** years
        future_storage = future_users * data_per_user
        
        # Add overhead for replication, indexes, etc.
        overhead_multiplier = 3  # 3x for replicas, indexes, logs
        total_storage = future_storage * overhead_multiplier
        
        return {
            'current_storage_gb': current_storage / self.constants.GB,
            'projected_storage_gb': total_storage / self.constants.GB,
            'daily_growth_gb': (total_storage - current_storage) / (retention_period / self.constants.DAY) / self.constants.GB
        }
    
    def estimate_bandwidth_requirements(self, daily_active_users, avg_session_duration, data_per_request):
        """Estimate bandwidth requirements"""
        
        # Calculate peak traffic (assume 3x average)
        peak_multiplier = 3
        
        # Requests per second during peak
        peak_users = daily_active_users * 0.1  # 10% concurrent during peak
        requests_per_user_per_second = avg_session_duration / 60  # Rough estimate
        peak_rps = peak_users * requests_per_user_per_second * peak_multiplier
        
        # Bandwidth calculation
        bandwidth_bps = peak_rps * data_per_request * 8  # Convert to bits
        bandwidth_mbps = bandwidth_bps / (1024 * 1024)
        
        return {
            'peak_requests_per_second': peak_rps,
            'required_bandwidth_mbps': bandwidth_mbps,
            'recommended_bandwidth_mbps': bandwidth_mbps * 2  # 2x safety margin
        }
    
    def estimate_compute_requirements(self, rps, avg_processing_time_ms, cpu_cores_per_request):
        """Estimate compute requirements"""
        
        # Calculate concurrent requests
        concurrent_requests = rps * (avg_processing_time_ms / 1000)
        
        # CPU requirements
        total_cpu_cores = concurrent_requests * cpu_cores_per_request
        
        # Memory requirements (rough estimate)
        memory_per_request_mb = 50  # Typical web app
        total_memory_gb = (concurrent_requests * memory_per_request_mb) / 1024
        
        # Add safety margin
        safety_margin = 2
        
        return {
            'concurrent_requests': concurrent_requests,
            'required_cpu_cores': total_cpu_cores * safety_margin,
            'required_memory_gb': total_memory_gb * safety_margin,
            'recommended_servers': self.calculate_server_count(total_cpu_cores * safety_margin, total_memory_gb * safety_margin)
        }
```

## Workload Analysis

### 1. Traffic Patterns
```python
class TrafficPatternAnalyzer:
    def __init__(self):
        self.hourly_multipliers = {
            0: 0.3, 1: 0.2, 2: 0.15, 3: 0.1, 4: 0.1, 5: 0.15,
            6: 0.3, 7: 0.5, 8: 0.8, 9: 1.0, 10: 1.1, 11: 1.2,
            12: 1.3, 13: 1.2, 14: 1.1, 15: 1.0, 16: 0.9, 17: 0.8,
            18: 0.7, 19: 0.8, 20: 1.0, 21: 1.1, 22: 0.9, 23: 0.6
        }
        
        self.daily_multipliers = {
            'monday': 1.0, 'tuesday': 1.1, 'wednesday': 1.1,
            'thursday': 1.0, 'friday': 0.9, 'saturday': 0.7, 'sunday': 0.6
        }
    
    def calculate_peak_traffic(self, average_rps):
        """Calculate peak traffic based on patterns"""
        
        # Find peak hour and day
        peak_hour_multiplier = max(self.hourly_multipliers.values())
        peak_day_multiplier = max(self.daily_multipliers.values())
        
        # Calculate peak RPS
        peak_rps = average_rps * peak_hour_multiplier * peak_day_multiplier
        
        # Add seasonal/event spikes
        seasonal_spike_multiplier = 3  # Black Friday, etc.
        absolute_peak_rps = peak_rps * seasonal_spike_multiplier
        
        return {
            'average_rps': average_rps,
            'daily_peak_rps': peak_rps,
            'absolute_peak_rps': absolute_peak_rps,
            'scaling_factor_needed': absolute_peak_rps / average_rps
        }
    
    def estimate_geographic_distribution(self, global_users):
        """Estimate traffic distribution by region"""
        
        # Typical distribution for global service
        distribution = {
            'north_america': 0.35,
            'europe': 0.25,
            'asia_pacific': 0.30,
            'other': 0.10
        }
        
        regional_traffic = {}
        for region, percentage in distribution.items():
            regional_traffic[region] = {
                'users': int(global_users * percentage),
                'percentage': percentage,
                'timezone_offset_hours': self.get_timezone_offset(region)
            }
        
        return regional_traffic
```

### 2. Resource Utilization Analysis
```python
class ResourceUtilizationAnalyzer:
    def __init__(self):
        self.target_utilization = {
            'cpu': 0.70,      # 70% target CPU utilization
            'memory': 0.80,   # 80% target memory utilization
            'disk': 0.75,     # 75% target disk utilization
            'network': 0.60   # 60% target network utilization
        }
    
    def calculate_required_capacity(self, workload_requirements):
        """Calculate required capacity based on target utilization"""
        
        required_capacity = {}
        
        for resource, requirement in workload_requirements.items():
            target_util = self.target_utilization[resource]
            
            # Calculate required capacity to stay below target utilization
            required_capacity[resource] = requirement / target_util
        
        return required_capacity
    
    def analyze_scaling_needs(self, current_capacity, projected_growth):
        """Analyze when scaling will be needed"""
        
        scaling_timeline = {}
        
        for resource, current in current_capacity.items():
            target_util = self.target_utilization[resource]
            max_workload = current * target_util
            
            # Calculate when we'll hit capacity limits
            growth_rate = projected_growth[resource]
            current_workload = projected_growth[f"current_{resource}_usage"]
            
            if growth_rate > 0:
                months_to_capacity = math.log(max_workload / current_workload) / math.log(1 + growth_rate)
                scaling_timeline[resource] = {
                    'months_until_scaling_needed': months_to_capacity,
                    'recommended_scaling_capacity': max_workload * 2  # 2x current capacity
                }
        
        return scaling_timeline
```

## Database Capacity Planning

### 1. Storage Estimation
```python
class DatabaseCapacityPlanner:
    def __init__(self):
        self.storage_calculator = StorageCalculator()
    
    def estimate_table_storage(self, table_schema, estimated_rows):
        """Estimate storage for database table"""
        
        row_size = 0
        
        for column in table_schema:
            column_type = column['type']
            column_size = self.get_column_size(column_type, column.get('length'))
            row_size += column_size
        
        # Add overhead for row headers, indexes, etc.
        overhead_multiplier = 1.5
        effective_row_size = row_size * overhead_multiplier
        
        # Calculate total storage
        total_storage_bytes = estimated_rows * effective_row_size
        
        # Add index storage (estimate 30% of table size)
        index_storage = total_storage_bytes * 0.3
        
        total_with_indexes = total_storage_bytes + index_storage
        
        return {
            'row_size_bytes': row_size,
            'effective_row_size_bytes': effective_row_size,
            'table_storage_gb': total_storage_bytes / (1024**3),
            'index_storage_gb': index_storage / (1024**3),
            'total_storage_gb': total_with_indexes / (1024**3)
        }
    
    def estimate_query_performance(self, table_size_rows, query_type, index_available):
        """Estimate query performance"""
        
        if query_type == 'point_lookup':
            if index_available:
                # B-tree index lookup: O(log n)
                estimated_time_ms = math.log2(table_size_rows) * 0.1
            else:
                # Full table scan: O(n)
                estimated_time_ms = table_size_rows * 0.001
        
        elif query_type == 'range_scan':
            if index_available:
                # Indexed range scan
                estimated_time_ms = math.log2(table_size_rows) * 0.5
            else:
                # Full table scan
                estimated_time_ms = table_size_rows * 0.001
        
        elif query_type == 'aggregation':
            # Aggregation typically requires full scan or index scan
            estimated_time_ms = table_size_rows * 0.0005
        
        return {
            'estimated_time_ms': estimated_time_ms,
            'performance_category': self.categorize_performance(estimated_time_ms)
        }
    
    def categorize_performance(self, time_ms):
        """Categorize query performance"""
        if time_ms < 10:
            return 'excellent'
        elif time_ms < 100:
            return 'good'
        elif time_ms < 1000:
            return 'acceptable'
        else:
            return 'poor'
```

### 2. Connection Pool Sizing
```python
class ConnectionPoolSizer:
    def __init__(self):
        self.default_connection_overhead = 8 * 1024 * 1024  # 8MB per connection
    
    def calculate_optimal_pool_size(self, workload_characteristics):
        """Calculate optimal connection pool size"""
        
        # Extract workload characteristics
        peak_rps = workload_characteristics['peak_rps']
        avg_query_time_ms = workload_characteristics['avg_query_time_ms']
        concurrent_users = workload_characteristics['concurrent_users']
        
        # Calculate required connections using Little's Law
        # Connections = RPS × Average Query Time
        required_connections = peak_rps * (avg_query_time_ms / 1000)
        
        # Add safety margin
        safety_margin = 1.5
        recommended_connections = int(required_connections * safety_margin)
        
        # Consider database limits
        db_max_connections = workload_characteristics.get('db_max_connections', 1000)
        
        # Memory impact
        memory_usage_mb = recommended_connections * (self.default_connection_overhead / (1024 * 1024))
        
        return {
            'calculated_connections': required_connections,
            'recommended_connections': min(recommended_connections, db_max_connections),
            'memory_usage_mb': memory_usage_mb,
            'utilization_percentage': (recommended_connections / db_max_connections) * 100
        }
```

## Cache Capacity Planning

### 1. Cache Size Estimation
```python
class CacheCapacityPlanner:
    def __init__(self):
        self.cache_overhead = 1.2  # 20% overhead for cache metadata
    
    def estimate_cache_size(self, data_characteristics):
        """Estimate required cache size"""
        
        # Extract characteristics
        total_data_items = data_characteristics['total_items']
        avg_item_size_bytes = data_characteristics['avg_item_size_bytes']
        cache_hit_target = data_characteristics['target_hit_ratio']
        access_pattern = data_characteristics['access_pattern']  # 'uniform', 'zipf', 'temporal'
        
        # Calculate cache size based on access pattern
        if access_pattern == 'zipf':
            # 80/20 rule: 20% of data generates 80% of requests
            cache_ratio = 0.2 / cache_hit_target
        elif access_pattern == 'temporal':
            # Recent data is accessed more frequently
            cache_ratio = 0.3 / cache_hit_target
        else:
            # Uniform access pattern
            cache_ratio = cache_hit_target
        
        # Calculate cache size
        cached_items = int(total_data_items * cache_ratio)
        cache_size_bytes = cached_items * avg_item_size_bytes * self.cache_overhead
        
        return {
            'cached_items': cached_items,
            'cache_size_gb': cache_size_bytes / (1024**3),
            'cache_ratio': cache_ratio,
            'estimated_hit_ratio': min(cache_hit_target, 0.95)
        }
    
    def estimate_cache_performance_impact(self, cache_config, workload):
        """Estimate performance impact of caching"""
        
        # Without cache
        avg_db_latency = workload['avg_db_query_time_ms']
        
        # With cache
        cache_hit_ratio = cache_config['estimated_hit_ratio']
        cache_latency = 1  # 1ms cache access
        
        # Weighted average latency
        avg_latency_with_cache = (
            cache_hit_ratio * cache_latency +
            (1 - cache_hit_ratio) * avg_db_latency
        )
        
        # Performance improvement
        improvement_ratio = avg_db_latency / avg_latency_with_cache
        
        return {
            'latency_without_cache_ms': avg_db_latency,
            'latency_with_cache_ms': avg_latency_with_cache,
            'performance_improvement': improvement_ratio,
            'cache_hit_ratio': cache_hit_ratio
        }
```

### 2. Multi-Level Cache Planning
```python
class MultiLevelCachePlanner:
    def __init__(self):
        self.cache_levels = {
            'l1': {'latency_ms': 0.1, 'cost_per_gb': 1000, 'max_size_gb': 1},
            'l2': {'latency_ms': 1, 'cost_per_gb': 100, 'max_size_gb': 100},
            'l3': {'latency_ms': 10, 'cost_per_gb': 10, 'max_size_gb': 1000}
        }
    
    def optimize_cache_hierarchy(self, access_patterns, budget_limit):
        """Optimize cache hierarchy for given access patterns and budget"""
        
        # Sort data by access frequency
        sorted_data = sorted(
            access_patterns.items(),
            key=lambda x: x[1]['access_frequency'],
            reverse=True
        )
        
        cache_allocation = {'l1': [], 'l2': [], 'l3': []}
        total_cost = 0
        
        for data_key, data_info in sorted_data:
            data_size_gb = data_info['size_gb']
            access_freq = data_info['access_frequency']
            
            # Try to allocate to fastest cache level that has space
            allocated = False
            
            for level in ['l1', 'l2', 'l3']:
                level_info = self.cache_levels[level]
                current_usage = sum(item['size_gb'] for item in cache_allocation[level])
                
                if current_usage + data_size_gb <= level_info['max_size_gb']:
                    cost = data_size_gb * level_info['cost_per_gb']
                    
                    if total_cost + cost <= budget_limit:
                        cache_allocation[level].append({
                            'data_key': data_key,
                            'size_gb': data_size_gb,
                            'access_frequency': access_freq
                        })
                        total_cost += cost
                        allocated = True
                        break
            
            if not allocated:
                # Data doesn't fit in any cache level within budget
                break
        
        return {
            'cache_allocation': cache_allocation,
            'total_cost': total_cost,
            'estimated_performance': self.calculate_performance(cache_allocation, access_patterns)
        }
```

## Network Capacity Planning

### 1. Bandwidth Estimation
```python
class NetworkCapacityPlanner:
    def __init__(self):
        self.protocol_overhead = {
            'tcp': 0.05,    # 5% overhead
            'http': 0.10,   # 10% overhead
            'https': 0.12,  # 12% overhead (TLS)
            'grpc': 0.08    # 8% overhead
        }
    
    def estimate_bandwidth_requirements(self, service_communication_matrix):
        """Estimate bandwidth between services"""
        
        bandwidth_matrix = {}
        
        for source_service, targets in service_communication_matrix.items():
            bandwidth_matrix[source_service] = {}
            
            for target_service, communication_info in targets.items():
                # Extract communication characteristics
                rps = communication_info['requests_per_second']
                avg_payload_kb = communication_info['avg_payload_size_kb']
                protocol = communication_info['protocol']
                
                # Calculate bandwidth
                overhead = self.protocol_overhead.get(protocol, 0.10)
                effective_payload = avg_payload_kb * (1 + overhead)
                
                # Bidirectional bandwidth (request + response)
                response_size_kb = communication_info.get('avg_response_size_kb', avg_payload_kb)
                total_transfer_kb = effective_payload + response_size_kb
                
                bandwidth_kbps = rps * total_transfer_kb
                
                bandwidth_matrix[source_service][target_service] = {
                    'bandwidth_kbps': bandwidth_kbps,
                    'bandwidth_mbps': bandwidth_kbps / 1024,
                    'protocol': protocol,
                    'rps': rps
                }
        
        return bandwidth_matrix
    
    def calculate_network_topology_requirements(self, bandwidth_matrix):
        """Calculate network topology requirements"""
        
        # Calculate total bandwidth per service
        service_bandwidth = {}
        
        for source, targets in bandwidth_matrix.items():
            outbound_bandwidth = sum(
                target_info['bandwidth_kbps'] 
                for target_info in targets.values()
            )
            
            # Calculate inbound bandwidth
            inbound_bandwidth = sum(
                bandwidth_matrix[other_source][source]['bandwidth_kbps']
                for other_source in bandwidth_matrix
                if source in bandwidth_matrix[other_source]
            )
            
            service_bandwidth[source] = {
                'outbound_kbps': outbound_bandwidth,
                'inbound_kbps': inbound_bandwidth,
                'total_kbps': outbound_bandwidth + inbound_bandwidth
            }
        
        # Recommend network infrastructure
        recommendations = []
        
        for service, bandwidth in service_bandwidth.items():
            total_mbps = bandwidth['total_kbps'] / 1024
            
            if total_mbps > 1000:  # > 1 Gbps
                recommendations.append({
                    'service': service,
                    'recommendation': '10 Gbps network interface',
                    'current_requirement': f"{total_mbps:.1f} Mbps"
                })
            elif total_mbps > 100:  # > 100 Mbps
                recommendations.append({
                    'service': service,
                    'recommendation': '1 Gbps network interface',
                    'current_requirement': f"{total_mbps:.1f} Mbps"
                })
        
        return {
            'service_bandwidth_requirements': service_bandwidth,
            'infrastructure_recommendations': recommendations
        }
```

## Auto-Scaling Planning

### 1. Horizontal Auto-Scaling
```python
class HorizontalAutoScaler:
    def __init__(self):
        self.scale_up_threshold = 0.70    # 70% CPU
        self.scale_down_threshold = 0.30  # 30% CPU
        self.cooldown_period = 300        # 5 minutes
        self.min_instances = 2
        self.max_instances = 100
    
    def calculate_scaling_decision(self, current_metrics, current_instances):
        """Calculate if scaling is needed"""
        
        avg_cpu_utilization = current_metrics['avg_cpu_utilization']
        avg_memory_utilization = current_metrics['avg_memory_utilization']
        request_queue_length = current_metrics['request_queue_length']
        
        # Determine scaling action
        if (avg_cpu_utilization > self.scale_up_threshold or 
            avg_memory_utilization > 0.80 or
            request_queue_length > 100):
            
            # Scale up
            target_instances = self.calculate_scale_up_instances(
                current_instances, 
                avg_cpu_utilization
            )
            
            return {
                'action': 'scale_up',
                'current_instances': current_instances,
                'target_instances': target_instances,
                'reason': f"CPU: {avg_cpu_utilization:.1%}, Memory: {avg_memory_utilization:.1%}"
            }
        
        elif (avg_cpu_utilization < self.scale_down_threshold and
              avg_memory_utilization < 0.50 and
              request_queue_length < 10):
            
            # Scale down
            target_instances = max(
                self.min_instances,
                current_instances - 1
            )
            
            return {
                'action': 'scale_down',
                'current_instances': current_instances,
                'target_instances': target_instances,
                'reason': f"Low utilization - CPU: {avg_cpu_utilization:.1%}"
            }
        
        else:
            return {
                'action': 'no_change',
                'current_instances': current_instances,
                'target_instances': current_instances,
                'reason': 'Metrics within acceptable range'
            }
    
    def calculate_scale_up_instances(self, current_instances, cpu_utilization):
        """Calculate how many instances to scale up to"""
        
        # Target 50% CPU utilization after scaling
        target_cpu = 0.50
        required_capacity_multiplier = cpu_utilization / target_cpu
        
        target_instances = int(current_instances * required_capacity_multiplier)
        
        # Limit scaling speed (max 50% increase at once)
        max_increase = int(current_instances * 1.5)
        target_instances = min(target_instances, max_increase)
        
        # Respect maximum limit
        target_instances = min(target_instances, self.max_instances)
        
        return target_instances
```

### 2. Predictive Scaling
```python
class PredictiveScaler:
    def __init__(self, historical_data):
        self.historical_data = historical_data
        self.prediction_model = self.train_prediction_model()
    
    def predict_future_load(self, time_horizon_hours):
        """Predict future load based on historical patterns"""
        
        current_time = datetime.now()
        predictions = []
        
        for hour_offset in range(time_horizon_hours):
            future_time = current_time + timedelta(hours=hour_offset)
            
            # Extract time features
            features = {
                'hour_of_day': future_time.hour,
                'day_of_week': future_time.weekday(),
                'day_of_month': future_time.day,
                'month': future_time.month,
                'is_weekend': future_time.weekday() >= 5,
                'is_holiday': self.is_holiday(future_time)
            }
            
            # Predict load
            predicted_load = self.prediction_model.predict(features)
            
            predictions.append({
                'timestamp': future_time,
                'predicted_rps': predicted_load,
                'confidence_interval': self.calculate_confidence_interval(predicted_load)
            })
        
        return predictions
    
    def calculate_proactive_scaling(self, predictions, current_instances):
        """Calculate proactive scaling based on predictions"""
        
        scaling_plan = []
        
        for prediction in predictions:
            predicted_rps = prediction['predicted_rps']
            
            # Calculate required instances
            # Assume each instance can handle 100 RPS at 70% CPU
            instance_capacity = 100
            target_utilization = 0.70
            
            required_instances = math.ceil(
                predicted_rps / (instance_capacity * target_utilization)
            )
            
            scaling_plan.append({
                'timestamp': prediction['timestamp'],
                'required_instances': required_instances,
                'scaling_action': self.determine_scaling_action(
                    current_instances, 
                    required_instances
                )
            })
        
        return scaling_plan
```

## Cost Optimization

### 1. Resource Cost Analysis
```python
class ResourceCostAnalyzer:
    def __init__(self):
        # Example AWS pricing (simplified)
        self.pricing = {
            'compute': {
                't3.medium': {'cpu': 2, 'memory_gb': 4, 'cost_per_hour': 0.0416},
                't3.large': {'cpu': 2, 'memory_gb': 8, 'cost_per_hour': 0.0832},
                'c5.large': {'cpu': 2, 'memory_gb': 4, 'cost_per_hour': 0.085},
                'c5.xlarge': {'cpu': 4, 'memory_gb': 8, 'cost_per_hour': 0.17}
            },
            'storage': {
                'ebs_gp2': {'cost_per_gb_month': 0.10},
                'ebs_gp3': {'cost_per_gb_month': 0.08},
                's3_standard': {'cost_per_gb_month': 0.023}
            },
            'network': {
                'data_transfer_out': {'cost_per_gb': 0.09}
            }
        }
    
    def calculate_infrastructure_cost(self, resource_requirements):
        """Calculate monthly infrastructure cost"""
        
        costs = {}
        
        # Compute costs
        compute_req = resource_requirements['compute']
        instance_type = self.select_optimal_instance_type(
            compute_req['cpu_cores'],
            compute_req['memory_gb']
        )
        
        instance_count = compute_req['instance_count']
        hours_per_month = 24 * 30
        
        costs['compute'] = (
            self.pricing['compute'][instance_type]['cost_per_hour'] *
            instance_count *
            hours_per_month
        )
        
        # Storage costs
        storage_req = resource_requirements['storage']
        costs['storage'] = (
            storage_req['total_gb'] *
            self.pricing['storage']['ebs_gp3']['cost_per_gb_month']
        )
        
        # Network costs
        network_req = resource_requirements['network']
        costs['network'] = (
            network_req['outbound_gb_month'] *
            self.pricing['network']['data_transfer_out']['cost_per_gb']
        )
        
        total_cost = sum(costs.values())
        
        return {
            'breakdown': costs,
            'total_monthly_cost': total_cost,
            'instance_type': instance_type,
            'instance_count': instance_count
        }
    
    def optimize_cost_performance(self, requirements, budget_constraint):
        """Find optimal cost-performance configuration"""
        
        configurations = []
        
        # Try different instance types and counts
        for instance_type, specs in self.pricing['compute'].items():
            for instance_count in range(1, 21):  # Try 1-20 instances
                
                total_cpu = specs['cpu'] * instance_count
                total_memory = specs['memory_gb'] * instance_count
                
                # Check if configuration meets requirements
                if (total_cpu >= requirements['min_cpu_cores'] and
                    total_memory >= requirements['min_memory_gb']):
                    
                    # Calculate cost
                    monthly_cost = (
                        specs['cost_per_hour'] * instance_count * 24 * 30
                    )
                    
                    if monthly_cost <= budget_constraint:
                        # Calculate performance score
                        performance_score = self.calculate_performance_score(
                            total_cpu, total_memory, requirements
                        )
                        
                        configurations.append({
                            'instance_type': instance_type,
                            'instance_count': instance_count,
                            'total_cpu': total_cpu,
                            'total_memory': total_memory,
                            'monthly_cost': monthly_cost,
                            'performance_score': performance_score,
                            'cost_efficiency': performance_score / monthly_cost
                        })
        
        # Sort by cost efficiency
        configurations.sort(key=lambda x: x['cost_efficiency'], reverse=True)
        
        return configurations[:5]  # Top 5 configurations
```

## Capacity Planning for Specific Scenarios

### 1. Social Media Platform
```python
class SocialMediaCapacityPlanner:
    def __init__(self):
        self.user_behavior_model = UserBehaviorModel()
    
    def estimate_timeline_service_capacity(self, user_base, engagement_metrics):
        """Estimate capacity for timeline service"""
        
        # User behavior assumptions
        daily_active_users = user_base * 0.6  # 60% DAU
        posts_per_user_per_day = engagement_metrics['avg_posts_per_user_day']
        timeline_views_per_user_day = engagement_metrics['avg_timeline_views_per_user_day']
        
        # Calculate daily operations
        daily_posts = daily_active_users * posts_per_user_per_day
        daily_timeline_reads = daily_active_users * timeline_views_per_user_day
        
        # Peak traffic (assume 3x average during peak hours)
        peak_posts_per_second = (daily_posts * 3) / (24 * 3600)
        peak_timeline_reads_per_second = (daily_timeline_reads * 3) / (24 * 3600)
        
        # Storage estimation
        avg_post_size = 500  # bytes (text + metadata)
        daily_storage_growth = daily_posts * avg_post_size
        
        # Timeline cache estimation
        avg_timeline_size = 50 * avg_post_size  # 50 posts per timeline
        timeline_cache_size = daily_active_users * avg_timeline_size
        
        return {
            'peak_write_rps': peak_posts_per_second,
            'peak_read_rps': peak_timeline_reads_per_second,
            'daily_storage_growth_gb': daily_storage_growth / (1024**3),
            'timeline_cache_size_gb': timeline_cache_size / (1024**3),
            'recommended_architecture': self.recommend_timeline_architecture(
                peak_posts_per_second, 
                peak_timeline_reads_per_second
            )
        }
```

### 2. Video Streaming Platform
```python
class VideoStreamingCapacityPlanner:
    def __init__(self):
        self.video_bitrates = {
            '240p': 0.4,    # Mbps
            '480p': 1.0,    # Mbps
            '720p': 2.5,    # Mbps
            '1080p': 5.0,   # Mbps
            '4k': 15.0      # Mbps
        }
    
    def estimate_streaming_capacity(self, user_metrics, content_metrics):
        """Estimate capacity for video streaming"""
        
        concurrent_viewers = user_metrics['peak_concurrent_viewers']
        avg_session_duration_minutes = user_metrics['avg_session_duration_minutes']
        quality_distribution = user_metrics['quality_distribution']
        
        # Calculate bandwidth requirements
        total_bandwidth_mbps = 0
        
        for quality, percentage in quality_distribution.items():
            viewers_at_quality = concurrent_viewers * percentage
            bitrate_mbps = self.video_bitrates[quality]
            bandwidth_for_quality = viewers_at_quality * bitrate_mbps
            total_bandwidth_mbps += bandwidth_for_quality
        
        # Convert to GB/month
        seconds_per_month = 30 * 24 * 3600
        gb_per_month = (total_bandwidth_mbps * seconds_per_month) / (8 * 1024)
        
        # Storage requirements
        content_catalog_size_tb = content_metrics['total_content_hours'] * self.estimate_storage_per_hour()
        
        # CDN requirements
        cdn_cache_ratio = 0.8  # 80% cache hit ratio
        origin_bandwidth_mbps = total_bandwidth_mbps * (1 - cdn_cache_ratio)
        
        return {
            'peak_bandwidth_gbps': total_bandwidth_mbps / 1000,
            'monthly_bandwidth_tb': gb_per_month / 1024,
            'content_storage_tb': content_catalog_size_tb,
            'origin_bandwidth_mbps': origin_bandwidth_mbps,
            'cdn_requirements': {
                'cache_storage_tb': content_catalog_size_tb * cdn_cache_ratio,
                'edge_locations_needed': self.calculate_edge_locations(concurrent_viewers)
            }
        }
    
    def estimate_storage_per_hour(self):
        """Estimate storage required per hour of content"""
        
        # Multiple quality versions
        storage_gb_per_hour = (
            self.video_bitrates['240p'] * 3600 / (8 * 1024) +   # 240p
            self.video_bitrates['480p'] * 3600 / (8 * 1024) +   # 480p
            self.video_bitrates['720p'] * 3600 / (8 * 1024) +   # 720p
            self.video_bitrates['1080p'] * 3600 / (8 * 1024) +  # 1080p
            self.video_bitrates['4k'] * 3600 / (8 * 1024)       # 4K
        )
        
        return storage_gb_per_hour
```

## Capacity Monitoring and Alerting

### 1. Capacity Metrics
```python
class CapacityMonitor:
    def __init__(self):
        self.metrics_collector = MetricsCollector()
        self.alerting_system = AlertingSystem()
    
    def monitor_capacity_metrics(self):
        """Monitor key capacity metrics"""
        
        metrics = {
            'cpu_utilization': self.get_avg_cpu_utilization(),
            'memory_utilization': self.get_avg_memory_utilization(),
            'disk_utilization': self.get_avg_disk_utilization(),
            'network_utilization': self.get_avg_network_utilization(),
            'database_connections': self.get_db_connection_usage(),
            'cache_hit_ratio': self.get_cache_hit_ratio(),
            'queue_depth': self.get_avg_queue_depth()
        }
        
        # Check thresholds and alert
        self.check_capacity_thresholds(metrics)
        
        return metrics
    
    def check_capacity_thresholds(self, metrics):
        """Check if any metrics exceed capacity thresholds"""
        
        thresholds = {
            'cpu_utilization': {'warning': 0.70, 'critical': 0.85},
            'memory_utilization': {'warning': 0.75, 'critical': 0.90},
            'disk_utilization': {'warning': 0.80, 'critical': 0.90},
            'database_connections': {'warning': 0.80, 'critical': 0.95}
        }
        
        for metric_name, value in metrics.items():
            if metric_name in thresholds:
                threshold = thresholds[metric_name]
                
                if value >= threshold['critical']:
                    self.alerting_system.send_alert(
                        'CRITICAL',
                        f"{metric_name} is at {value:.1%} (critical threshold: {threshold['critical']:.1%})"
                    )
                elif value >= threshold['warning']:
                    self.alerting_system.send_alert(
                        'WARNING',
                        f"{metric_name} is at {value:.1%} (warning threshold: {threshold['warning']:.1%})"
                    )
    
    def generate_capacity_report(self, time_period_days):
        """Generate capacity utilization report"""
        
        historical_metrics = self.get_historical_metrics(time_period_days)
        
        report = {
            'period': f"Last {time_period_days} days",
            'average_utilization': self.calculate_average_utilization(historical_metrics),
            'peak_utilization': self.calculate_peak_utilization(historical_metrics),
            'capacity_trends': self.analyze_capacity_trends(historical_metrics),
            'scaling_recommendations': self.generate_scaling_recommendations(historical_metrics)
        }
        
        return report
```

## Exercise Problems

1. Estimate the infrastructure requirements for a messaging app with 10M daily active users
2. Plan the capacity for a video streaming service expecting 1M concurrent viewers
3. Design an auto-scaling strategy for an e-commerce platform during Black Friday
4. Calculate the database capacity needed for a social media platform with 100M users

## Capacity Planning Checklist

### Pre-Planning
- [ ] Define performance requirements
- [ ] Identify peak usage patterns
- [ ] Understand data growth patterns
- [ ] Set budget constraints
- [ ] Define SLA requirements

### Estimation Process
- [ ] Estimate traffic patterns
- [ ] Calculate storage requirements
- [ ] Estimate compute requirements
- [ ] Plan network capacity
- [ ] Consider geographic distribution

### Validation
- [ ] Load testing
- [ ] Stress testing
- [ ] Capacity testing
- [ ] Disaster recovery testing

### Monitoring
- [ ] Set up capacity monitoring
- [ ] Configure alerting thresholds
- [ ] Implement auto-scaling
- [ ] Regular capacity reviews

## Key Takeaways

- Start with back-of-the-envelope calculations
- Consider peak traffic, not just average
- Plan for growth and seasonal variations
- Monitor actual usage vs estimates
- Build in safety margins for critical systems
- Consider geographic distribution of users
- Factor in data replication and backup needs
- Regular capacity reviews are essential
- Auto-scaling helps with unpredictable load
- Cost optimization is an ongoing process

## Next Steps

Move to: **05-best-practices.md** and then to **Module 4: Case Studies**