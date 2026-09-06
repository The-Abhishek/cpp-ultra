# 17 — Advanced Patterns (Segment Tree, BIT, String Algorithms)

## Segment Tree Template
**Pattern:** Range Query + Point Update
**Key Insight:** Bottom-up segment tree stores leaves at indices [n, 2n), internal nodes at [1, n). Update propagates up; query walks two pointers inward.
```cpp
class SegmentTree {
    vector<int> tree;
    int n;
public:
    SegmentTree(vector<int>& arr) {
        n = arr.size();
        tree.assign(2 * n, 0);
        for (int i = 0; i < n; i++) tree[n + i] = arr[i];
        for (int i = n - 1; i > 0; --i) tree[i] = tree[i<<1] + tree[i<<1 | 1]; // Left and right child
    }
    void update(int p, int val) {
        for (tree[p += n] = val; p > 1; p >>= 1) tree[p>>1] = tree[p] + tree[p^1]; // Update parent
    }
    int query(int l, int r) {
        int res = 0;
        for (l += n, r += n; l < r; l >>= 1, r >>= 1) {
            if (l & 1) res += tree[l++]; // Add if l is a right child
            if (r & 1) res += tree[--r]; // Add if r is a right child (r is exclusive)
        }
        return res; // Query range [l, r)
    }
};
```
**Complexity:** Build: O(N) | Update/Query: O(log N)

## Binary Indexed Tree / Fenwick Tree
**Pattern:** Prefix sums with updates
**Key Insight:** Each index `i` is responsible for a range of size equal to its lowest set bit (`i & -i`). Updating shifts right, querying shifts left.
```cpp
class BIT {
    vector<int> tree;
public:
    BIT(int n) : tree(n + 1, 0) {}
    void update(int i, int delta) {
        for (++i; i < tree.size(); i += i & -i) tree[i] += delta; // Move to parent (next interval)
    }
    int query(int i) {
        int sum = 0;
        for (++i; i > 0; i -= i & -i) sum += tree[i]; // Move to next prefix range
        return sum; // Query prefix sum up to i
    }
};
```
**Complexity:** Update/Query: O(log N) | Space: O(N)

## Range Sum Query - Mutable (LC 307)
**Pattern:** Segment Tree or BIT (direct application of templates above).
**Key Insight:** Use Segment Tree or BIT to efficiently answer range sum queries and process individual element updates in O(log N) time.

## Count of Smaller Numbers After Self (LC 315)
**Pattern:** BIT + Coordinate Compression / Merge Sort
**Key Insight:** Merge sort naturally counts inversions; when merging, if a right element is smaller, it means it is smaller than all remaining left elements.
```cpp
vector<int> countSmaller(vector<int>& nums) {
    int n = nums.size();
    vector<pair<int, int>> arr(n);
    for (int i = 0; i < n; i++) arr[i] = {nums[i], i};
    vector<int> res(n, 0);
    
    function<void(int, int)> mergeSort = [&](int l, int r) {
        if (l >= r) return;
        int mid = l + (r - l) / 2;
        mergeSort(l, mid); mergeSort(mid + 1, r);
        vector<pair<int, int>> temp;
        int i = l, j = mid + 1, rightCount = 0;
        while (i <= mid && j <= r) {
            if (arr[j].first < arr[i].first) { rightCount++; temp.push_back(arr[j++]); } // Right is smaller
            else { res[arr[i].second] += rightCount; temp.push_back(arr[i++]); }
        }
        while (i <= mid) { res[arr[i].second] += rightCount; temp.push_back(arr[i++]); }
        while (j <= r) temp.push_back(arr[j++]);
        for (int k = l; k <= r; k++) arr[k] = temp[k - l];
    };
    mergeSort(0, n - 1);
    return res;
}
```
**Complexity:** O(N log N) Time | O(N) Space

## Reverse Pairs (LC 493)
**Pattern:** Merge Sort
**Key Insight:** Similar to inversion counting with merge sort, but requires a separate two-pointer pass before merging to check the `nums[i] > 2*nums[j]` condition.
```cpp
int reversePairs(vector<int>& nums) {
    function<int(int, int)> mergeSort = [&](int l, int r) {
        if (l >= r) return 0;
        int mid = l + (r - l) / 2;
        int count = mergeSort(l, mid) + mergeSort(mid + 1, r);
        int j = mid + 1;
        for (int i = l; i <= mid; i++) {
            while (j <= r && nums[i] > 2LL * nums[j]) j++; // Count valid pairs ahead of merge
            count += (j - (mid + 1));
        }
        inplace_merge(nums.begin() + l, nums.begin() + mid + 1, nums.begin() + r + 1);
        return count;
    };
    return mergeSort(0, nums.size() - 1);
}
```
**Complexity:** O(N log N) Time | O(N) Space (for inplace merge/temp)

## KMP String Matching (LC 28)
**Pattern:** LPS Array (Longest Prefix Suffix)
**Key Insight:** The Longest Prefix Suffix (LPS) array tracks matching prefixes, allowing the search to skip redundant comparisons after a mismatch.
```cpp
vector<int> computeLPS(string& pat) {
    int m = pat.length(), len = 0, i = 1;
    vector<int> lps(m, 0);
    while (i < m) {
        if (pat[i] == pat[len]) lps[i++] = ++len;
        else if (len != 0) len = lps[len - 1]; // Fallback to previous longest prefix
        else lps[i++] = 0;
    }
    return lps;
}
int strStr(string txt, string pat) {
    if (pat.empty()) return 0;
    vector<int> lps = computeLPS(pat);
    int i = 0, j = 0;
    while (i < txt.length()) {
        if (pat[j] == txt[i]) { i++; j++; }
        if (j == pat.length()) return i - j;
        else if (i < txt.length() && pat[j] != txt[i]) {
            if (j != 0) j = lps[j - 1]; else i++;
        }
    }
    return -1;
}
```
**Complexity:** O(N + M) Time | O(M) Space

## Rabin-Karp Rolling Hash
**Pattern:** Rolling Hash for substring matching
**Key Insight:** Treats substrings as base-d numbers; rolling hash computes the next window's hash in O(1) by sliding out the old char and adding the new one.
```cpp
int rabinKarp(string txt, string pat) {
    int d = 256, q = 101, m = pat.length(), n = txt.length();
    if (m > n) return -1;
    int p = 0, t = 0, h = 1;
    for (int i = 0; i < m - 1; i++) h = (h * d) % q;
    for (int i = 0; i < m; i++) {
        p = (d * p + pat[i]) % q;
        t = (d * t + txt[i]) % q;
    }
    for (int i = 0; i <= n - m; i++) {
        if (p == t) {
            bool match = true;
            for (int j = 0; j < m; j++) if (txt[i+j] != pat[j]) { match = false; break; }
            if (match) return i;
        }
        if (i < n - m) {
            t = (d * (t - txt[i] * h) + txt[i + m]) % q; // Slide window: remove old, add new char
            if (t < 0) t += q;
        }
    }
    return -1;
}
```
**Complexity:** Avg O(N+M), Worst O(N*M) Time | O(1) Space

## Z-Algorithm
**Pattern:** Z-array (longest common prefix with pattern)
**Key Insight:** The Z-array stores the length of the longest substring starting at `i` that matches the prefix, optimizing via a moving `[l, r]` bounding box.
```cpp
vector<int> getZArray(string s) {
    int n = s.length(), l = 0, r = 0;
    vector<int> z(n, 0);
    for (int i = 1; i < n; i++) {
        if (i <= r) z[i] = min(r - i + 1, z[i - l]); // Use previously computed values within Z-box
        while (i + z[i] < n && s[z[i]] == s[i + z[i]]) z[i]++;
        if (i + z[i] - 1 > r) { l = i; r = i + z[i] - 1; }
    }
    return z;
}
```
**Complexity:** O(N) Time | O(N) Space

## Manacher's Algorithm
**Pattern:** Longest Palindromic Substring in O(N)
**Key Insight:** Transforms string with `#` to handle even/odd palindromes identically, using a mirrored center bounding box to skip redundant palindrome expansion.
```cpp
string longestPalindrome(string s) {
    string t = "^#";
    for (char c : s) { t += c; t += "#"; }
    t += "$";
    int n = t.length(), c = 0, r = 0, maxLen = 0, centerIndex = 0;
    vector<int> p(n, 0);
    for (int i = 1; i < n - 1; i++) {
        int i_mirror = 2 * c - i; // Find mirror index around center c
        if (r > i) p[i] = min(r - i, p[i_mirror]);
        while (t[i + 1 + p[i]] == t[i - 1 - p[i]]) p[i]++;
        if (i + p[i] > r) { c = i; r = i + p[i]; }
        if (p[i] > maxLen) { maxLen = p[i]; centerIndex = i; }
    }
    return s.substr((centerIndex - 1 - maxLen) / 2, maxLen);
}
```
**Complexity:** O(N) Time | O(N) Space

## LFU Cache (LC 460)
**Pattern:** Hash maps + Doubly linked lists per frequency
**Key Insight:** Combining a hash map for values and another map holding doubly linked lists (LRU queues) per frequency allows O(1) updates and minimum frequency tracking.
```cpp
// Core Logic:
unordered_map<int, pair<int, int>> keyVals; // key -> {val, freq}
unordered_map<int, list<int>> freqLists; // freq -> list of keys (LRU ordered queue per freq)
unordered_map<int, list<int>::iterator> keyIters; // key -> iter in freqLists
int minFreq = 0;
// On get/put: Increment freq, move to freqLists[freq+1], update minFreq
// On evict: Pop back of freqLists[minFreq]
```
**Complexity:** O(1) Time | O(N) Space

## Design In-Memory File System (LC 588)
**Pattern:** Trie / N-ary tree with string keys
**Key Insight:** Modeling paths as a Trie where each node stores a map of children allows efficient O(L log K) traversal and directory listing.
```cpp
struct TrieNode {
    bool isFile = false;
    string content = "";
    map<string, TrieNode*> children; // map keeps children lexicographically sorted
};
// Core Logic: Split path by '/', traverse Trie. 
// If ls: return single file name or children keys.
// If mkdir/addContent: create nodes as needed, set isFile and append content.
```
**Complexity:** O(P + L log K) (Path length, Keys count) | O(Total Data) Space

## Prefix Sum 2D
**Pattern:** Inclusion-Exclusion Principle
**Key Insight:** 2D prefix sums use the inclusion-exclusion principle—adding top and left rectangles, subtracting their overlap, and adding the current cell.
```cpp
// Build
vector<vector<int>> dp(m + 1, vector<int>(n + 1, 0));
for (int i = 1; i <= m; i++)
    for (int j = 1; j <= n; j++)
        dp[i][j] = mat[i-1][j-1] + dp[i-1][j] + dp[i][j-1] - dp[i-1][j-1]; // Inclusion-exclusion for 2D area

// Query sum from (r1, c1) to (r2, c2)
int sum = dp[r2+1][c2+1] - dp[r1][c2+1] - dp[r2+1][c1] + dp[r1][c1];
```
**Complexity:** O(M*N) Build, O(1) Query | O(M*N) Space

## Difference Array
**Pattern:** Range additions O(1)
**Key Insight:** Recording `+V` at start `L` and `-V` at end `R+1` means taking the prefix sum naturally applies `V` to exactly the range `[L, R]` in O(1) updates.
```cpp
vector<int> diff(n + 1, 0);
// Add V to range [L, R]
void add(int l, int r, int v) {
    diff[l] += v;
    diff[r + 1] -= v; // Cancel out the addition after R
}
// Retrieve array
vector<int> res(n);
int curr = 0;
for (int i = 0; i < n; i++) {
    curr += diff[i];
    res[i] = curr;
}
```
**Complexity:** O(1) Update, O(N) Build | O(N) Space

## Sparse Table
**Pattern:** Static Range Minimum/Maximum Query (RMQ)
**Key Insight:** Precomputes answers for all intervals of length 2^k. Range queries are answered in O(1) by overlapping two power-of-2 intervals that cover the range.
```cpp
int st[N][K]; // N elements, K = log2(N)
for (int i = 0; i < n; i++) st[i][0] = arr[i];
for (int j = 1; j <= K; j++)
    for (int i = 0; i + (1 << j) <= n; i++)
        st[i][j] = min(st[i][j-1], st[i + (1 << (j - 1))][j - 1]);
        
// Query range [L, R]
int j = log2(R - L + 1); // Largest power of 2 fitting in range
int minimum = min(st[L][j], st[R - (1 << j) + 1][j]);
```
**Complexity:** O(N log N) Build, O(1) Query | O(N log N) Space

## Square Root Decomposition
**Pattern:** Blocks of size sqrt(N)
**Key Insight:** Divides array into blocks of size `sqrt(N)`. Queries sum full blocks in O(1) and iterates partial blocks at the boundaries in O(sqrt(N)).
```cpp
int b_size = sqrt(n);
vector<int> b(n / b_size + 1, 0);
for (int i = 0; i < n; i++) b[i / b_size] += arr[i]; // Precompute block sums

// Query sum [l, r]
int sum = 0;
for (int i = l; i <= r; ) {
    if (i % b_size == 0 && i + b_size - 1 <= r) {
        sum += b[i / b_size]; // Add entire block sum
        i += b_size;
    } else {
        sum += arr[i];
        i++;
    }
}
```
**Complexity:** O(N) Build, O(sqrt(N)) Query/Update | O(sqrt(N)) Extra Space

## Coordinate Compression
**Pattern:** Map sparse large values to contiguous small ranks
**Key Insight:** Sorting unique elements maps wide-range values to contiguous small indices [0, K], essential for fitting data into a BIT or Segment Tree.
```cpp
vector<int> sorted_arr = arr;
sort(sorted_arr.begin(), sorted_arr.end());
sorted_arr.erase(unique(sorted_arr.begin(), sorted_arr.end()), sorted_arr.end());

for (int i = 0; i < arr.size(); i++) {
    arr[i] = lower_bound(sorted_arr.begin(), sorted_arr.end(), arr[i]) - sorted_arr.begin(); // Map original to compressed rank
}
```
**Complexity:** O(N log N) Time | O(N) Extra Space
