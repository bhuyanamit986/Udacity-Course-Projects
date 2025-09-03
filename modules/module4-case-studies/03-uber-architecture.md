# Uber Architecture Case Study

## System Requirements

### Functional Requirements
- Users can request rides
- Drivers can accept/decline ride requests
- Real-time location tracking
- Route optimization and navigation
- Fare calculation and payment processing
- Rating system for drivers and passengers
- Trip history and receipts
- Surge pricing during high demand

### Non-Functional Requirements
- **Scale**: 100M+ active users, 15M+ trips daily
- **Latency**: Location updates < 1 second, ride matching < 10 seconds
- **Availability**: 99.99% uptime (critical for safety)
- **Global**: 70+ countries with local regulations
- **Real-time**: Live tracking and updates
- **Accuracy**: Precise location and fare calculation

## High-Level Architecture

```
Mobile Apps → API Gateway → Microservices → Databases
     ↓              ↓            ↓            ↓
[Rider App]   [Load Balancer] [Location] [PostgreSQL]
[Driver App]      ↓         [Matching]  [Cassandra]
                [CDN]       [Payment]   [Redis]
                           [Dispatch]   [Kafka]
```

## Core Services Deep Dive

### 1. Location Service
```python
class UberLocationService:
    def __init__(self):
        self.location_storage = LocationStorage()
        self.geohash_index = GeohashIndex()
        self.real_time_tracker = RealTimeTracker()
    
    def update_driver_location(self, driver_id, lat, lng, heading, timestamp):
        """Update driver location with high frequency"""
        
        # Validate location data
        if not self.is_valid_location(lat, lng):
            raise InvalidLocationError("Invalid coordinates")
        
        # Create location update
        location_update = {
            'driver_id': driver_id,
            'latitude': lat,
            'longitude': lng,
            'heading': heading,
            'timestamp': timestamp,
            'geohash': self.calculate_geohash(lat, lng)
        }
        
        # Store in fast storage for real-time queries
        self.real_time_tracker.update_location(driver_id, location_update)
        
        # Update geospatial index
        self.geohash_index.update_driver_location(
            driver_id, 
            location_update['geohash']
        )
        
        # Batch store for historical tracking
        self.location_storage.batch_store(location_update)
        
        # Trigger nearby driver updates for active ride requests
        self.update_nearby_ride_requests(location_update)
    
    def find_nearby_drivers(self, pickup_lat, pickup_lng, radius_km=5):
        """Find available drivers near pickup location"""
        
        # Calculate geohash for pickup location
        pickup_geohash = self.calculate_geohash(pickup_lat, pickup_lng)
        
        # Get nearby geohash cells
        nearby_geohashes = self.get_nearby_geohashes(pickup_geohash, radius_km)
        
        nearby_drivers = []
        
        for geohash in nearby_geohashes:
            # Get drivers in this geohash cell
            drivers_in_cell = self.geohash_index.get_drivers_in_cell(geohash)
            
            for driver in drivers_in_cell:
                # Check if driver is available
                if self.is_driver_available(driver['driver_id']):
                    # Calculate exact distance
                    distance = self.calculate_distance(
                        pickup_lat, pickup_lng,
                        driver['latitude'], driver['longitude']
                    )
                    
                    if distance <= radius_km:
                        nearby_drivers.append({
                            'driver_id': driver['driver_id'],
                            'distance_km': distance,
                            'eta_minutes': distance / 0.5,  # Assume 30 km/h avg speed
                            'rating': driver['rating'],
                            'car_type': driver['car_type']
                        })
        
        # Sort by distance and rating
        nearby_drivers.sort(key=lambda d: (d['distance_km'], -d['rating']))
        
        return nearby_drivers
    
    def calculate_geohash(self, lat, lng, precision=7):
        """Calculate geohash for location indexing"""
        
        # Simplified geohash implementation
        lat_range = [-90.0, 90.0]
        lng_range = [-180.0, 180.0]
        
        geohash = ""
        bits = 0
        bit_count = 0
        even_bit = True
        
        while len(geohash) < precision:
            if even_bit:  # longitude
                mid = (lng_range[0] + lng_range[1]) / 2
                if lng > mid:
                    bits = (bits << 1) + 1
                    lng_range[0] = mid
                else:
                    bits = bits << 1
                    lng_range[1] = mid
            else:  # latitude
                mid = (lat_range[0] + lat_range[1]) / 2
                if lat > mid:
                    bits = (bits << 1) + 1
                    lat_range[0] = mid
                else:
                    bits = bits << 1
                    lat_range[1] = mid
            
            even_bit = not even_bit
            bit_count += 1
            
            if bit_count == 5:
                geohash += self.base32_encode(bits)
                bits = 0
                bit_count = 0
        
        return geohash
```

### 2. Ride Matching Service
```python
class UberRideMatchingService:
    def __init__(self):
        self.location_service = LocationService()
        self.driver_service = DriverService()
        self.pricing_service = PricingService()
        self.dispatch_optimizer = DispatchOptimizer()
    
    def request_ride(self, ride_request):
        """Process ride request and match with driver"""
        
        # Validate ride request
        self.validate_ride_request(ride_request)
        
        # Calculate fare estimate
        fare_estimate = self.pricing_service.calculate_fare(
            ride_request['pickup_location'],
            ride_request['destination'],
            ride_request['ride_type']
        )
        
        # Find available drivers
        nearby_drivers = self.location_service.find_nearby_drivers(
            ride_request['pickup_location']['lat'],
            ride_request['pickup_location']['lng'],
            radius_km=10
        )
        
        if not nearby_drivers:
            return {
                'status': 'no_drivers_available',
                'estimated_wait_time': self.estimate_wait_time(ride_request)
            }
        
        # Optimize driver assignment
        optimal_assignment = self.dispatch_optimizer.optimize_assignment(
            ride_request,
            nearby_drivers
        )
        
        # Send ride request to selected driver
        dispatch_result = self.dispatch_ride_request(
            optimal_assignment['driver_id'],
            ride_request,
            fare_estimate
        )
        
        return dispatch_result
    
    def optimize_driver_assignment(self, ride_request, available_drivers):
        """Optimize driver assignment considering multiple factors"""
        
        assignment_scores = []
        
        for driver in available_drivers:
            score = self.calculate_assignment_score(
                ride_request,
                driver
            )
            
            assignment_scores.append({
                'driver_id': driver['driver_id'],
                'score': score,
                'eta_minutes': driver['eta_minutes'],
                'distance_km': driver['distance_km']
            })
        
        # Sort by score (higher is better)
        assignment_scores.sort(key=lambda x: x['score'], reverse=True)
        
        return assignment_scores[0]  # Best assignment
    
    def calculate_assignment_score(self, ride_request, driver):
        """Calculate assignment score based on multiple factors"""
        
        # Distance factor (closer is better)
        distance_score = max(0, 10 - driver['distance_km'])
        
        # Driver rating factor
        rating_score = driver['rating'] * 2
        
        # ETA factor
        eta_score = max(0, 10 - driver['eta_minutes'] / 2)
        
        # Driver utilization factor (balance driver workload)
        utilization_score = self.calculate_utilization_score(driver['driver_id'])
        
        # Combine scores with weights
        total_score = (
            distance_score * 0.4 +
            rating_score * 0.3 +
            eta_score * 0.2 +
            utilization_score * 0.1
        )
        
        return total_score
```

### 3. Real-time Tracking Service
```python
class UberRealTimeTrackingService:
    def __init__(self):
        self.websocket_manager = WebSocketManager()
        self.location_stream = LocationEventStream()
        self.trip_service = TripService()
    
    def start_trip_tracking(self, trip_id, driver_id, rider_id):
        """Start real-time tracking for active trip"""
        
        # Create tracking session
        tracking_session = {
            'trip_id': trip_id,
            'driver_id': driver_id,
            'rider_id': rider_id,
            'start_time': time.time(),
            'status': 'active'
        }
        
        # Establish WebSocket connections
        self.websocket_manager.create_session(rider_id, f"trip:{trip_id}")
        
        # Subscribe to driver location updates
        self.location_stream.subscribe(
            f"driver_location:{driver_id}",
            lambda location: self.handle_driver_location_update(tracking_session, location)
        )
        
        return tracking_session
    
    def handle_driver_location_update(self, tracking_session, location_update):
        """Handle real-time driver location update during trip"""
        
        trip_id = tracking_session['trip_id']
        rider_id = tracking_session['rider_id']
        
        # Calculate ETA to destination
        current_location = (location_update['latitude'], location_update['longitude'])
        destination = self.trip_service.get_trip_destination(trip_id)
        
        eta_minutes = self.calculate_eta(current_location, destination)
        
        # Prepare real-time update for rider
        rider_update = {
            'trip_id': trip_id,
            'driver_location': current_location,
            'eta_minutes': eta_minutes,
            'timestamp': location_update['timestamp']
        }
        
        # Send to rider via WebSocket
        self.websocket_manager.send_to_user(rider_id, rider_update)
        
        # Store location for trip history
        self.trip_service.add_location_point(trip_id, location_update)
        
        # Check if trip is completed
        if self.is_near_destination(current_location, destination):
            self.handle_trip_completion(tracking_session)
```

## Surge Pricing Algorithm

### 1. Dynamic Pricing Engine
```python
class UberSurgePricingEngine:
    def __init__(self):
        self.demand_predictor = DemandPredictor()
        self.supply_tracker = SupplyTracker()
        self.pricing_model = PricingModel()
        self.market_analyzer = MarketAnalyzer()
    
    def calculate_surge_multiplier(self, location, current_time):
        """Calculate surge pricing multiplier for a location"""
        
        # Get supply and demand data
        supply_data = self.supply_tracker.get_current_supply(location)
        demand_data = self.demand_predictor.get_current_demand(location)
        
        # Calculate supply/demand ratio
        supply_demand_ratio = supply_data['available_drivers'] / max(demand_data['ride_requests'], 1)
        
        # Base surge calculation
        if supply_demand_ratio > 1.5:
            surge_multiplier = 1.0  # No surge
        elif supply_demand_ratio > 1.0:
            surge_multiplier = 1.2  # Light surge
        elif supply_demand_ratio > 0.5:
            surge_multiplier = 1.5  # Moderate surge
        elif supply_demand_ratio > 0.25:
            surge_multiplier = 2.0  # High surge
        else:
            surge_multiplier = 3.0  # Maximum surge
        
        # Adjust for external factors
        surge_multiplier = self.adjust_for_external_factors(
            surge_multiplier,
            location,
            current_time
        )
        
        # Smooth surge changes to avoid dramatic price swings
        previous_surge = self.get_previous_surge(location)
        smoothed_surge = self.smooth_surge_transition(previous_surge, surge_multiplier)
        
        # Apply surge pricing
        self.apply_surge_pricing(location, smoothed_surge)
        
        return smoothed_surge
    
    def adjust_for_external_factors(self, base_surge, location, current_time):
        """Adjust surge based on external factors"""
        
        adjustments = 1.0
        
        # Weather impact
        weather = self.get_weather(location)
        if weather['condition'] in ['rain', 'snow', 'storm']:
            adjustments *= 1.3  # 30% increase for bad weather
        
        # Event impact
        nearby_events = self.get_nearby_events(location, current_time)
        for event in nearby_events:
            if event['type'] == 'concert' and event['attendees'] > 10000:
                adjustments *= 1.5
            elif event['type'] == 'sports' and event['attendees'] > 50000:
                adjustments *= 2.0
        
        # Time-based adjustments
        if self.is_peak_hour(current_time):
            adjustments *= 1.2
        
        return min(base_surge * adjustments, 5.0)  # Cap at 5x surge
    
    def smooth_surge_transition(self, previous_surge, new_surge):
        """Smooth surge transitions to avoid price shock"""
        
        max_change_per_minute = 0.1  # Maximum 0.1x change per minute
        
        if abs(new_surge - previous_surge) > max_change_per_minute:
            if new_surge > previous_surge:
                # Gradually increase surge
                return previous_surge + max_change_per_minute
            else:
                # Gradually decrease surge
                return previous_surge - max_change_per_minute
        
        return new_surge
```

### 2. Driver Dispatch System
```python
class UberDispatchSystem:
    def __init__(self):
        self.location_service = LocationService()
        self.driver_state_manager = DriverStateManager()
        self.optimization_engine = DispatchOptimizationEngine()
        self.notification_service = NotificationService()
    
    def dispatch_ride_request(self, ride_request):
        """Dispatch ride request to optimal driver"""
        
        # Get candidate drivers
        candidate_drivers = self.location_service.find_nearby_drivers(
            ride_request['pickup_lat'],
            ride_request['pickup_lng'],
            radius_km=10
        )
        
        # Filter available drivers
        available_drivers = [
            driver for driver in candidate_drivers
            if self.driver_state_manager.is_available(driver['driver_id'])
        ]
        
        if not available_drivers:
            return self.handle_no_available_drivers(ride_request)
        
        # Optimize dispatch decision
        optimization_result = self.optimization_engine.optimize_dispatch(
            ride_request,
            available_drivers
        )
        
        selected_driver = optimization_result['selected_driver']
        
        # Send ride request to driver
        dispatch_result = self.send_ride_request_to_driver(
            selected_driver['driver_id'],
            ride_request
        )
        
        return dispatch_result
    
    def optimize_global_dispatch(self, all_ride_requests, all_available_drivers):
        """Global optimization of all ride requests and drivers"""
        
        # This is a complex optimization problem (assignment problem)
        # Simplified version using greedy algorithm
        
        assignments = []
        unassigned_requests = all_ride_requests.copy()
        available_drivers = all_available_drivers.copy()
        
        while unassigned_requests and available_drivers:
            best_assignment = None
            best_score = -1
            
            for request in unassigned_requests:
                for driver in available_drivers:
                    score = self.calculate_assignment_score(request, driver)
                    
                    if score > best_score:
                        best_score = score
                        best_assignment = (request, driver)
            
            if best_assignment:
                request, driver = best_assignment
                assignments.append({
                    'request_id': request['request_id'],
                    'driver_id': driver['driver_id'],
                    'score': best_score
                })
                
                unassigned_requests.remove(request)
                available_drivers.remove(driver)
            else:
                break  # No more valid assignments
        
        return {
            'assignments': assignments,
            'unassigned_requests': len(unassigned_requests),
            'total_optimization_score': sum(a['score'] for a in assignments)
        }
```

### 3. Trip Management Service
```python
class UberTripManagementService:
    def __init__(self):
        self.trip_storage = TripStorage()
        self.payment_service = PaymentService()
        self.notification_service = NotificationService()
        self.routing_service = RoutingService()
    
    def start_trip(self, trip_id, driver_id, rider_id):
        """Start trip when driver picks up rider"""
        
        trip = self.trip_storage.get_trip(trip_id)
        
        # Validate trip can be started
        if trip['status'] != 'driver_arrived':
            raise InvalidTripStateError("Trip cannot be started")
        
        # Update trip status
        trip['status'] = 'in_progress'
        trip['start_time'] = time.time()
        trip['start_location'] = self.location_service.get_current_location(driver_id)
        
        # Calculate optimal route
        optimal_route = self.routing_service.calculate_route(
            trip['pickup_location'],
            trip['destination'],
            preferences={
                'optimize_for': 'time',  # vs 'distance' or 'cost'
                'avoid_tolls': trip.get('avoid_tolls', False),
                'avoid_highways': trip.get('avoid_highways', False)
            }
        )
        
        trip['planned_route'] = optimal_route
        trip['estimated_fare'] = self.calculate_estimated_fare(optimal_route, trip)
        
        # Save updated trip
        self.trip_storage.update_trip(trip)
        
        # Notify rider that trip has started
        self.notification_service.send_trip_started_notification(rider_id, trip)
        
        # Start real-time tracking
        self.start_real_time_tracking(trip_id, driver_id, rider_id)
        
        return trip
    
    def complete_trip(self, trip_id, end_location):
        """Complete trip and process payment"""
        
        trip = self.trip_storage.get_trip(trip_id)
        
        # Update trip with completion data
        trip['status'] = 'completed'
        trip['end_time'] = time.time()
        trip['end_location'] = end_location
        trip['actual_distance'] = self.calculate_actual_distance(trip)
        trip['actual_duration'] = trip['end_time'] - trip['start_time']
        
        # Calculate final fare
        final_fare = self.calculate_final_fare(trip)
        trip['final_fare'] = final_fare
        
        # Process payment
        payment_result = self.payment_service.process_trip_payment(
            trip['rider_id'],
            final_fare,
            trip['payment_method']
        )
        
        trip['payment_status'] = payment_result['status']
        
        # Save completed trip
        self.trip_storage.update_trip(trip)
        
        # Send completion notifications
        self.send_trip_completion_notifications(trip)
        
        # Update driver availability
        self.driver_state_manager.mark_driver_available(trip['driver_id'])
        
        return trip
```

## Uber's Geospatial Architecture

### 1. Geospatial Indexing
```python
class UberGeospatialIndex:
    """Efficient geospatial indexing for location-based queries"""
    
    def __init__(self):
        self.geohash_precision = 7  # ~150m precision
        self.driver_index = {}  # geohash -> [driver_ids]
        self.redis_client = Redis()
    
    def index_driver_location(self, driver_id, lat, lng):
        """Index driver location for fast proximity queries"""
        
        geohash = self.calculate_geohash(lat, lng, self.geohash_precision)
        
        # Store in Redis for fast queries
        # Use Redis geospatial commands
        self.redis_client.geoadd(
            'driver_locations',
            lng, lat, driver_id
        )
        
        # Also maintain geohash index
        if geohash not in self.driver_index:
            self.driver_index[geohash] = set()
        
        self.driver_index[geohash].add(driver_id)
    
    def find_drivers_in_radius(self, center_lat, center_lng, radius_km):
        """Find all drivers within radius using Redis geospatial queries"""
        
        # Use Redis GEORADIUS command for efficient proximity search
        nearby_drivers = self.redis_client.georadius(
            'driver_locations',
            center_lng, center_lat,
            radius_km, unit='km',
            withcoord=True, withdist=True
        )
        
        # Format results
        formatted_drivers = []
        for driver_data in nearby_drivers:
            driver_id = driver_data[0]
            distance_km = float(driver_data[1])
            coordinates = driver_data[2]
            
            formatted_drivers.append({
                'driver_id': driver_id,
                'distance_km': distance_km,
                'latitude': coordinates[1],
                'longitude': coordinates[0]
            })
        
        return formatted_drivers
    
    def get_supply_heatmap(self, region_bounds):
        """Generate supply heatmap for operational insights"""
        
        # Divide region into grid
        grid_size = 0.01  # ~1km grid cells
        supply_grid = {}
        
        lat_min, lat_max = region_bounds['lat_range']
        lng_min, lng_max = region_bounds['lng_range']
        
        lat = lat_min
        while lat < lat_max:
            lng = lng_min
            while lng < lng_max:
                # Count drivers in this grid cell
                drivers_in_cell = self.find_drivers_in_radius(
                    lat, lng, radius_km=0.5
                )
                
                supply_grid[f"{lat:.3f},{lng:.3f}"] = {
                    'driver_count': len(drivers_in_cell),
                    'supply_density': len(drivers_in_cell) / 0.25  # drivers per km²
                }
                
                lng += grid_size
            lat += grid_size
        
        return supply_grid
```

### 2. Route Optimization
```python
class UberRouteOptimization:
    def __init__(self):
        self.map_service = MapService()
        self.traffic_service = TrafficService()
        self.route_cache = RouteCache()
    
    def calculate_optimal_route(self, origin, destination, optimization_criteria):
        """Calculate optimal route considering multiple factors"""
        
        # Check cache first
        route_key = self.generate_route_key(origin, destination, optimization_criteria)
        cached_route = self.route_cache.get(route_key)
        
        if cached_route and not self.is_route_stale(cached_route):
            return cached_route
        
        # Get current traffic conditions
        traffic_conditions = self.traffic_service.get_current_conditions(origin, destination)
        
        # Calculate multiple route options
        route_options = self.map_service.calculate_routes(
            origin, destination,
            options={
                'alternatives': 3,
                'avoid_tolls': optimization_criteria.get('avoid_tolls', False),
                'avoid_highways': optimization_criteria.get('avoid_highways', False)
            }
        )
        
        # Score each route
        scored_routes = []
        
        for route in route_options:
            score = self.score_route(route, traffic_conditions, optimization_criteria)
            scored_routes.append({
                'route': route,
                'score': score,
                'estimated_duration': route['duration_seconds'],
                'estimated_distance': route['distance_meters']
            })
        
        # Select best route
        best_route = max(scored_routes, key=lambda r: r['score'])
        
        # Cache result
        self.route_cache.set(route_key, best_route, ttl=300)  # 5 minutes
        
        return best_route
    
    def score_route(self, route, traffic_conditions, criteria):
        """Score route based on optimization criteria"""
        
        # Base scores
        time_score = 100 / max(route['duration_seconds'] / 60, 1)  # Inverse of time in minutes
        distance_score = 100 / max(route['distance_meters'] / 1000, 1)  # Inverse of distance in km
        
        # Traffic adjustment
        traffic_factor = 1.0
        for segment in route['segments']:
            segment_traffic = traffic_conditions.get(segment['road_id'], 'normal')
            if segment_traffic == 'heavy':
                traffic_factor *= 0.7
            elif segment_traffic == 'moderate':
                traffic_factor *= 0.85
        
        time_score *= traffic_factor
        
        # Combine scores based on optimization preference
        if criteria.get('optimize_for') == 'time':
            total_score = time_score * 0.8 + distance_score * 0.2
        elif criteria.get('optimize_for') == 'distance':
            total_score = time_score * 0.3 + distance_score * 0.7
        else:
            total_score = time_score * 0.6 + distance_score * 0.4
        
        return total_score
```

## Data Architecture

### 1. Real-time Data Pipeline
```python
class UberRealTimeDataPipeline:
    def __init__(self):
        self.kafka_clusters = {
            'location_events': KafkaCluster('location-events'),
            'trip_events': KafkaCluster('trip-events'),
            'payment_events': KafkaCluster('payment-events')
        }
        
        self.stream_processors = {
            'location_processor': LocationStreamProcessor(),
            'trip_processor': TripStreamProcessor(),
            'surge_processor': SurgePricingProcessor()
        }
    
    def process_location_event(self, location_event):
        """Process real-time location event"""
        
        # Send to Kafka for downstream processing
        self.kafka_clusters['location_events'].send(
            topic='driver_locations',
            key=location_event['driver_id'],
            value=location_event
        )
        
        # Update real-time indexes
        self.location_service.update_driver_location(
            location_event['driver_id'],
            location_event['latitude'],
            location_event['longitude']
        )
        
        # Trigger supply analysis
        self.stream_processors['surge_processor'].process_location_update(location_event)
    
    def process_trip_event(self, trip_event):
        """Process trip lifecycle events"""
        
        event_type = trip_event['event_type']
        
        # Send to appropriate Kafka topic
        self.kafka_clusters['trip_events'].send(
            topic=f'trip_{event_type}',
            key=trip_event['trip_id'],
            value=trip_event
        )
        
        # Real-time processing based on event type
        if event_type == 'trip_requested':
            self.handle_trip_request_event(trip_event)
        elif event_type == 'trip_started':
            self.handle_trip_start_event(trip_event)
        elif event_type == 'trip_completed':
            self.handle_trip_completion_event(trip_event)
```

### 2. Analytics and Machine Learning
```python
class UberAnalyticsAndML:
    def __init__(self):
        self.data_warehouse = DataWarehouse()
        self.ml_platform = MLPlatform()
        self.feature_store = FeatureStore()
    
    def generate_demand_predictions(self, time_horizon_hours=2):
        """Predict ride demand for capacity planning"""
        
        # Extract features for prediction
        features = self.feature_store.get_demand_features({
            'time_features': ['hour_of_day', 'day_of_week', 'month'],
            'weather_features': ['temperature', 'precipitation', 'wind'],
            'event_features': ['nearby_events', 'event_attendance'],
            'historical_features': ['demand_last_week', 'demand_trend']
        })
        
        # Predict demand by location grid
        predictions = {}
        
        for location_grid in self.get_location_grids():
            location_features = self.get_location_specific_features(location_grid, features)
            
            # Use trained ML model for prediction
            demand_prediction = self.ml_platform.predict(
                model_name='demand_forecasting',
                features=location_features
            )
            
            predictions[location_grid] = {
                'predicted_demand': demand_prediction['demand'],
                'confidence_interval': demand_prediction['confidence'],
                'contributing_factors': demand_prediction['feature_importance']
            }
        
        return predictions
    
    def optimize_driver_positioning(self, current_supply, predicted_demand):
        """Recommend optimal driver positioning"""
        
        recommendations = []
        
        for location_grid, demand_info in predicted_demand.items():
            current_supply_count = current_supply.get(location_grid, 0)
            predicted_demand_count = demand_info['predicted_demand']
            
            supply_demand_ratio = current_supply_count / max(predicted_demand_count, 1)
            
            if supply_demand_ratio < 0.8:  # Undersupplied
                # Recommend drivers move to this area
                nearby_oversupplied_areas = self.find_nearby_oversupplied_areas(location_grid)
                
                for oversupplied_area in nearby_oversupplied_areas:
                    recommendations.append({
                        'action': 'relocate_drivers',
                        'from_area': oversupplied_area,
                        'to_area': location_grid,
                        'driver_count': min(5, predicted_demand_count - current_supply_count),
                        'expected_benefit': self.calculate_relocation_benefit(
                            oversupplied_area, location_grid
                        )
                    })
        
        return recommendations
```

## Uber's Scalability Solutions

### 1. Database Sharding Strategy
```python
class UberDatabaseSharding:
    def __init__(self):
        # Different sharding strategies for different data types
        self.sharding_strategies = {
            'trips': GeographicSharding(),      # Shard by city/region
            'users': HashSharding(),            # Shard by user_id hash
            'drivers': GeographicSharding(),    # Shard by operating city
            'payments': TimeBasedSharding()     # Shard by transaction date
        }
    
    def get_shard_for_trip(self, trip_data):
        """Get appropriate shard for trip data"""
        
        # Determine city from pickup location
        city = self.get_city_from_location(
            trip_data['pickup_lat'],
            trip_data['pickup_lng']
        )
        
        return self.sharding_strategies['trips'].get_shard(city)
    
    def get_shard_for_user(self, user_id):
        """Get shard for user data"""
        
        return self.sharding_strategies['users'].get_shard(user_id)
    
    def handle_cross_shard_queries(self, query_type, parameters):
        """Handle queries that span multiple shards"""
        
        if query_type == 'user_trip_history':
            # User's trips might be across multiple geographic shards
            user_id = parameters['user_id']
            user_cities = self.get_user_active_cities(user_id)
            
            trip_results = []
            
            # Query each relevant shard
            for city in user_cities:
                shard = self.sharding_strategies['trips'].get_shard(city)
                city_trips = shard.query_user_trips(user_id)
                trip_results.extend(city_trips)
            
            # Merge and sort results
            trip_results.sort(key=lambda t: t['created_at'], reverse=True)
            
            return trip_results
```

### 2. Global Distribution Architecture
```python
class UberGlobalArchitecture:
    def __init__(self):
        self.regions = {
            'north_america': {
                'primary_dc': 'us-west-2',
                'backup_dc': 'us-east-1',
                'cities': ['new_york', 'san_francisco', 'chicago', 'toronto']
            },
            'europe': {
                'primary_dc': 'eu-west-1',
                'backup_dc': 'eu-central-1',
                'cities': ['london', 'paris', 'berlin', 'amsterdam']
            },
            'asia_pacific': {
                'primary_dc': 'ap-southeast-1',
                'backup_dc': 'ap-northeast-1',
                'cities': ['singapore', 'tokyo', 'sydney', 'mumbai']
            }
        }
    
    def route_request_to_region(self, user_location):
        """Route user request to appropriate region"""
        
        # Determine user's region
        user_region = self.get_region_from_location(user_location)
        
        # Get region configuration
        region_config = self.regions[user_region]
        
        # Check primary datacenter health
        primary_dc = region_config['primary_dc']
        if self.is_datacenter_healthy(primary_dc):
            return primary_dc
        else:
            # Failover to backup datacenter
            backup_dc = region_config['backup_dc']
            return backup_dc
    
    def handle_cross_region_trips(self, trip_request):
        """Handle trips that cross regional boundaries"""
        
        pickup_region = self.get_region_from_location(trip_request['pickup_location'])
        destination_region = self.get_region_from_location(trip_request['destination'])
        
        if pickup_region != destination_region:
            # Cross-region trip requires special handling
            return {
                'trip_type': 'cross_region',
                'primary_region': pickup_region,
                'secondary_region': destination_region,
                'coordination_required': True,
                'special_pricing': True
            }
        else:
            return {
                'trip_type': 'local',
                'region': pickup_region
            }
```

## Performance Optimizations

### 1. Location Update Optimization
```python
class UberLocationOptimization:
    def __init__(self):
        self.batch_processor = BatchLocationProcessor()
        self.compression_service = LocationCompressionService()
        self.update_scheduler = LocationUpdateScheduler()
    
    def optimize_location_updates(self, driver_id, location_history):
        """Optimize frequency and accuracy of location updates"""
        
        driver_state = self.get_driver_state(driver_id)
        
        if driver_state == 'on_trip':
            # High frequency updates during trip
            update_frequency = 4  # seconds
            accuracy_requirement = 'high'
        elif driver_state == 'available':
            # Moderate frequency when available
            update_frequency = 8  # seconds
            accuracy_requirement = 'medium'
        else:
            # Low frequency when offline
            update_frequency = 30  # seconds
            accuracy_requirement = 'low'
        
        # Compress location updates to reduce bandwidth
        compressed_updates = self.compression_service.compress_location_sequence(
            location_history,
            accuracy_requirement
        )
        
        return {
            'update_frequency_seconds': update_frequency,
            'compressed_updates': compressed_updates,
            'bandwidth_savings': self.calculate_bandwidth_savings(location_history, compressed_updates)
        }
    
    def batch_location_processing(self, location_updates):
        """Process location updates in batches for efficiency"""
        
        # Group updates by geographic region
        regional_batches = defaultdict(list)
        
        for update in location_updates:
            region = self.get_region_from_location(update['latitude'], update['longitude'])
            regional_batches[region].append(update)
        
        # Process each regional batch
        processing_results = {}
        
        for region, batch in regional_batches.items():
            # Process batch for this region
            result = self.batch_processor.process_regional_batch(region, batch)
            processing_results[region] = result
        
        return processing_results
```

## Exercise Problems

1. Design Uber's payment processing system with support for multiple payment methods
2. How would you implement Uber's driver rating and feedback system?
3. Design a system to handle Uber's promotional codes and discounts
4. How would you implement Uber Pool (ride sharing) functionality?

## Key Lessons from Uber

### Technical Lessons
- **Real-time systems**: Require different architecture than batch systems
- **Geospatial indexing**: Critical for location-based services
- **Event-driven architecture**: Enables real-time responsiveness
- **Global distribution**: Must account for local regulations and preferences
- **Optimization algorithms**: Can significantly improve business metrics

### Business Lessons
- **Supply and demand**: Technology can help balance marketplace dynamics
- **Local adaptation**: Global systems need local customization
- **Regulatory compliance**: Must be built into the architecture
- **Data as competitive advantage**: Real-time data enables better decisions

### Operational Lessons
- **Monitoring is critical**: Real-time systems need real-time monitoring
- **Gradual rollouts**: Important for safety-critical systems
- **Disaster recovery**: Must account for regional failures
- **Performance optimization**: Small improvements have large business impact

## Uber's Technology Evolution
```python
uber_evolution = {
    'phase_1_monolith': {
        'year': '2009-2013',
        'architecture': 'PHP monolith',
        'challenges': ['Scaling bottlenecks', 'Development velocity'],
        'scale': '< 1M users'
    },
    
    'phase_2_soa': {
        'year': '2013-2016',
        'architecture': 'Service-oriented architecture',
        'improvements': ['Better scaling', 'Team independence'],
        'scale': '10M+ users'
    },
    
    'phase_3_microservices': {
        'year': '2016-present',
        'architecture': 'Domain-driven microservices',
        'improvements': ['Global scaling', 'Real-time capabilities'],
        'scale': '100M+ users'
    }
}
```

## Next Steps

Move to: **04-whatsapp-architecture.md**