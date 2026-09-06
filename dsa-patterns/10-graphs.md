# 10 — Graphs (BFS, DFS, Topological Sort, Union-Find, Dijkstra)

### Number of Islands (LC 200)
**Pattern:** BFS/DFS Flood Fill
**Key Insight:** Each unvisited '1' starts a new island. DFS/BFS sinks the entire connected component to '0' to avoid revisiting.
```cpp
void dfs(int r, int c, vector<vector<char>>& grid) {
    if (r < 0 || c < 0 || r >= R || c >= C || grid[r][c] == '0') return;
    grid[r][c] = '0'; // mark visited to prevent cycles and duplicate island counts
    dfs(r+1, c, grid); dfs(r-1, c, grid);
    dfs(r, c+1, grid); dfs(r, c-1, grid);
}
```
**Complexity:** Time: `O(M * N)` | Space: `O(M * N)`

### Clone Graph (LC 133)
**Pattern:** DFS + Hash Map for state
**Key Insight:** A hash map tracks original-to-clone mappings to prevent infinite loops in cycles and ensure one clone per node.
```cpp
unordered_map<Node*, Node*> copies;
Node* cloneGraph(Node* node) {
    if (!node) return NULL;
    if (copies.count(node)) return copies[node]; // return already cloned node to handle cycles
    Node* copy = new Node(node->val);
    copies[node] = copy;
    for (Node* neighbor : node->neighbors) {
        copy->neighbors.push_back(cloneGraph(neighbor));
    }
    return copy;
}
```
**Complexity:** Time: `O(V + E)` | Space: `O(V)`

### Pacific Atlantic Water Flow (LC 417)
**Pattern:** Reverse DFS from borders
**Key Insight:** Instead of checking each cell, flow water backwards from the oceans uphill; cells reachable by both oceans are the answer.
```cpp
void dfs(int r, int c, vector<vector<int>>& h, vector<vector<bool>>& ocean) {
    ocean[r][c] = true;
    int dirs[4][2] = {{0,1}, {1,0}, {0,-1}, {-1,0}};
    for (auto d : dirs) {
        int nr = r + d[0], nc = c + d[1];
        if (nr >= 0 && nc >= 0 && nr < R && nc < C && !ocean[nr][nc] && h[nr][nc] >= h[r][c]) { // flowing uphill from the ocean
            dfs(nr, nc, h, ocean);
        }
    }
}
```
**Complexity:** Time: `O(M * N)` | Space: `O(M * N)`

### Course Schedule (LC 207)
**Pattern:** Topological Sort (Kahn's BFS)
**Key Insight:** Treat courses as nodes and prerequisites as directed edges. A valid ordering exists if we can topologically sort without finding cycles.
```cpp
vector<int> inDegree(numCourses, 0);
vector<vector<int>> adj(numCourses);
for (auto& edge : prerequisites) {
    adj[edge[1]].push_back(edge[0]);
    inDegree[edge[0]]++;
}
queue<int> q;
for (int i = 0; i < numCourses; i++) if (inDegree[i] == 0) q.push(i);
int count = 0;
while (!q.empty()) {
    int u = q.front(); q.pop();
    count++;
    for (int v : adj[u]) {
        if (--inDegree[v] == 0) q.push(v); // a course is ready when all its prerequisites are completed
    }
}
return count == numCourses;
```
**Complexity:** Time: `O(V + E)` | Space: `O(V + E)`

### Course Schedule II (LC 210)
**Pattern:** Topological Sort (order construction)
**Key Insight:** Building on cycle detection, Kahn's algorithm processes nodes in topological order naturally.
*(Same as Course Schedule, just push `u` to a result vector instead of merely counting, return vector if size == numCourses)*
**Complexity:** Time: `O(V + E)` | Space: `O(V + E)`

### Graph Valid Tree (LC 261)
**Pattern:** Union-Find (cycle detection & component count)
**Key Insight:** A tree is a graph with exactly n-1 edges and no cycles. Union-Find detects cycles and counts connected components.
```cpp
vector<int> parent(n);
iota(parent.begin(), parent.end(), 0);
int components = n;
function<int(int)> find = [&](int i) { return parent[i] == i ? i : parent[i] = find(parent[i]); };
for (auto& e : edges) {
    int p1 = find(e[0]), p2 = find(e[1]);
    if (p1 == p2) return false; // Cycle detected: both nodes already in the same component
    parent[p1] = p2;
    components--;
}
return components == 1; // Must be fully connected
```
**Complexity:** Time: `O(V + E \alpha(V))` | Space: `O(V)`

### Number of Connected Components (LC 323)
**Pattern:** Union-Find or DFS/BFS traversal
**Key Insight:** Start with N components and decrement for every valid union.
*(Same Union-Find template as Graph Valid Tree, just return `components`)*
**Complexity:** Time: `O(V + E \alpha(V))` | Space: `O(V)`

### Redundant Connection (LC 684)
**Pattern:** Union-Find
**Key Insight:** In a tree, adding an edge creates exactly one cycle. The first edge causing a cycle in Union-Find is the redundant one.
*(First edge that introduces a cycle i.e. `find(e[0]) == find(e[1])` is the redundant one)*
**Complexity:** Time: `O(N \alpha(N))` | Space: `O(N)`

### Word Ladder (LC 127)
**Pattern:** BFS for shortest path
**Key Insight:** Treat words as nodes and 1-letter differences as edges. BFS guarantees the shortest transformation sequence.
```cpp
queue<pair<string, int>> q;
q.push({beginWord, 1});
while (!q.empty()) {
    auto [word, steps] = q.front(); q.pop();
    if (word == endWord) return steps;
    for (int i = 0; i < word.size(); i++) {
        char orig = word[i];
        for (char c = 'a'; c <= 'z'; c++) {
            word[i] = c;
            if (wordList.count(word)) {
                q.push({word, steps + 1});
                wordList.erase(word); // avoid cycles by removing words once visited
            }
        }
        word[i] = orig;
    }
}
```
**Complexity:** Time: `O(M^2 * N)` | Space: `O(M * N)`

### Rotting Oranges (LC 994)
**Pattern:** Multi-source BFS
**Key Insight:** Push all rotten oranges into a queue initially. BFS propagates rot layer by layer, simulating time.
```cpp
queue<pair<int, int>> q;
int fresh = 0, time = 0;
// initialize q with all rotting, count fresh (queue represents 0th minute)
while (!q.empty() && fresh > 0) {
    int sz = q.size();
    while (sz--) {
        auto [r, c] = q.front(); q.pop();
        // check 4-directions. if fresh:
        // grid[nr][nc] = 2, q.push({nr, nc}), fresh--
    }
    time++;
}
return fresh == 0 ? time : -1;
```
**Complexity:** Time: `O(M * N)` | Space: `O(M * N)`

### Walls and Gates (LC 286)
**Pattern:** Multi-source BFS from gates
**Key Insight:** Pushing all gates into the queue first allows a parallel BFS, ensuring each empty room is reached by its nearest gate.
*(Identical to Rotting Oranges, start BFS from all gates concurrently updating shortest distances)*
**Complexity:** Time: `O(M * N)` | Space: `O(M * N)`

### Surrounded Regions (LC 130)
**Pattern:** Reverse DFS from borders
**Key Insight:** Any 'O' connected to the border cannot be captured. Mark them first, then capture all remaining 'O's.
*(Any 'O' connected to a border is safe. DFS all border 'O's and mark as 'S'. Then flip remaining 'O's to 'X', and 'S's back to 'O'.)*
**Complexity:** Time: `O(M * N)` | Space: `O(M * N)`

### Cheapest Flights Within K Stops (LC 787)
**Pattern:** Bellman-Ford / BFS
**Key Insight:** Standard Dijkstra ignores the K-stop limit. Bellman-Ford run exactly K+1 times naturally limits the path length.
```cpp
vector<int> dist(n, INT_MAX);
dist[src] = 0;
for (int i = 0; i <= k; i++) {
    vector<int> temp = dist; // use temp to ensure paths don't exceed current K iteration
    for (auto& f : flights) {
        if (dist[f[0]] != INT_MAX) {
            temp[f[1]] = min(temp[f[1]], dist[f[0]] + f[2]);
        }
    }
    dist = temp;
}
return dist[dst] == INT_MAX ? -1 : dist[dst];
```
**Complexity:** Time: `O(K * E)` | Space: `O(V)`

### Network Delay Time (LC 743)
**Pattern:** Dijkstra's Algorithm
**Key Insight:** Dijkstra finds the shortest path to all nodes. The max of these shortest paths is the time for the signal to reach everyone.
```cpp
priority_queue<pair<int, int>, vector<pair<int, int>>, greater<>> pq;
vector<int> dist(n + 1, INT_MAX);
pq.push({0, k}); dist[k] = 0;

while (!pq.empty()) {
    auto [d, u] = pq.top(); pq.pop();
    if (d > dist[u]) continue; // optimization: ignore outdated longer paths in PQ
    for (auto& [v, w] : adj[u]) {
        if (dist[u] + w < dist[v]) {
            dist[v] = dist[u] + w;
            pq.push({dist[v], v});
        }
    }
}
```
**Complexity:** Time: `O(E log V)` | Space: `O(V + E)`

### Swim in Rising Water (LC 778)
**Pattern:** Dijkstra / Modified BFS (Min-Max path)
**Key Insight:** We want a path where the maximum edge weight is minimized. Dijkstra with a priority queue tracking the max height so far finds this optimally.
```cpp
priority_queue<vector<int>, vector<vector<int>>, greater<>> pq; // {max_height_so_far, r, c} tracks path with lowest peak
pq.push({grid[0][0], 0, 0});
grid[0][0] = -1; // visited
while (!pq.empty()) {
    auto t = pq.top(); pq.pop();
    if (t[1] == N-1 && t[2] == N-1) return t[0];
    // for each valid neighbor:
    // pq.push({max(t[0], grid[nr][nc]), nr, nc})
}
```
**Complexity:** Time: `O(N^2 log N)` | Space: `O(N^2)`

### Alien Dictionary (LC 269)
**Pattern:** Topological Sort on Lexicographical rules
**Key Insight:** Lexicographical order implies directed edges between mismatching characters. A topological sort gives the alphabet order.
*(Compare adjacent words character by character to find directed edges `w1[i] -> w2[i]`. Run Kahn's BFS or DFS Post-Order. Return "" if cycle exists.)*
**Complexity:** Time: `O(Total Chars)` | Space: `O(Total Unique Chars)`

### Accounts Merge (LC 721)
**Pattern:** Union-Find on emails
**Key Insight:** Emails act as nodes; belonging to the same account acts as an edge. Union-Find groups connected emails under one root.
*(Map emails to owner ID. Union emails in the same account. Gather emails by UF root, sort, prepend name.)*
**Complexity:** Time: `O(N log N)` | Space: `O(N)`

### Minimum Spanning Tree (Kruskal's Template)
**Pattern:** Sort Edges + Union-Find
**Key Insight:** Greedily pick the smallest edges. Use Union-Find to avoid cycles until N-1 edges are picked.
```cpp
sort(edges.begin(), edges.end(), [](auto& a, auto& b) { return a[2] < b[2]; }); // {u, v, w}
int mstCost = 0, edgesUsed = 0;
for (auto& e : edges) {
    if (find(e[0]) != find(e[1])) { // only add edge if it doesn't form a cycle
        unite(e[0], e[1]);
        mstCost += e[2];
        if (++edgesUsed == n - 1) break;
    }
}
```
**Complexity:** Time: `O(E log E)` | Space: `O(V)`

### Detect Cycle in Directed/Undirected Graph
**Pattern:** DFS / Union-Find
**Key Insight:** Undirected uses Union-Find; Directed requires 3-state DFS to distinguish back-edges (cycles) from cross-edges.
- **Undirected**: Union-Find (cycle if components already connected).
- **Directed**: DFS states `0=unvisited`, `1=visiting`, `2=visited`. If neighbor state `1` -> cycle.

### Tarjan's Bridge / Articulation Point
**Pattern:** DFS with discovery & low-link times
**Key Insight:** A bridge exists if a neighbor can only be reached via the current node (its low-link value is strictly greater than the current node's discovery time).
```cpp
void dfs(int u, int p) {
    disc[u] = low[u] = ++timer;
    for (int v : adj[u]) {
        if (v == p) continue;
        if (disc[v]) { // back-edge
            low[u] = min(low[u], disc[v]);
        } else {
            dfs(v, u);
            low[u] = min(low[u], low[v]);
            if (low[v] > disc[u]) { // v cannot reach any ancestor of u without using edge u-v
                // Bridge found: u-v
            }
        }
    }
}
```
**Complexity:** Time: `O(V + E)` | Space: `O(V)`
