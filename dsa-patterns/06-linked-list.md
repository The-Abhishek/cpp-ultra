# 06 — Linked List

### Reverse Linked List (LC 206)
**Pattern:** Iterative Pointer Reversal
**Key Insight:** Keep track of `prev`, update `curr->next` to `prev`, and shift both pointers forward.
```cpp
ListNode* prev = nullptr;
ListNode* curr = head;
while (curr) {
    ListNode* nextNode = curr->next;
    curr->next = prev;
    prev = curr;
    curr = nextNode;
}
return prev;
```
**Complexity:** Time: O(N) | Space: O(1)

### Merge Two Sorted Lists (LC 21)
**Pattern:** Dummy Head + Two Pointers
**Key Insight:** Use a dummy head to easily build the merged list, pointing `tail` to the smaller node each step.
```cpp
ListNode dummy(0);
ListNode* tail = &dummy;
while (list1 && list2) {
    if (list1->val < list2->val) {
        tail->next = list1;
        list1 = list1->next;
    } else {
        tail->next = list2;
        list2 = list2->next;
    }
    tail = tail->next;
}
tail->next = list1 ? list1 : list2;
return dummy.next;
```
**Complexity:** Time: O(N + M) | Space: O(1)

### Linked List Cycle (LC 141)
**Pattern:** Floyd's Tortoise and Hare
**Key Insight:** Slow pointer moves 1 step, fast moves 2. They will inevitably collide if a cycle exists.
```cpp
ListNode *slow = head, *fast = head;
while (fast && fast->next) {
    slow = slow->next;
    fast = fast->next->next;
    if (slow == fast) return true;
}
return false;
```
**Complexity:** Time: O(N) | Space: O(1)

### Linked List Cycle II (LC 142)
**Pattern:** Floyd's Cycle Entry Finding
**Key Insight:** Upon collision, reset one pointer to head. Move both 1 step; they will meet exactly at the cycle start.
```cpp
ListNode *slow = head, *fast = head;
while (fast && fast->next) {
    slow = slow->next;
    fast = fast->next->next;
    if (slow == fast) {
        ListNode* entry = head;
        // reset to head, find intersection point
        while (entry != slow) {
            entry = entry->next;
            slow = slow->next;
        }
        return entry;
    }
}
return nullptr;
```
**Complexity:** Time: O(N) | Space: O(1)

### Reorder List (LC 143)
**Pattern:** Find Mid + Reverse Half + Merge
**Key Insight:** Find middle to split, reverse the second half, then interleave nodes from both halves.
```cpp
// 1. Find mid (slow pointer)
// 2. Reverse second half (prev pointer)
ListNode* first = head;
ListNode* second = prev; 
while (second) {
    ListNode* tmp1 = first->next;
    ListNode* tmp2 = second->next;
    first->next = second;
    second->next = tmp1;
    first = tmp1;
    second = tmp2;
}
```
**Complexity:** Time: O(N) | Space: O(1)

### Remove Nth Node From End (LC 19)
**Pattern:** Two Pointers with Gap
**Key Insight:** Advance `fast` by `n+1` steps. Moving both until `fast` hits end leaves `slow` right before the target node.
```cpp
ListNode dummy(0);
dummy.next = head;
ListNode *fast = &dummy, *slow = &dummy;
for (int i = 0; i <= n; ++i) 
    fast = fast->next; // space fast and slow by n+1 nodes
while (fast) {
    fast = fast->next;
    slow = slow->next;
}
slow->next = slow->next->next;
return dummy.next;
```
**Complexity:** Time: O(N) | Space: O(1)

### Add Two Numbers (LC 2)
**Pattern:** Dummy Head + Carry Math
**Key Insight:** Iterate through both lists and carry. Use dummy head to build result cleanly and handle trailing carry.
```cpp
ListNode dummy(0);
ListNode* curr = &dummy;
int carry = 0;
while (l1 || l2 || carry) {
    // include carry from previous sum
    int sum = carry + (l1 ? l1->val : 0) + (l2 ? l2->val : 0);
    carry = sum / 10;
    curr->next = new ListNode(sum % 10);
    curr = curr->next;
    if (l1) l1 = l1->next;
    if (l2) l2 = l2->next;
}
return dummy.next;
```
**Complexity:** Time: O(max(N, M)) | Space: O(1) (excluding output)

### Merge K Sorted Lists (LC 23)
**Pattern:** Min-Heap
**Key Insight:** Maintain a min-heap of the heads of all lists. Pop the smallest, append, and push its next node.
```cpp
auto comp = [](ListNode* a, ListNode* b) { return a->val > b->val; };
priority_queue<ListNode*, vector<ListNode*>, decltype(comp)> pq(comp);
for (ListNode* head : lists) 
    if (head) pq.push(head);

ListNode dummy(0);
ListNode* tail = &dummy;
while (!pq.empty()) {
    tail->next = pq.top(); pq.pop();
    tail = tail->next;
    if (tail->next) pq.push(tail->next); // push next node of the popped element
}
return dummy.next;
```
**Complexity:** Time: O(N log K) | Space: O(K)

### Copy List with Random Pointer (LC 138)
**Pattern:** Interleave Trick (O(1) space)
**Key Insight:** Weave copies adjacent to originals (`A->A'->B->B'`). This allows setting random pointers without a hash map.
```cpp
// 1. Interleave: A->A'->B->B'
for (Node* curr = head; curr; curr = curr->next->next) {
    Node* copy = new Node(curr->val);
    copy->next = curr->next;
    curr->next = copy;
}
// 2. Assign randoms
for (Node* curr = head; curr; curr = curr->next->next)
    if (curr->random) curr->next->random = curr->random->next;
// 3. Detach
Node dummy(0); Node* tail = &dummy;
for (Node* curr = head; curr; curr = curr->next) {
    tail->next = curr->next;
    tail = tail->next;
    curr->next = curr->next->next;
}
return dummy.next;
```
**Complexity:** Time: O(N) | Space: O(1)

### LRU Cache (LC 146)
**Pattern:** HashMap + Doubly Linked List
**Key Insight:** Map provides O(1) node lookup. Doubly-linked list allows O(1) removal and insertion at head.
```cpp
struct Node { int key, val; Node *prev, *next; };
unordered_map<int, Node*> cache;
Node *head, *tail;
// Core logic: Move to head on access/insert. Evict tail.prev on overflow.
void remove(Node* node) {
    node->prev->next = node->next;
    node->next->prev = node->prev;
}
void insertFront(Node* node) {
    node->next = head->next;
    node->prev = head;
    head->next->prev = node;
    head->next = node;
}
```
**Complexity:** Time: O(1) get/put | Space: O(K)

### Reverse Nodes in K-Group (LC 25)
**Pattern:** Check length + Reverse K
**Key Insight:** Recursively reverse chunks of size K after verifying enough nodes remain. Link tail of reversed chunk to recursive result.
```cpp
ListNode* curr = head;
for (int i = 0; i < k; ++i) { // Check length
    if (!curr) return head;
    curr = curr->next;
}
ListNode *prev = nullptr, *currNode = head;
for (int i = 0; i < k; ++i) {
    ListNode* nextNode = currNode->next;
    currNode->next = prev;
    prev = currNode;
    currNode = nextNode;
}
head->next = reverseKGroup(currNode, k); // recursive call for next chunks
return prev;
```
**Complexity:** Time: O(N) | Space: O(N/K) implicit call stack

### Sort List (LC 148)
**Pattern:** Merge Sort on Linked List
**Key Insight:** Standard merge sort logic: find mid (slow/fast pointers), recursively sort halves, merge two sorted lists.
```cpp
if (!head || !head->next) return head;
// Find mid
ListNode *slow = head, *fast = head->next;
while (fast && fast->next) {
    slow = slow->next; fast = fast->next->next;
}
ListNode* mid = slow->next;
slow->next = nullptr; // sever list into two halves
// Sort halves and merge (using Merge Two Sorted Lists logic)
return mergeLists(sortList(head), sortList(mid));
```
**Complexity:** Time: O(N log N) | Space: O(log N) stack

### Intersection of Two Linked Lists (LC 160)
**Pattern:** Two Pointers Switching Heads
**Key Insight:** Traverse both lists. When reaching the end, switch to the other list's head. Both pointers traverse `len(A) + len(B)` and meet.
```cpp
ListNode *a = headA, *b = headB;
while (a != b) {
    a = a ? a->next : headB; // switch to B's head when A is exhausted
    b = b ? b->next : headA; // switch to A's head when B is exhausted
}
return a;
```
**Complexity:** Time: O(N + M) | Space: O(1)

### Palindrome Linked List (LC 234)
**Pattern:** Find Mid + Reverse Half + Compare
**Key Insight:** Find mid using slow/fast, reverse the second half, and compare node by node with the first half.
```cpp
// 1. Find mid with fast/slow
// 2. Reverse from slow->next (creates second half list)
ListNode* left = head;
ListNode* right = prev; // Head of reversed second half
while (right) {
    if (left->val != right->val) return false;
    left = left->next;
    right = right->next;
}
return true;
```
**Complexity:** Time: O(N) | Space: O(1)
