# Message Queues and Event Streaming

## What are Message Queues?

Message queues are communication mechanisms that allow applications to send and receive messages asynchronously, enabling loose coupling between system components.

## Core Concepts

### Producer-Consumer Pattern
```
Producer → Message Queue → Consumer
```

### Message Components
- **Payload**: The actual data being sent
- **Headers**: Metadata about the message
- **Routing Key**: Determines message destination
- **Timestamp**: When message was created

## Message Queue Patterns

### 1. Point-to-Point (Queue)
```python
# One producer, one consumer
class PointToPointQueue:
    def __init__(self):
        self.queue = queue.Queue()
    
    def send_message(self, message):
        self.queue.put(message)
    
    def receive_message(self):
        return self.queue.get(block=True, timeout=30)
```

**Characteristics**:
- Each message consumed by exactly one consumer
- Messages are removed after consumption
- Good for work distribution

### 2. Publish-Subscribe (Topic)
```python
class PubSubTopic:
    def __init__(self):
        self.subscribers = []
        self.messages = []
    
    def subscribe(self, subscriber):
        self.subscribers.append(subscriber)
    
    def publish(self, message):
        self.messages.append(message)
        # Notify all subscribers
        for subscriber in self.subscribers:
            subscriber.notify(message)
```

**Characteristics**:
- Each message delivered to all subscribers
- Messages can be retained for multiple consumers
- Good for event broadcasting

### 3. Request-Reply
```python
class RequestReplyPattern:
    def __init__(self, request_queue, reply_queue):
        self.request_queue = request_queue
        self.reply_queue = reply_queue
        self.correlation_ids = {}
    
    def send_request(self, message):
        correlation_id = str(uuid.uuid4())
        request = {
            'id': correlation_id,
            'payload': message,
            'reply_to': self.reply_queue.name
        }
        
        self.request_queue.send(request)
        return correlation_id
    
    def get_reply(self, correlation_id, timeout=30):
        start_time = time.time()
        
        while time.time() - start_time < timeout:
            reply = self.reply_queue.receive(timeout=1)
            if reply and reply.get('correlation_id') == correlation_id:
                return reply['payload']
        
        raise TimeoutError("No reply received")
```

## Message Queue Technologies

### 1. RabbitMQ
```python
import pika

class RabbitMQProducer:
    def __init__(self, host='localhost'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()
    
    def send_to_queue(self, queue_name, message):
        self.channel.queue_declare(queue=queue_name, durable=True)
        self.channel.basic_publish(
            exchange='',
            routing_key=queue_name,
            body=json.dumps(message),
            properties=pika.BasicProperties(
                delivery_mode=2,  # Make message persistent
            )
        )
    
    def send_to_exchange(self, exchange_name, routing_key, message):
        self.channel.exchange_declare(
            exchange=exchange_name, 
            exchange_type='topic'
        )
        self.channel.basic_publish(
            exchange=exchange_name,
            routing_key=routing_key,
            body=json.dumps(message)
        )

class RabbitMQConsumer:
    def __init__(self, host='localhost'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()
    
    def consume_queue(self, queue_name, callback):
        self.channel.queue_declare(queue=queue_name, durable=True)
        self.channel.basic_qos(prefetch_count=1)
        self.channel.basic_consume(
            queue=queue_name,
            on_message_callback=callback,
            auto_ack=False
        )
        
        print(f"Waiting for messages from {queue_name}...")
        self.channel.start_consuming()
    
    def process_message(self, ch, method, properties, body):
        try:
            message = json.loads(body)
            # Process the message
            self.handle_message(message)
            
            # Acknowledge successful processing
            ch.basic_ack(delivery_tag=method.delivery_tag)
        except Exception as e:
            print(f"Error processing message: {e}")
            # Reject and requeue the message
            ch.basic_nack(
                delivery_tag=method.delivery_tag,
                requeue=True
            )
```

### 2. Apache Kafka
```python
from kafka import KafkaProducer, KafkaConsumer

class KafkaProducerWrapper:
    def __init__(self, bootstrap_servers):
        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda x: json.dumps(x).encode('utf-8'),
            key_serializer=lambda x: x.encode('utf-8') if x else None
        )
    
    def send_message(self, topic, message, key=None):
        future = self.producer.send(topic, value=message, key=key)
        # Block until message is sent
        record_metadata = future.get(timeout=10)
        return record_metadata

class KafkaConsumerWrapper:
    def __init__(self, topics, bootstrap_servers, group_id):
        self.consumer = KafkaConsumer(
            *topics,
            bootstrap_servers=bootstrap_servers,
            group_id=group_id,
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            auto_offset_reset='earliest'
        )
    
    def consume_messages(self, message_handler):
        for message in self.consumer:
            try:
                message_handler(message.value)
                # Commit offset after successful processing
                self.consumer.commit()
            except Exception as e:
                print(f"Error processing message: {e}")
                # Could implement retry logic here
```

### 3. Amazon SQS
```python
import boto3

class SQSQueue:
    def __init__(self, queue_url, region='us-east-1'):
        self.sqs = boto3.client('sqs', region_name=region)
        self.queue_url = queue_url
    
    def send_message(self, message, delay_seconds=0):
        response = self.sqs.send_message(
            QueueUrl=self.queue_url,
            MessageBody=json.dumps(message),
            DelaySeconds=delay_seconds
        )
        return response['MessageId']
    
    def receive_messages(self, max_messages=10):
        response = self.sqs.receive_message(
            QueueUrl=self.queue_url,
            MaxNumberOfMessages=max_messages,
            WaitTimeSeconds=20  # Long polling
        )
        
        return response.get('Messages', [])
    
    def delete_message(self, receipt_handle):
        self.sqs.delete_message(
            QueueUrl=self.queue_url,
            ReceiptHandle=receipt_handle
        )
```

## Event Streaming Platforms

### Apache Kafka Deep Dive

#### Kafka Architecture
```
Topic: user_events
├── Partition 0: [msg1, msg3, msg5]
├── Partition 1: [msg2, msg6, msg8]
└── Partition 2: [msg4, msg7, msg9]
```

#### Producer Configuration
```python
class KafkaProducerOptimized:
    def __init__(self, bootstrap_servers):
        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            
            # Performance settings
            batch_size=16384,  # Batch size in bytes
            linger_ms=10,      # Wait time for batching
            compression_type='snappy',
            
            # Reliability settings
            acks='all',        # Wait for all replicas
            retries=3,
            retry_backoff_ms=100,
            
            # Serialization
            value_serializer=lambda x: json.dumps(x).encode('utf-8'),
            key_serializer=lambda x: x.encode('utf-8') if x else None
        )
    
    def send_user_event(self, user_id, event_data):
        # Use user_id as key for partition consistency
        return self.producer.send(
            'user_events',
            value=event_data,
            key=str(user_id)
        )
```

#### Consumer Groups
```python
class KafkaConsumerGroup:
    def __init__(self, topics, group_id, bootstrap_servers):
        self.consumer = KafkaConsumer(
            *topics,
            bootstrap_servers=bootstrap_servers,
            group_id=group_id,
            
            # Offset management
            auto_offset_reset='earliest',
            enable_auto_commit=False,  # Manual commit for reliability
            
            # Performance settings
            fetch_min_bytes=1024,
            fetch_max_wait_ms=500,
            
            # Deserialization
            value_deserializer=lambda m: json.loads(m.decode('utf-8'))
        )
    
    def process_messages(self):
        for message in self.consumer:
            try:
                # Process the message
                self.handle_message(message.value)
                
                # Commit offset after successful processing
                self.consumer.commit()
                
            except Exception as e:
                print(f"Error processing message: {e}")
                # Could implement dead letter queue here
```

## Message Delivery Guarantees

### 1. At-Most-Once
- Messages may be lost but never duplicated
- **Use case**: Metrics, logs where some loss is acceptable

### 2. At-Least-Once
- Messages are never lost but may be duplicated
- **Use case**: Most business applications with idempotent processing

### 3. Exactly-Once
- Messages are delivered exactly once
- **Use case**: Financial transactions, critical business processes

```python
class ExactlyOnceProcessor:
    def __init__(self, message_queue, database):
        self.queue = message_queue
        self.database = database
        self.processed_messages = set()
    
    def process_message(self, message):
        message_id = message['id']
        
        # Check if already processed (idempotency)
        if message_id in self.processed_messages:
            print(f"Message {message_id} already processed, skipping")
            return
        
        # Process message in transaction
        with self.database.transaction():
            # Business logic
            self.handle_business_logic(message)
            
            # Mark as processed
            self.database.insert_processed_message(message_id)
            self.processed_messages.add(message_id)
            
            # Acknowledge message
            self.queue.ack_message(message)
```

## Queue Design Patterns

### 1. Work Queue Pattern
```python
class WorkQueue:
    """Distribute time-consuming tasks among workers"""
    
    def __init__(self, queue_name):
        self.queue_name = queue_name
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()
        self.channel.queue_declare(queue=queue_name, durable=True)
    
    def add_task(self, task_data):
        message = json.dumps(task_data)
        self.channel.basic_publish(
            exchange='',
            routing_key=self.queue_name,
            body=message,
            properties=pika.BasicProperties(delivery_mode=2)  # Persistent
        )
    
    def start_worker(self, worker_function):
        def callback(ch, method, properties, body):
            task_data = json.loads(body)
            try:
                worker_function(task_data)
                ch.basic_ack(delivery_tag=method.delivery_tag)
            except Exception as e:
                print(f"Task failed: {e}")
                ch.basic_nack(
                    delivery_tag=method.delivery_tag,
                    requeue=True
                )
        
        self.channel.basic_qos(prefetch_count=1)
        self.channel.basic_consume(
            queue=self.queue_name,
            on_message_callback=callback
        )
        
        self.channel.start_consuming()
```

### 2. Priority Queue
```python
class PriorityQueue:
    def __init__(self, queue_name):
        self.queue_name = queue_name
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()
        
        # Declare queue with priority support
        self.channel.queue_declare(
            queue=queue_name,
            durable=True,
            arguments={'x-max-priority': 10}
        )
    
    def send_message(self, message, priority=0):
        self.channel.basic_publish(
            exchange='',
            routing_key=self.queue_name,
            body=json.dumps(message),
            properties=pika.BasicProperties(
                priority=priority,
                delivery_mode=2
            )
        )
```

### 3. Dead Letter Queue
```python
class DeadLetterQueue:
    def __init__(self, main_queue, dlq_name):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()
        
        # Declare dead letter queue
        self.channel.queue_declare(queue=dlq_name, durable=True)
        
        # Declare main queue with DLQ configuration
        self.channel.queue_declare(
            queue=main_queue,
            durable=True,
            arguments={
                'x-dead-letter-exchange': '',
                'x-dead-letter-routing-key': dlq_name,
                'x-message-ttl': 60000,  # 1 minute TTL
                'x-max-retries': 3
            }
        )
```

### 4. Delayed Message Queue
```python
class DelayedMessageQueue:
    def __init__(self, redis_client):
        self.redis = redis_client
    
    def schedule_message(self, message, delay_seconds):
        execute_at = time.time() + delay_seconds
        self.redis.zadd(
            'delayed_messages',
            {json.dumps(message): execute_at}
        )
    
    def process_delayed_messages(self):
        """Background process to handle delayed messages"""
        while True:
            current_time = time.time()
            
            # Get messages ready to be processed
            messages = self.redis.zrangebyscore(
                'delayed_messages',
                0, current_time,
                withscores=True
            )
            
            for message_json, score in messages:
                message = json.loads(message_json)
                
                # Process the message
                self.handle_message(message)
                
                # Remove from delayed queue
                self.redis.zrem('delayed_messages', message_json)
            
            time.sleep(1)  # Check every second
```

## Event Streaming

### Apache Kafka for Event Streaming
```python
class EventStreamer:
    def __init__(self, bootstrap_servers):
        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda x: json.dumps(x).encode('utf-8')
        )
        
        self.consumer = KafkaConsumer(
            bootstrap_servers=bootstrap_servers,
            value_deserializer=lambda m: json.loads(m.decode('utf-8'))
        )
    
    def publish_event(self, event_type, event_data):
        event = {
            'event_type': event_type,
            'timestamp': time.time(),
            'data': event_data,
            'event_id': str(uuid.uuid4())
        }
        
        self.producer.send(f'events_{event_type}', value=event)
    
    def subscribe_to_events(self, event_types, handler):
        topics = [f'events_{event_type}' for event_type in event_types]
        self.consumer.subscribe(topics)
        
        for message in self.consumer:
            handler(message.value)
```

### Event Sourcing Pattern
```python
class EventStore:
    def __init__(self, kafka_producer):
        self.producer = kafka_producer
        self.event_handlers = {}
    
    def append_event(self, aggregate_id, event_type, event_data):
        event = {
            'aggregate_id': aggregate_id,
            'event_type': event_type,
            'event_data': event_data,
            'timestamp': time.time(),
            'version': self.get_next_version(aggregate_id)
        }
        
        # Store event
        self.producer.send(
            f'events_{aggregate_id}',
            value=event,
            key=aggregate_id
        )
        
        return event
    
    def replay_events(self, aggregate_id):
        """Rebuild aggregate state from events"""
        events = self.get_events_for_aggregate(aggregate_id)
        
        state = {}
        for event in events:
            state = self.apply_event(state, event)
        
        return state
    
    def apply_event(self, state, event):
        handler = self.event_handlers.get(event['event_type'])
        if handler:
            return handler(state, event['event_data'])
        return state

# Example usage
event_store = EventStore(kafka_producer)

# Register event handlers
event_store.event_handlers['user_created'] = lambda state, data: {
    **state, 'name': data['name'], 'email': data['email']
}
event_store.event_handlers['email_updated'] = lambda state, data: {
    **state, 'email': data['new_email']
}
```

## Message Queue Use Cases

### 1. Asynchronous Processing
```python
class OrderProcessingSystem:
    def __init__(self, queue):
        self.queue = queue
    
    def place_order(self, order_data):
        # Immediate response to user
        order_id = self.create_order_record(order_data)
        
        # Queue background tasks
        self.queue.send_message('process_payment', {
            'order_id': order_id,
            'payment_info': order_data['payment']
        })
        
        self.queue.send_message('update_inventory', {
            'order_id': order_id,
            'items': order_data['items']
        })
        
        self.queue.send_message('send_confirmation', {
            'order_id': order_id,
            'email': order_data['user_email']
        })
        
        return {'order_id': order_id, 'status': 'processing'}
```

### 2. Microservices Communication
```python
class MicroserviceEventBus:
    def __init__(self, kafka_producer):
        self.producer = kafka_producer
    
    def publish_domain_event(self, service_name, event_type, data):
        event = {
            'service': service_name,
            'event_type': event_type,
            'data': data,
            'timestamp': time.time(),
            'version': '1.0'
        }
        
        # Publish to service-specific topic
        self.producer.send(f'{service_name}_events', value=event)
        
        # Publish to global events topic
        self.producer.send('global_events', value=event)

# Service A publishes event
event_bus.publish_domain_event('user_service', 'user_registered', {
    'user_id': '12345',
    'email': 'user@example.com'
})

# Service B subscribes to user events
def handle_user_registered(event):
    user_id = event['data']['user_id']
    email = event['data']['email']
    
    # Create user profile in this service
    profile_service.create_profile(user_id, email)
```

### 3. Load Leveling
```python
class LoadLevelingQueue:
    def __init__(self, queue, max_workers=5):
        self.queue = queue
        self.max_workers = max_workers
        self.active_workers = 0
        self.worker_pool = []
    
    def handle_traffic_spike(self):
        """Automatically scale workers based on queue depth"""
        queue_depth = self.queue.get_message_count()
        
        if queue_depth > 100 and self.active_workers < self.max_workers:
            # Scale up workers
            self.add_worker()
        elif queue_depth < 10 and self.active_workers > 1:
            # Scale down workers
            self.remove_worker()
    
    def add_worker(self):
        worker = threading.Thread(target=self.worker_loop)
        worker.start()
        self.worker_pool.append(worker)
        self.active_workers += 1
    
    def worker_loop(self):
        while True:
            message = self.queue.receive_message(timeout=30)
            if message:
                self.process_message(message)
            else:
                # No messages, worker can exit
                break
        
        self.active_workers -= 1
```

## Message Queue Monitoring

### Key Metrics
```python
class QueueMetrics:
    def __init__(self, queue_name):
        self.queue_name = queue_name
        self.message_count = 0
        self.processing_times = []
        self.error_count = 0
    
    def record_message_processed(self, processing_time):
        self.message_count += 1
        self.processing_times.append(processing_time)
    
    def record_error(self):
        self.error_count += 1
    
    def get_metrics(self):
        return {
            'queue_name': self.queue_name,
            'messages_processed': self.message_count,
            'avg_processing_time': sum(self.processing_times) / len(self.processing_times) if self.processing_times else 0,
            'error_rate': self.error_count / self.message_count if self.message_count > 0 else 0,
            'queue_depth': self.get_queue_depth()
        }
```

### Alerting
```python
def monitor_queue_health(queue_metrics):
    metrics = queue_metrics.get_metrics()
    
    # Alert on high queue depth
    if metrics['queue_depth'] > 1000:
        alert('HIGH_QUEUE_DEPTH', f"Queue depth: {metrics['queue_depth']}")
    
    # Alert on high error rate
    if metrics['error_rate'] > 0.05:
        alert('HIGH_ERROR_RATE', f"Error rate: {metrics['error_rate']:.2%}")
    
    # Alert on slow processing
    if metrics['avg_processing_time'] > 5000:  # 5 seconds
        alert('SLOW_PROCESSING', f"Avg time: {metrics['avg_processing_time']}ms")
```

## Error Handling and Reliability

### Retry Mechanisms
```python
class RetryableMessageProcessor:
    def __init__(self, max_retries=3, backoff_multiplier=2):
        self.max_retries = max_retries
        self.backoff_multiplier = backoff_multiplier
    
    def process_with_retry(self, message, processor_func):
        for attempt in range(self.max_retries + 1):
            try:
                return processor_func(message)
            except Exception as e:
                if attempt == self.max_retries:
                    # Send to dead letter queue
                    self.send_to_dlq(message, str(e))
                    raise e
                
                # Exponential backoff
                delay = (self.backoff_multiplier ** attempt)
                time.sleep(delay)
                print(f"Retrying message processing, attempt {attempt + 1}")
```

### Circuit Breaker for Message Processing
```python
class MessageProcessorCircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
    
    def process_message(self, message, processor_func):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "HALF_OPEN"
            else:
                # Circuit is open, reject message
                raise Exception("Circuit breaker is OPEN")
        
        try:
            result = processor_func(message)
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise e
    
    def on_success(self):
        self.failure_count = 0
        self.state = "CLOSED"
    
    def on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = "OPEN"
```

## Exercise Problems

1. Design a message queue system for a social media platform's notification service
2. How would you implement exactly-once message delivery?
3. Design an event-driven architecture for an e-commerce order processing system
4. Compare RabbitMQ vs Kafka for different use cases

## Key Takeaways

- Message queues enable asynchronous, loosely coupled systems
- Choose the right delivery guarantee based on business requirements
- Plan for error handling and retry mechanisms
- Monitor queue depth and processing times
- Consider message ordering requirements
- Event streaming is powerful for real-time systems
- Dead letter queues are essential for error handling
- Circuit breakers prevent cascade failures

## Next Steps

Move to: **04-content-delivery-networks.md**