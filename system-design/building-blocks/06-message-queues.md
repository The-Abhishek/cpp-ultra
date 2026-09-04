# 06. Message Queues

Message queues facilitate asynchronous communication between decoupled services. They provide a buffer to absorb traffic spikes and ensure reliable message delivery.

## Patterns

### 1. Point-to-Point (Queue)
- **Concept:** One sender, one receiver. Once a message is consumed by a worker, it is removed from the queue.
- **Use Case:** Task distribution, order processing.
- **Example:** AWS SQS, RabbitMQ.

### 2. Publish-Subscribe (Pub/Sub)
- **Concept:** One sender (Publisher), multiple receivers (Subscribers). A message is broadcasted to all subscribers of a specific topic.
- **Use Case:** Event notifications, logging systems.
- **Example:** Apache Kafka, AWS SNS.

## Delivery Guarantees
- **At-most-once:** Messages might be lost but are never duplicated. (Fastest, least reliable)
- **At-least-once:** Messages are never lost but might be delivered multiple times. Consumers must be **idempotent** (safe to process duplicates).
- **Exactly-once:** Messages are delivered exactly once. (Hardest to implement, high performance cost).

## Ordering Guarantees
Standard queues do not guarantee strict global ordering due to concurrent processing.
- **FIFO Queues:** Guarantee order but severely limit throughput (e.g., SQS FIFO).
- **Partition-based Ordering (Kafka approach):** Messages with the same key (e.g., `user_id`) go to the same partition. Order is guaranteed *within a partition*, allowing high throughput.

## Core Concepts
- **Backpressure:** When consumers are too slow, the queue acts as a buffer. If the queue fills up, the system must either drop messages or reject producer requests (backpressure).
- **Dead Letter Queue (DLQ):** A secondary queue where messages that fail to be processed after multiple retries are sent. Used for debugging and manual intervention.

## Comparison: Kafka vs RabbitMQ vs SQS

| Feature | Apache Kafka | RabbitMQ | AWS SQS |
|---------|--------------|----------|---------|
| Paradigm | Distributed Log (Pub/Sub) | Smart Broker / Smart Routing | Simple Queue |
| Retention | Retains messages (persistent) | Deletes after ack | Deletes after ack |
| Scaling | Partition-based | Cluster-based | Fully Managed / Infinite |
| Best For | High throughput, Event Sourcing | Complex routing, AMQP | Simple, cloud-native decoupled tasks |

> **Interview Tip:** In HLD interviews, use queues for async tasks like sending emails, video processing, or decoupling heavy write loads from the primary database.
