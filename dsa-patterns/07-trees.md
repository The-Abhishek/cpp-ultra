# 07 — Trees (Binary Tree & BST)

### Invert Binary Tree (LC 226)
**Pattern:** Post-order / Pre-order Swap
**Key Insight:** At each node, swap left and right children, then recurse. Works because swapping at every level inverts the entire tree.
```cpp
if (!root) return nullptr;
swap(root->left, root->right); // Swap current level
invertTree(root->left);
invertTree(root->right);
return root;
```
**Complexity:** Time: O(N) | Space: O(H)

### Maximum Depth (LC 104)
**Pattern:** Post-order DFS
**Key Insight:** The maximum depth is 1 (for the current node) plus the maximum of the depths of its left and right subtrees.
```cpp
if (!root) return 0;
return 1 + max(maxDepth(root->left), maxDepth(root->right)); // Bottom-up calculation
```
**Complexity:** Time: O(N) | Space: O(H)

### Diameter of Binary Tree (LC 543)
**Pattern:** Post-order DFS + Global Max
**Key Insight:** The longest path passing through a node is the sum of the max depths of its left and right subtrees. We track this global maximum while returning the max depth.
```cpp
int maxD = 0;
int dfs(TreeNode* root) {
    if (!root) return 0;
    int left = dfs(root->left);
    int right = dfs(root->right);
    maxD = max(maxD, left + right); // Update global diameter
    return 1 + max(left, right); // Return depth to parent
}
```
**Complexity:** Time: O(N) | Space: O(H)

### Balanced Binary Tree (LC 110)
**Pattern:** Post-order DFS with early exit
**Key Insight:** Return -1 to indicate imbalance; propagate this -1 upwards to avoid redundant checks.
```cpp
int checkHeight(TreeNode* root) {
    if (!root) return 0;
    int left = checkHeight(root->left);
    if (left == -1) return -1; // Propagate imbalance
    int right = checkHeight(root->right);
    if (right == -1 || abs(left - right) > 1) return -1; // Check imbalance at current node
    return 1 + max(left, right);
}
// Return checkHeight(root) != -1;
```
**Complexity:** Time: O(N) | Space: O(H)

### Same Tree (LC 100)
**Pattern:** Pre-order Parallel Traversal
**Key Insight:** Traverse both trees simultaneously, comparing the current node's value and recursively checking their children.
```cpp
if (!p && !q) return true;
if (!p || !q || p->val != q->val) return false;
return isSameTree(p->left, q->left) && isSameTree(p->right, q->right);
```
**Complexity:** Time: O(min(N, M)) | Space: O(min(H1, H2))

### Subtree of Another Tree (LC 572)
**Pattern:** DFS + Same Tree Check
**Key Insight:** Check if the current tree matches the subtree. If not, recursively search the left and right children.
```cpp
if (!subRoot) return true;
if (!root) return false;
if (isSameTree(root, subRoot)) return true;
return isSubtree(root->left, subRoot) || isSubtree(root->right, subRoot);
```
**Complexity:** Time: O(N * M) | Space: O(H)

### Lowest Common Ancestor (LC 236)
**Pattern:** Post-order DFS Bubbling
**Key Insight:** If a node is either p or q, return it. If both left and right subtrees return a valid node, the current node is the LCA.
```cpp
if (!root || root == p || root == q) return root; // Found target or leaf
TreeNode* left = lowestCommonAncestor(root->left, p, q);
TreeNode* right = lowestCommonAncestor(root->right, p, q);
if (left && right) return root; // Both targets found in different subtrees
return left ? left : right; // Bubble up the found target
```
**Complexity:** Time: O(N) | Space: O(H)

### Binary Tree Level Order Traversal (LC 102)
**Pattern:** BFS with Queue Size
**Key Insight:** Using `queue.size()` before processing ensures we only process nodes for the current level in one iteration.
```cpp
vector<vector<int>> res;
if (!root) return res;
queue<TreeNode*> q{{root}};
while (!q.empty()) {
    int sz = q.size(); // Snapshot of current level's size
    vector<int> level;
    while (sz--) {
        TreeNode* node = q.front(); q.pop();
        level.push_back(node->val);
        if (node->left) q.push(node->left);
        if (node->right) q.push(node->right);
    }
    res.push_back(move(level));
}
return res;
```
**Complexity:** Time: O(N) | Space: O(N)

### Binary Tree Right Side View (LC 199)
**Pattern:** BFS taking last element (or DFS right-first)
**Key Insight:** Use level-order traversal, and record the very last node processed at each level.
```cpp
vector<int> res;
if (!root) return res;
queue<TreeNode*> q{{root}};
while (!q.empty()) {
    int sz = q.size();
    for (int i = 0; i < sz; ++i) {
        TreeNode* node = q.front(); q.pop();
        if (i == sz - 1) res.push_back(node->val); // Capture the rightmost node
        if (node->left) q.push(node->left);
        if (node->right) q.push(node->right);
    }
}
return res;
```
**Complexity:** Time: O(N) | Space: O(N)

### Validate BST (LC 98)
**Pattern:** DFS with Range Boundaries
**Key Insight:** Each node must satisfy a valid range `(minVal, maxVal)`. Left children update `maxVal`, right children update `minVal`.
```cpp
bool isValid(TreeNode* root, long minVal, long maxVal) {
    if (!root) return true;
    if (root->val <= minVal || root->val >= maxVal) return false;
    return isValid(root->left, minVal, root->val) &&  // Constrain left subtree's max
           isValid(root->right, root->val, maxVal);   // Constrain right subtree's min
}
// Start: isValid(root, LONG_MIN, LONG_MAX);
```
**Complexity:** Time: O(N) | Space: O(H)

### Kth Smallest Element in BST (LC 230)
**Pattern:** In-order Traversal
**Key Insight:** In-order traversal visits BST nodes in sorted order. We stop when we reach the `k`-th node.
```cpp
int res = -1, kCount = 0;
void inorder(TreeNode* root, int k) {
    if (!root || kCount >= k) return; // Early exit if k is reached
    inorder(root->left, k);
    if (++kCount == k) { // Increment and check against k
        res = root->val;
        return;
    }
    inorder(root->right, k);
}
```
**Complexity:** Time: O(H + K) | Space: O(H)

### Construct Binary Tree from Preorder and Inorder (LC 105)
**Pattern:** Hash Map + Divide and Conquer
**Key Insight:** Preorder gives the root. Inorder gives the sizes of left/right subtrees. Use a hash map for fast root index lookup in inorder.
```cpp
unordered_map<int, int> inMap; // val -> idx
TreeNode* build(vector<int>& pre, int preStart, int preEnd, int inStart, int inEnd) {
    if (preStart > preEnd || inStart > inEnd) return nullptr;
    TreeNode* root = new TreeNode(pre[preStart]);
    int inRoot = inMap[root->val]; // Find root in inorder array
    int numsLeft = inRoot - inStart; // Size of left subtree
    root->left = build(pre, preStart + 1, preStart + numsLeft, inStart, inRoot - 1);
    root->right = build(pre, preStart + numsLeft + 1, preEnd, inRoot + 1, inEnd);
    return root;
}
```
**Complexity:** Time: O(N) | Space: O(N)

### Serialize and Deserialize Binary Tree (LC 297)
**Pattern:** Pre-order Traversal with stringstream
**Key Insight:** Pre-order traversal with a special marker (like `#`) for null pointers uniquely identifies a binary tree's structure.
```cpp
void serialize(TreeNode* root, ostringstream& out) {
    if (!root) { out << "# "; return; } // Marker for null
    out << root->val << " ";
    serialize(root->left, out);
    serialize(root->right, out);
}
TreeNode* deserialize(istringstream& in) {
    string val; in >> val;
    if (val == "#") return nullptr;
    TreeNode* root = new TreeNode(stoi(val));
    root->left = deserialize(in);
    root->right = deserialize(in);
    return root;
}
```
**Complexity:** Time: O(N) | Space: O(N)

### Binary Tree Maximum Path Sum (LC 124)
**Pattern:** Post-order DFS + Global Max
**Key Insight:** Ignore negative subpaths by taking `max(0, dfs())`. The path can pass through a node combining both left and right, but only one branch can be returned.
```cpp
int maxSum = INT_MIN;
int dfs(TreeNode* root) {
    if (!root) return 0;
    int left = max(0, dfs(root->left)); // Discard negative subpaths
    int right = max(0, dfs(root->right));
    maxSum = max(maxSum, left + right + root->val); // Update global max with split path
    return max(left, right) + root->val; // Return max single branch
}
```
**Complexity:** Time: O(N) | Space: O(H)

### Count Good Nodes (LC 1448)
**Pattern:** Pre-order DFS with Max Tracker
**Key Insight:** Pass down the maximum value seen so far along the path. If the current node is >= this max, it's a "good" node.
```cpp
int dfs(TreeNode* root, int maxSoFar) {
    if (!root) return 0;
    int res = root->val >= maxSoFar ? 1 : 0;
    maxSoFar = max(maxSoFar, root->val); // Update max for children
    res += dfs(root->left, maxSoFar);
    res += dfs(root->right, maxSoFar);
    return res;
}
```
**Complexity:** Time: O(N) | Space: O(H)

### Vertical Order Traversal (LC 987)
**Pattern:** BFS/DFS + TreeMap + Priority Queue
**Key Insight:** Assign coordinates `(x, y)` to each node. Use an ordered map of maps of multisets to group and sort nodes automatically by `x`, then `y`, then value.
```cpp
map<int, map<int, multiset<int>>> nodes; // x -> y -> values (sorted automatically)
void dfs(TreeNode* root, int x, int y) {
    if (!root) return;
    nodes[x][y].insert(root->val);
    dfs(root->left, x - 1, y + 1); // Move left: x-1, down: y+1
    dfs(root->right, x + 1, y + 1); // Move right: x+1, down: y+1
}
// Combine grouped elements into vector<vector<int>> res
```
**Complexity:** Time: O(N log N) | Space: O(N)

### Morris Traversal (O(1) space inorder)
**Pattern:** Threaded Binary Tree
**Key Insight:** Temporarily modify the tree to create threads from predecessors back to current nodes, eliminating the need for a call stack.
```cpp
TreeNode* curr = root;
while (curr) {
    if (!curr->left) {
        visit(curr);
        curr = curr->right;
    } else {
        TreeNode* prev = curr->left;
        while (prev->right && prev->right != curr) prev = prev->right; // Find inorder predecessor
        
        if (!prev->right) {
            prev->right = curr; // Thread back to curr
            curr = curr->left;
        } else {
            prev->right = nullptr; // Unthread to restore tree
            visit(curr);
            curr = curr->right;
        }
    }
}
```
**Complexity:** Time: O(N) | Space: O(1)
