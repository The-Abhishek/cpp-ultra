# 08 — Heap / Priority Queue

### Kth Largest Element in Stream (LC 703)
**Pattern:** Min-Heap of size K
**Key Insight:** Maintain a min-heap of size K. The smallest element in this heap (the top) will always be the Kth largest element overall.
```cpp
priority_queue<int, vector<int>, greater<int>> minHeap;
int k; // initialized in constructor
// Add logic
minHeap.push(val);
if (minHeap.size() > k) minHeap.pop(); // Evict smallest to maintain size K
return minHeap.top();
```
**Complexity:** Time: `O(log K)` per add | Space: `O(K)`

### Kth Largest Element in Array (LC 215)
**Pattern:** Min-Heap for top K
**Key Insight:** By popping elements when the heap exceeds size K, we ensure the heap only keeps the top K largest elements seen so far.
```cpp
priority_queue<int, vector<int>, greater<int>> minHeap;
for (int num : nums) {
    minHeap.push(num);
    if (minHeap.size() > k) minHeap.pop(); // Discard smaller elements
}
return minHeap.top();
```
**Complexity:** Time: `O(N log K)` | Space: `O(K)`

### Last Stone Weight (LC 1046)
**Pattern:** Max-Heap simulation
**Key Insight:** Use a max-heap to efficiently extract the two heaviest stones in each step.
```cpp
priority_queue<int> maxHeap(stones.begin(), stones.end());
while (maxHeap.size() > 1) {
    int y = maxHeap.top(); maxHeap.pop(); // Heaviest
    int x = maxHeap.top(); maxHeap.pop(); // Second heaviest
    if (x != y) maxHeap.push(y - x); // Push remaining weight
}
return maxHeap.empty() ? 0 : maxHeap.top();
```
**Complexity:** Time: `O(N log N)` | Space: `O(N)`

### K Closest Points to Origin (LC 973)
**Pattern:** Max-Heap based on distance
**Key Insight:** Max-heap of size K keeps track of the K closest points. When the heap exceeds K, the point with the largest distance (at the top) is popped.
```cpp
priority_queue<pair<int, vector<int>>> maxHeap; // {dist, point}
for (auto& p : points) {
    maxHeap.push({p[0]*p[0] + p[1]*p[1], p}); // No need to take sqrt for comparison
    if (maxHeap.size() > k) maxHeap.pop();
}
```
**Complexity:** Time: `O(N log K)` | Space: `O(K)`

### Task Scheduler (LC 621)
**Pattern:** Max-Heap + Queue (Cooldown)
**Key Insight:** Process the most frequent tasks first. Use a queue to put tasks on cooldown until `time + n`.
```cpp
priority_queue<int> maxHeap; // counts
queue<pair<int, int>> q; // {count, availableTime}
int time = 0;
while (!maxHeap.empty() || !q.empty()) {
    time++;
    if (!maxHeap.empty()) {
        if (maxHeap.top() - 1 > 0) q.push({maxHeap.top() - 1, time + n}); // Put on cooldown
        maxHeap.pop();
    }
    if (!q.empty() && q.front().second == time) {
        maxHeap.push(q.front().first); // Cooldown finished, back to heap
        q.pop();
    }
}
```
**Complexity:** Time: `O(N)` | Space: `O(1)` (max 26 elements)

### Design Twitter (LC 355)
**Pattern:** K-way Merge / Max-Heap
**Key Insight:** Since each user's tweets are ordered, we can do a K-way merge (like merging sorted lists) to fetch the 10 most recent tweets.
```cpp
priority_queue<vector<int>> maxHeap; // {time, tweetId, index}
// Push the most recent tweet from each followee
while (!maxHeap.empty() && res.size() < 10) {
    auto t = maxHeap.top(); maxHeap.pop();
    res.push_back(t[1]);
    // push next tweet from the same user's tweet list if available
}
```
**Complexity:** Time: `O(N log K)` | Space: `O(N)`

### Find Median from Data Stream (LC 295)
**Pattern:** Two Heaps (Max-Heap left, Min-Heap right)
**Key Insight:** A max-heap stores the smaller half and a min-heap stores the larger half. Balancing them gives O(1) median access.
```cpp
priority_queue<int> left; // Max-heap for smaller half
priority_queue<int, vector<int>, greater<int>> right; // Min-heap for larger half
void addNum(int num) {
    left.push(num);
    right.push(left.top()); left.pop(); // Route through left to ensure right gets largest
    if (left.size() < right.size()) {
        left.push(right.top()); right.pop(); // Rebalance so left is always >= right
    }
}
double findMedian() {
    return left.size() > right.size() ? left.top() : (left.top() + right.top()) / 2.0;
}
```
**Complexity:** Time: `O(log N)` add, `O(1)` find | Space: `O(N)`

### Reorganize String (LC 767)
**Pattern:** Max-Heap with Delayed Push
**Key Insight:** Always place the most frequent character next. Temporarily hold the placed character in `prev` so it doesn't get used twice in a row.
```cpp
priority_queue<pair<int, char>> maxHeap; // {count, char}
pair<int, char> prev = {0, '#'};
while (!maxHeap.empty()) {
    auto [cnt, ch] = maxHeap.top(); maxHeap.pop();
    res += ch;
    if (prev.first > 0) maxHeap.push(prev); // Put previous char back into heap
    prev = {cnt - 1, ch}; // Hold current char for next iteration
}
// if prev.first > 0, reorganization impossible
```
**Complexity:** Time: `O(N log 26)` | Space: `O(26)`

### Top K Frequent Words (LC 692)
**Pattern:** Min-Heap with custom comparator (freq, lexical)
**Key Insight:** A min-heap (size K) keeps the largest frequencies. For ties, lexical order must be inverted so we evict the lexicographically larger word.
```cpp
auto comp = [](auto& a, auto& b) {
    // Evict if smaller freq, or if same freq but lexicographically larger
    return a.first > b.first || (a.first == b.first && a.second < b.second);
};
priority_queue<pair<int, string>, vector<pair<int, string>>, decltype(comp)> pq(comp);
for (auto& [w, f] : freq) {
    pq.push({f, w});
    if (pq.size() > k) pq.pop();
}
```
**Complexity:** Time: `O(N log K)` | Space: `O(N)`

### Merge K Sorted Lists (LC 23)
**Pattern:** Min-Heap of ListNodes
**Key Insight:** Insert the head of each list into a min-heap. Pop the smallest, append to the result, and push its `next` node.
```cpp
auto comp = [](ListNode* a, ListNode* b) { return a->val > b->val; };
priority_queue<ListNode*, vector<ListNode*>, decltype(comp)> pq(comp);
for (auto list : lists) if (list) pq.push(list);
while (!pq.empty()) {
    ListNode* top = pq.top(); pq.pop(); // Get current smallest
    tail->next = top;
    tail = tail->next;
    if (top->next) pq.push(top->next); // Push next node from the same list
}
```
**Complexity:** Time: `O(N log K)` | Space: `O(K)`

### Ugly Number II (LC 264)
**Pattern:** Min-Heap with Set (deduplication)
**Key Insight:** Generate ugly numbers in ascending order by multiplying the smallest known ugly number by 2, 3, and 5.
```cpp
priority_queue<long, vector<long>, greater<long>> pq;
unordered_set<long> seen;
pq.push(1); seen.insert(1);
long curr;
for (int i = 0; i < n; i++) {
    curr = pq.top(); pq.pop(); // Get next smallest ugly number
    for (int f : {2, 3, 5}) {
        if (seen.insert(curr * f).second) pq.push(curr * f); // Add new candidates
    }
}
return curr;
```
**Complexity:** Time: `O(N log N)` | Space: `O(N)`

### IPO (LC 502)
**Pattern:** Two Heaps (Min-Heap for capital, Max-Heap for profit)
**Key Insight:** Use a min-heap to unlock affordable projects based on current capital, then use a max-heap to pick the most profitable unlocked project.
```cpp
priority_queue<pair<int, int>, vector<pair<int, int>>, greater<>> minCap;
priority_queue<int> maxProf;
for (int i = 0; i < profits.size(); ++i) minCap.push({capital[i], profits[i]});
while (k--) {
    while (!minCap.empty() && minCap.top().first <= w) { // Unlock affordable projects
        maxProf.push(minCap.top().second);
        minCap.pop();
    }
    if (maxProf.empty()) break; // No projects can be afforded
    w += maxProf.top(); maxProf.pop(); // Invest in max profit project
}
```
**Complexity:** Time: `O(N log N)` | Space: `O(N)`
