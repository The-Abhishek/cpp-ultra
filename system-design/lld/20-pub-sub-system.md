# Design Pub-Sub Messaging System

## 1. Requirements Clarification
**Functional Requirements:**
- Topics can be created.
- Publishers send messages to topics.
- Subscribers consume messages from subscribed topics.

**Non-Functional Requirements:**
- High availability and scalability.
- Thread-safe publishing and consuming.
- (Optional but good) Persistent offsets for subscribers.

> **Interview Tip**: Distinguish between *Message Queues* (RabbitMQ, point-to-point) and *Pub-Sub/Log Streams* (Kafka, broadcast to multiple consumer groups).

## 2. Actors & Use Cases
- **Publisher**: Generates events.
- **Subscriber**: Listens and processes events.
- **MessageBroker**: The middleman ensuring delivery and decoupling.

## 3. Core Entities & Patterns
- **Topic**: A named channel containing a sequence of messages.
- **Message**: Immutable payload.
- **Broker**: Manages topics and routes messages.
- **Design Patterns:**
  - **Observer Pattern**: *Why?* The essence of Pub-Sub. Topics observe incoming messages and notify subscribers.
  - **Mediator Pattern**: *Why?* The Broker acts as a mediator so Publishers don't need to know about Subscribers, ensuring loose coupling.

## 4. Class Diagram & Architecture

```text
+---------------+      +-------------------+      +----------------+
|  Publisher    |----->|  MessageBroker    |----->|  Subscriber    |
+---------------+      |-------------------|      +----------------+
                       | - topics: map     |
                       | + publish()       |
                       | + subscribe()     |
                       +-------------------+
                                |
                                v
                       +-------------------+
                       |      Topic        |
                       |-------------------|
                       | - messages: queue |
                       | - subs: list      |
                       +-------------------+
```

## 5. Deep Dive: Offset Tracking & Async Delivery
**Sync vs Async Delivery**: 
If the Broker uses a synchronous `for-loop` to call `sub->receive(msg)` for every subscriber, a slow subscriber will block the Publisher!
*Solution*: Use an internal Thread Pool per Topic or push to Subscriber-specific queues (Push model), or let Subscribers pull messages at their own pace tracking their own `offset` (Pull model like Kafka).

**Offset Tracking**: 
Each subscriber maintains an `atomic<int>` offset indicating the index of the last read message in the Topic's message vector.

## 6. Full C++ Implementation (Thread-Safe Pull Model)

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <unordered_map>
#include <memory>
#include <mutex>
#include <shared_mutex>
#include <thread>
#include <atomic>

struct Message {
    std::string payload;
    Message(std::string p) : payload(std::move(p)) {}
};

class Topic {
private:
    std::string name;
    std::vector<std::shared_ptr<Message>> messages;
    mutable std::shared_mutex rw_lock; // Allows multiple concurrent readers

public:
    Topic(std::string n) : name(n) {}

    void addMessage(const std::shared_ptr<Message>& msg) {
        std::unique_lock lock(rw_lock);
        messages.push_back(msg);
    }

    std::shared_ptr<Message> getMessage(int offset) const {
        std::shared_lock lock(rw_lock);
        if (offset < messages.size()) {
            return messages[offset];
        }
        return nullptr;
    }
    
    int getSize() const {
        std::shared_lock lock(rw_lock);
        return messages.size();
    }
};

class Subscriber {
private:
    std::string id;
    std::unordered_map<std::string, int> offsets; // TopicName -> offset

public:
    Subscriber(std::string identifier) : id(identifier) {}

    void poll(const std::string& topicName, std::shared_ptr<Topic> topic) {
        if (offsets.find(topicName) == offsets.end()) {
            offsets[topicName] = 0;
        }

        int currentOffset = offsets[topicName];
        while (auto msg = topic->getMessage(currentOffset)) {
            std::cout << "Subscriber [" << id << "] received: " << msg->payload << "\n";
            currentOffset++;
        }
        offsets[topicName] = currentOffset;
    }
};

class MessageBroker {
private:
    std::unordered_map<std::string, std::shared_ptr<Topic>> topics;
    std::mutex broker_lock;

public:
    void createTopic(const std::string& name) {
        std::lock_guard<std::mutex> lock(broker_lock);
        if (topics.find(name) == topics.end()) {
            topics[name] = std::make_shared<Topic>(name);
        }
    }

    void publish(const std::string& topicName, const std::string& payload) {
        std::shared_ptr<Topic> topic;
        {
            std::lock_guard<std::mutex> lock(broker_lock);
            if (topics.find(topicName) == topics.end()) return;
            topic = topics[topicName];
        }
        topic->addMessage(std::make_shared<Message>(payload));
    }

    std::shared_ptr<Topic> getTopic(const std::string& topicName) {
        std::lock_guard<std::mutex> lock(broker_lock);
        if (topics.find(topicName) != topics.end()) {
            return topics[topicName];
        }
        return nullptr;
    }
};

int main() {
    MessageBroker broker;
    broker.createTopic("updates");

    Subscriber sub1("WorkerA");
    Subscriber sub2("WorkerB");

    broker.publish("updates", "Event 1");
    broker.publish("updates", "Event 2");

    auto topic = broker.getTopic("updates");

    // Both subscribers poll independently and get all messages
    sub1.poll("updates", topic);
    
    broker.publish("updates", "Event 3");
    
    sub2.poll("updates", topic); // Gets 1, 2, 3
    sub1.poll("updates", topic); // Gets only 3

    return 0;
}
```
