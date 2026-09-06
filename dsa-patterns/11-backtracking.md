# 11 — Backtracking

## Subsets & Combinations

### Subsets (LC 78)
**Pattern:** Pick / Don't Pick
**Key Insight:** Every element has two independent choices: include it or exclude it. This forms a perfect binary tree of possibilities.
```cpp
void dfs(int i, vector<int>& nums, vector<int>& path, vector<vector<int>>& res) {
    if (i == nums.size()) { res.push_back(path); return; }
    // Pick
    path.push_back(nums[i]); // Choose to include current element
    dfs(i + 1, nums, path, res);
    path.pop_back();
    // Don't Pick
    dfs(i + 1, nums, path, res);
}
```
**Complexity:** O(N * 2^N) Time | O(N) Space

### Subsets II (LC 90)
**Pattern:** Pick / Don't Pick with duplicate skipping
**Key Insight:** To avoid duplicate subsets, if we choose to exclude an element, we must also exclude all its identical copies at the current level.
```cpp
void dfs(int i, vector<int>& nums, vector<int>& path, vector<vector<int>>& res) {
    if (i == nums.size()) { res.push_back(path); return; }
    // Pick
    path.push_back(nums[i]);
    dfs(i + 1, nums, path, res);
    path.pop_back();
    // Skip duplicates
    while (i + 1 < nums.size() && nums[i] == nums[i + 1]) i++; // Skip identical elements to avoid duplicate branches
    // Don't Pick
    dfs(i + 1, nums, path, res);
}
// Note: nums must be sorted initially!
```
**Complexity:** O(N * 2^N) Time | O(N) Space

### Combination Sum (LC 39)
**Pattern:** Unlimited choices, move index only when not picking
**Key Insight:** Since we can reuse elements, the "pick" branch stays on the same index, while the "skip" branch moves to the next index.
```cpp
void dfs(int i, int target, vector<int>& nums, vector<int>& path, vector<vector<int>>& res) {
    if (target == 0) { res.push_back(path); return; }
    if (i == nums.size() || target < 0) return;
    // Pick same element again
    path.push_back(nums[i]);
    dfs(i, target - nums[i], nums, path, res); // Pass 'i' instead of 'i+1' to allow reusing the element
    path.pop_back();
    // Move to next
    dfs(i + 1, target, nums, path, res);
}
```
**Complexity:** O(N^(T/M)) Time (T=target, M=min_val) | O(T/M) Space

### Combination Sum II (LC 40)
**Pattern:** For-loop based DFS to skip duplicates across levels
**Key Insight:** Sorting groups duplicates together. Skipping identical adjacent elements in the for-loop prevents duplicate combinations.
```cpp
void dfs(int start, int target, vector<int>& nums, vector<int>& path, vector<vector<int>>& res) {
    if (target == 0) { res.push_back(path); return; }
    for (int i = start; i < nums.size(); ++i) {
        if (i > start && nums[i] == nums[i - 1]) continue; // Skip duplicates at the same tree level
        if (nums[i] > target) break; // Optimization
        path.push_back(nums[i]);
        dfs(i + 1, target - nums[i], nums, path, res);
        path.pop_back();
    }
}
// Note: nums must be sorted
```
**Complexity:** O(2^N) Time | O(N) Space

## Permutations

### Permutations (LC 46)
**Pattern:** Swap-based backtracking
**Key Insight:** Swapping elements in-place generates all orderings. Backtracking (swapping back) restores the original state for the next branch.
```cpp
void dfs(int i, vector<int>& nums, vector<vector<int>>& res) {
    if (i == nums.size()) { res.push_back(nums); return; }
    for (int j = i; j < nums.size(); ++j) {
        swap(nums[i], nums[j]); // Fix nums[j] at the current position i
        dfs(i + 1, nums, res);
        swap(nums[i], nums[j]);
    }
}
```
**Complexity:** O(N * N!) Time | O(N) Space

### Permutations II (LC 47)
**Pattern:** Hash set per level to avoid duplicate swaps
**Key Insight:** A hash set tracks which numbers have been swapped into the current position to prevent generating duplicate permutations.
```cpp
void dfs(int i, vector<int>& nums, vector<vector<int>>& res) {
    if (i == nums.size()) { res.push_back(nums); return; }
    unordered_set<int> seen;
    for (int j = i; j < nums.size(); ++j) {
        if (seen.count(nums[j])) continue; // Avoid placing the same number at position i again
        seen.insert(nums[j]);
        swap(nums[i], nums[j]);
        dfs(i + 1, nums, res);
        swap(nums[i], nums[j]);
    }
}
```
**Complexity:** O(N * N!) Time | O(N) Space

## String / Matrix Backtracking

### Word Search (LC 79)
**Pattern:** 2D Grid DFS with state modification
**Key Insight:** DFS explores paths. Modifying the grid in-place prevents revisiting the same cell in the current path without needing extra space.
```cpp
bool dfs(vector<vector<char>>& b, string& w, int i, int r, int c) {
    if (i == w.size()) return true;
    if (r<0 || r>=b.size() || c<0 || c>=b[0].size() || b[r][c] != w[i]) return false;
    char temp = b[r][c];
    b[r][c] = '#'; // Mark as visited for the current DFS path
    bool res = dfs(b, w, i+1, r+1, c) || dfs(b, w, i+1, r-1, c) || 
               dfs(b, w, i+1, r, c+1) || dfs(b, w, i+1, r, c-1);
    b[r][c] = temp; // Unmark
    return res;
}
```
**Complexity:** O(R * C * 4^L) Time | O(L) Space

### Palindrome Partitioning (LC 131)
**Pattern:** Partitioning with prefix check
**Key Insight:** Slice the string at every possible index. If the prefix is a palindrome, recursively partition the remaining suffix.
```cpp
void dfs(int start, string& s, vector<string>& path, vector<vector<string>>& res) {
    if (start == s.size()) { res.push_back(path); return; }
    for (int i = start; i < s.size(); ++i) {
        if (isPal(s, start, i)) { // Only branch if the current prefix is a valid palindrome
            path.push_back(s.substr(start, i - start + 1));
            dfs(i + 1, s, path, res);
            path.pop_back();
        }
    }
}
```
**Complexity:** O(N * 2^N) Time | O(N) Space

### Letter Combinations of Phone Number (LC 17)
**Pattern:** Index matching with dictionary mapping
**Key Insight:** Each digit maps to a set of characters. DFS recursively picks one character per digit to build combinations.
```cpp
vector<string> mapping = {"", "", "abc", "def", "ghi", "jkl", "mno", "pqrs", "tuv", "wxyz"};
void dfs(int i, string& d, string& path, vector<string>& res) {
    if (i == d.size()) { res.push_back(path); return; }
    for (char c : mapping[d[i] - '0']) { // Iterate through all letters mapped to the current digit
        path.push_back(c);
        dfs(i + 1, d, path, res);
        path.pop_back();
    }
}
```
**Complexity:** O(4^N) Time | O(N) Space

### Generate Parentheses (LC 22)
**Pattern:** Track open/close counts
**Key Insight:** Build strings incrementally. Adding '(' is valid if we haven't reached 'n', and adding ')' is valid if there are unmatched '('s.
```cpp
void dfs(int open, int close, int n, string& path, vector<string>& res) {
    if (path.size() == 2 * n) { res.push_back(path); return; }
    if (open < n) {
        path.push_back('(');
        dfs(open + 1, close, n, path, res);
        path.pop_back();
    }
    if (close < open) { // Can only close if there's an unmatched open parenthesis
        path.push_back(')');
        dfs(open, close + 1, n, path, res);
        path.pop_back();
    }
}
```
**Complexity:** O(4^N / sqrt(N)) Time | O(N) Space

## Hard Puzzles

### N-Queens (LC 51)
**Pattern:** Columns and Diagonals tracking
**Key Insight:** Use arrays/sets to track attacked columns and diagonals. Diagonals have constant `r+c` or `r-c` values.
```cpp
void dfs(int r, int n, vector<string>& board, vector<vector<string>>& res, 
         vector<int>& cols, vector<int>& diag1, vector<int>& diag2) {
    if (r == n) { res.push_back(board); return; }
    for (int c = 0; c < n; ++c) {
        if (cols[c] || diag1[r+c] || diag2[r-c+n]) continue;
        board[r][c] = 'Q';
        cols[c] = diag1[r+c] = diag2[r-c+n] = 1; // Mark column and both diagonals as attacked
        dfs(r + 1, n, board, res, cols, diag1, diag2);
        board[r][c] = '.';
        cols[c] = diag1[r+c] = diag2[r-c+n] = 0;
    }
}
```
**Complexity:** O(N!) Time | O(N) Space

### Sudoku Solver (LC 37)
**Pattern:** Find empty, try 1-9, validate
**Key Insight:** DFS attempts 1-9 for every empty cell. If a placement leads to no solution later, it backtracks and tries the next number.
```cpp
bool solve(vector<vector<char>>& b) {
    for (int r = 0; r < 9; ++r) {
        for (int c = 0; c < 9; ++c) {
            if (b[r][c] == '.') {
                for (char k = '1'; k <= '9'; ++k) {
                    if (isValid(b, r, c, k)) { // Check row, column, and 3x3 grid constraints
                        b[r][c] = k;
                        if (solve(b)) return true;
                        b[r][c] = '.';
                    }
                }
                return false;
            }
        }
    }
    return true;
}
```
**Complexity:** O(9^(81)) Time | O(81) Space
