# LRU Cache LLD

## 1. Requirements Collection
*   **Use Cases**: 
    *   `GET(key)`: Retrieve value, move item to Most Recently Used (MRU) position.
    *   `PUT(key, value)`: Add/Update item. If capacity is full, evict Least Recently Used (LRU) item.
*   **Constraints**: Both operations must be O(1) time complexity.

## 2. Terminology & Concepts
*   **LRU (Least Recently Used)**: Cache eviction policy that discards the least recently accessed items first.
*   **DLL (Doubly Linked List)**: Linked list where nodes have pointers to both previous and next nodes. Allows O(1) removals if pointer is known.
*   **Hash Map**: Key-Value store offering O(1) average lookup time.
*   **LFU (Least Frequently Used)**: Alternative policy based on access frequency.

## 3. Class Design & ASCII Diagram
```text
      HashMap: Key -> Node*
      
      [Key1] --------> Node1
      [Key2] --------> Node2
                       
    +------+      +-------+      +-------+      +------+
    | Head | <--> | Node1 | <--> | Node2 | <--> | Tail |
    +------+      +-------+      +-------+      +------+
      (MRU)                                       (LRU)
```
*Why this combination?*
*   HashMap gives O(1) lookup.
*   DLL gives O(1) removal (from anywhere, given pointer) and insertion (at head/tail).

## 4. Design Patterns & SOLID Principles
*   **Data Structure Design**: Not a standard GoF pattern, but a composite data structure pattern.
*   **SRP (Single Responsibility)**: `Node` just holds data. `LRUCache` orchestrates map and list.

## 5. Full C++ Implementation (Thread-Safe)
```cpp
#include <iostream>
#include <unordered_map>
#include <shared_mutex>
#include <mutex>

using namespace std;

struct Node {
    int key;
    int value;
    Node* prev;
    Node* next;
    Node(int k, int v) : key(k), value(v), prev(nullptr), next(nullptr) {}
};

class LRUCache {
private:
    int capacity;
    unordered_map<int, Node*> cache;
    Node* head;
    Node* tail;
    mutable shared_mutex rwMutex; // Thread safety

    void removeNode(Node* node) {
        node->prev->next = node->next;
        node->next->prev = node->prev;
    }

    void insertToHead(Node* node) {
        node->next = head->next;
        node->prev = head;
        head->next->prev = node;
        head->next = node;
    }

    void moveToHead(Node* node) {
        removeNode(node);
        insertToHead(node);
    }

public:
    LRUCache(int cap) : capacity(cap) {
        head = new Node(-1, -1);
        tail = new Node(-1, -1);
        head->next = tail;
        tail->prev = head;
    }

    int get(int key) {
        shared_lock<shared_mutex> readLock(rwMutex); // Allow concurrent reads if not moving
        // But since GET modifies state (moveToHead), we actually need a write lock!
        readLock.unlock();
        
        unique_lock<shared_mutex> writeLock(rwMutex);
        if (cache.find(key) != cache.end()) {
            Node* node = cache[key];
            moveToHead(node);
            return node->value;
        }
        return -1;
    }

    void put(int key, int value) {
        unique_lock<shared_mutex> writeLock(rwMutex);
        
        if (cache.find(key) != cache.end()) {
            Node* node = cache[key];
            node->value = value;
            moveToHead(node);
        } else {
            if (cache.size() >= capacity) {
                // Evict LRU (tail->prev)
                Node* lru = tail->prev;
                removeNode(lru);
                cache.erase(lru->key);
                delete lru;
            }
            Node* newNode = new Node(key, value);
            cache[key] = newNode;
            insertToHead(newNode);
        }
    }
    
    ~LRUCache() {
        Node* curr = head;
        while (curr) {
            Node* next = curr->next;
            delete curr;
            curr = next;
        }
    }
};

int main() {
    LRUCache lru(2);
    lru.put(1, 10);
    lru.put(2, 20);
    cout << "Get 1: " << lru.get(1) << "\n"; // Moves 1 to MRU
    lru.put(3, 30); // Evicts 2
    cout << "Get 2: " << lru.get(2) << "\n"; // Returns -1
    return 0;
}
```

## 6. Interview Tips & Follow-up Questions
*   **Q: Why `shared_mutex` if `get` also needs a write lock?**
    *   A: In strict LRU, `get` mutates the DLL, requiring an exclusive lock. If read performance is critical, we might use approximations (like marking nodes dirty and moving them in bulk asynchronously) to allow true shared reads.
*   **Tip**: Draw the pointer swapping logic on the whiteboard (the `removeNode` and `insertToHead` functions). That's where people usually make pointer mistakes.
*   **LFU Extension**: LFU requires O(1) operations too. Usually implemented with two maps: `Key->Node` and `Frequency->DLL of Nodes`.

