# 13 — Dynamic Programming (2D & Advanced)

## Grids

### Unique Paths (LC 62)
Pattern: Math (Combinatorics) or 1D rolling array
**Key Insight:** Only the current and previous row matter, so collapse 2D DP into 1D. `dp[j] += dp[j-1]` combines paths from above and left.
```cpp
int uniquePaths(int m, int n) {
    vector<int> dp(n, 1);
    for (int i = 1; i < m; ++i) {
        for (int j = 1; j < n; ++j) {
            dp[j] += dp[j - 1]; // paths from above (dp[j]) + paths from left (dp[j-1])
        }
    }
    return dp.back();
}
```
**Complexity:** O(M * N) Time | O(N) Space

### Unique Paths II (LC 63)
Pattern: Handle obstacles
**Key Insight:** If there's an obstacle, 0 paths go through it; otherwise, inherit the paths from left and above using a 1D array.
```cpp
int uniquePathsWithObstacles(vector<vector<int>>& obs) {
    int n = obs[0].size();
    vector<long> dp(n, 0);
    dp[0] = (obs[0][0] == 0) ? 1 : 0;
    for (int i = 0; i < obs.size(); ++i) {
        for (int j = 0; j < n; ++j) {
            if (obs[i][j] == 1) dp[j] = 0; // obstacle blocks all paths
            else if (j > 0) dp[j] += dp[j - 1]; // add paths from left
        }
    }
    return dp.back();
}
```
**Complexity:** O(M * N) Time | O(N) Space

### Minimum Path Sum (LC 64)
Pattern: In-place grid DP
**Key Insight:** We can reuse the input grid to store the minimum cost to reach each cell, picking the min of top or left cell plus current cost.
```cpp
int minPathSum(vector<vector<int>>& grid) {
    int m = grid.size(), n = grid[0].size();
    for(int i=0; i<m; ++i) {
        for(int j=0; j<n; ++j) {
            if(i==0 && j==0) continue;
            int up = (i==0) ? INT_MAX : grid[i-1][j];
            int left = (j==0) ? INT_MAX : grid[i][j-1];
            grid[i][j] += min(up, left); // min cost from up or left
        }
    }
    return grid[m-1][n-1];
}
```
**Complexity:** O(M * N) Time | O(1) Space

### Maximal Square (LC 221)
Pattern: dp(i,j) = min(up, left, diag) + 1
**Key Insight:** A square of size `k` ending at `(i,j)` requires squares of size `k-1` ending at `(i-1,j)`, `(i,j-1)`, and `(i-1,j-1)`.
```cpp
int maximalSquare(vector<vector<char>>& matrix) {
    int m = matrix.size(), n = matrix[0].size(), max_side = 0;
    vector<vector<int>> dp(m + 1, vector<int>(n + 1, 0));
    for (int i = 1; i <= m; ++i) {
        for (int j = 1; j <= n; ++j) {
            if (matrix[i - 1][j - 1] == '1') {
                // bounded by the smallest neighboring square
                dp[i][j] = min({dp[i - 1][j], dp[i][j - 1], dp[i - 1][j - 1]}) + 1;
                max_side = max(max_side, dp[i][j]);
            }
        }
    }
    return max_side * max_side;
}
```
**Complexity:** O(M * N) Time | O(M * N) Space

## Strings

### Longest Common Subsequence (LC 1143)
Pattern: 2D DP for string matching
**Key Insight:** If characters match, extend the LCS without them. If they differ, take the max LCS by dropping one character from either string.
```cpp
int longestCommonSubsequence(string t1, string t2) {
    int m = t1.size(), n = t2.size();
    vector<vector<int>> dp(m + 1, vector<int>(n + 1, 0));
    for (int i = 1; i <= m; ++i) {
        for (int j = 1; j <= n; ++j) {
            if (t1[i-1] == t2[j-1]) dp[i][j] = 1 + dp[i-1][j-1]; // match extends diagonal
            else dp[i][j] = max(dp[i-1][j], dp[i][j-1]); // mismatch pulls from top or left
        }
    }
    return dp[m][n];
}
```
**Complexity:** O(M * N) Time | O(M * N) Space

### Edit Distance (LC 72)
Pattern: min of insert, delete, replace
**Key Insight:** If characters match, no cost. Else, take the min cost of inserting, deleting, or replacing, plus 1 for the operation.
```cpp
int minDistance(string w1, string w2) {
    int m = w1.size(), n = w2.size();
    vector<vector<int>> dp(m + 1, vector<int>(n + 1));
    for (int i = 0; i <= m; ++i) dp[i][0] = i; // deleting all characters
    for (int j = 0; j <= n; ++j) dp[0][j] = j; // inserting all characters
    for (int i = 1; i <= m; ++i) {
        for (int j = 1; j <= n; ++j) {
            if (w1[i-1] == w2[j-1]) dp[i][j] = dp[i-1][j-1];
            // min of delete (i-1), insert (j-1), or replace (i-1, j-1)
            else dp[i][j] = 1 + min({dp[i-1][j], dp[i][j-1], dp[i-1][j-1]});
        }
    }
    return dp[m][n];
}
```
**Complexity:** O(M * N) Time | O(M * N) Space

### Interleaving String (LC 97)
Pattern: dp[i][j] uses s1[i] or s2[j] matching s3[i+j]
**Key Insight:** True if we can form the prefix of `s3` using valid interleaving prefixes of `s1` and `s2`. We recursively check which string provides the matching character.
```cpp
bool isInterleave(string s1, string s2, string s3) {
    if(s1.size() + s2.size() != s3.size()) return false;
    int m = s1.size(), n = s2.size();
    vector<bool> dp(n + 1, false);
    for (int i = 0; i <= m; ++i) {
        for (int j = 0; j <= n; ++j) {
            if (i == 0 && j == 0) dp[j] = true;
            else if (i == 0) dp[j] = dp[j-1] && s2[j-1] == s3[i+j-1];
            else if (j == 0) dp[j] = dp[j] && s1[i-1] == s3[i+j-1];
            // string 1 matches OR string 2 matches
            else dp[j] = (dp[j] && s1[i-1] == s3[i+j-1]) || (dp[j-1] && s2[j-1] == s3[i+j-1]);
        }
    }
    return dp[n];
}
```
**Complexity:** O(M * N) Time | O(N) Space

### Longest Palindromic Subsequence (LC 516)
Pattern: LCS of string and its reverse
**Key Insight:** The longest palindromic subsequence of `s` is simply the Longest Common Subsequence between `s` and reversed `s`.
```cpp
int longestPalindromeSubseq(string s) {
    string rev = s;
    reverse(rev.begin(), rev.end());
    int n = s.size();
    vector<int> dp(n + 1, 0);
    for (int i = 1; i <= n; ++i) {
        int prev = 0; // saves diagonal (i-1, j-1)
        for (int j = 1; j <= n; ++j) {
            int temp = dp[j];
            if (s[i-1] == rev[j-1]) dp[j] = 1 + prev;
            else dp[j] = max(dp[j], dp[j-1]);
            prev = temp;
        }
    }
    return dp[n];
}
```
**Complexity:** O(N^2) Time | O(N) Space

### Distinct Subsequences (LC 115)
Pattern: if match, pick or don't pick from S. Unsigned needed for overflow.
**Key Insight:** If characters match, we can either use it to match `t` (adding previous combinations) or skip it (keeping current combinations).
```cpp
int numDistinct(string s, string t) {
    vector<unsigned int> dp(t.size() + 1, 0);
    dp[0] = 1;
    for (int i = 1; i <= s.size(); ++i) {
        for (int j = t.size(); j >= 1; --j) { // iterate backwards to use 1D array
            if (s[i-1] == t[j-1]) dp[j] += dp[j-1];
        }
    }
    return dp.back();
}
```
**Complexity:** O(M * N) Time | O(N) Space

### Regular Expression Matching (LC 10)
Pattern: DP state machine matching
**Key Insight:** For `*`, either skip the `char*` pattern (0 matches) or consume a character from `s` if it matches (1 or more matches).
```cpp
bool isMatch(string s, string p) {
    int m = s.size(), n = p.size();
    vector<vector<bool>> dp(m + 1, vector<bool>(n + 1, false));
    dp[0][0] = true;
    for (int j = 1; j <= n; ++j)
        if (p[j-1] == '*') dp[0][j] = dp[0][j-2]; // matches 0 occurrences
        
    for (int i = 1; i <= m; ++i) {
        for (int j = 1; j <= n; ++j) {
            if (p[j-1] == s[i-1] || p[j-1] == '.') dp[i][j] = dp[i-1][j-1];
            else if (p[j-1] == '*') {
                // skip wildcard OR (char matches and we keep wildcard)
                dp[i][j] = dp[i][j-2] || ((s[i-1] == p[j-2] || p[j-2] == '.') && dp[i-1][j]);
            }
        }
    }
    return dp[m][n];
}
```
**Complexity:** O(M * N) Time | O(M * N) Space

### Wildcard Matching (LC 44)
Pattern: Star matches empty sequence or multiple characters
**Key Insight:** For `*`, either skip the star (match empty) or consume a character from `s` while keeping the star available.
```cpp
bool isMatch(string s, string p) {
    int m = s.size(), n = p.size();
    vector<vector<bool>> dp(m + 1, vector<bool>(n + 1, false));
    dp[0][0] = true;
    for (int j = 1; j <= n; ++j)
        if (p[j-1] == '*') dp[0][j] = dp[0][j-1]; // star matches empty string
        
    for (int i = 1; i <= m; ++i) {
        for (int j = 1; j <= n; ++j) {
            if (p[j-1] == '?' || s[i-1] == p[j-1]) dp[i][j] = dp[i-1][j-1];
            else if (p[j-1] == '*') dp[i][j] = dp[i-1][j] || dp[i][j-1]; // consume char OR match empty
        }
    }
    return dp[m][n];
}
```
**Complexity:** O(M * N) Time | O(M * N) Space

## Advanced & Subarrays

### Best Time to Buy/Sell Stock with Cooldown (LC 309)
Pattern: State Machine (Buy, Sell, Rest)
**Key Insight:** Track max profit at each state. `hold` updates by resting or buying from `rest`. `sold` updates by selling from `hold`.
```cpp
int maxProfit(vector<int>& prices) {
    int hold = INT_MIN, sold = 0, rest = 0;
    for (int p : prices) {
        int next_hold = max(hold, rest - p);
        int next_sold = hold + p;
        int next_rest = max(rest, sold);
        hold = next_hold;
        sold = next_sold;
        rest = next_rest; // enforces 1 day cooldown naturally
    }
    return max(sold, rest);
}
```
**Complexity:** O(N) Time | O(1) Space

### Target Sum (LC 494)
Pattern: Shifted DP or 0/1 Knapsack mapping
**Key Insight:** The problem mathematically reduces to finding a subset with a specific sum `(sum + S) / 2`, matching the 0/1 knapsack pattern.
```cpp
int findTargetSumWays(vector<int>& nums, int S) {
    int sum = accumulate(nums.begin(), nums.end(), 0);
    if (sum < abs(S) || (sum + S) % 2 != 0) return 0;
    int target = (sum + S) / 2;
    vector<int> dp(target + 1, 0);
    dp[0] = 1;
    for (int n : nums) {
        for (int i = target; i >= n; --i) { // iterate backwards for 0/1 knapsack
            dp[i] += dp[i - n];
        }
    }
    return dp[target];
}
```
**Complexity:** O(N * Target) Time | O(Target) Space

### Coin Change 2 (LC 518)
Pattern: Combinations formula (order matters vs doesn't)
**Key Insight:** By putting the coin loop outermost, we prevent permutations (e.g., `1+2` and `2+1`), only counting unique combinations.
```cpp
int change(int amount, vector<int>& coins) {
    vector<int> dp(amount + 1, 0);
    dp[0] = 1;
    for (int c : coins) { // coin outermost = combination, not permutation
        for (int i = c; i <= amount; ++i) {
            dp[i] += dp[i - c];
        }
    }
    return dp[amount];
}
```
**Complexity:** O(N * amount) Time | O(amount) Space

### Burst Balloons (LC 312)
Pattern: Interval DP (think backwards, pick LAST balloon to burst)
**Key Insight:** If we think forward, adjacent elements change. If we think backwards (which balloon bursts *last*), the subproblems become independent.
```cpp
int maxCoins(vector<int>& nums) {
    vector<int> a(nums.size() + 2, 1);
    for (int i = 0; i < nums.size(); ++i) a[i+1] = nums[i]; // pad with 1s
    int n = a.size();
    vector<vector<int>> dp(n, vector<int>(n, 0));
    
    for (int len = 2; len < n; ++len) { // length of interval
        for (int l = 0; l < n - len; ++l) { // left boundary
            int r = l + len; // right boundary
            for (int k = l + 1; k < r; ++k) { // last balloon to burst in interval
                dp[l][r] = max(dp[l][r], dp[l][k] + dp[k][r] + a[l]*a[k]*a[r]);
            }
        }
    }
    return dp[0][n-1];
}
```
**Complexity:** O(N^3) Time | O(N^2) Space

### Stone Game (LC 877)
Pattern: Interval DP minimax
**Key Insight:** Alice can always pick the optimal subset of elements (all evens or all odds) to maximize her score, guaranteeing a win.
```cpp
bool stoneGame(vector<int>& piles) {
    return true; // Math shortcut: Alice always wins if optimal
    /* Interval DP logic:
    int n = piles.size();
    vector<vector<int>> dp(n, vector<int>(n, 0));
    for (int i = 0; i < n; i++) dp[i][i] = piles[i];
    for (int d = 1; d < n; d++) // gap size
        for (int i = 0; i < n - d; i++) // left boundary
            // max of (pick left - optimal opponent) or (pick right - optimal opponent)
            dp[i][i+d] = max(piles[i]-dp[i+1][i+d], piles[i+d]-dp[i][i+d-1]);
    return dp[0][n-1] > 0;
    */
}
```
**Complexity:** O(1) Time | O(1) Space

### Matrix Chain Multiplication Pattern
Pattern: Interval DP splitting sequence
**Key Insight:** Try every possible split point `k` within interval `[i, j]` to find the minimum cost of multiplying the two resulting matrices.
```cpp
int matrixChainOrder(vector<int>& p) {
    int n = p.size();
    vector<vector<int>> dp(n, vector<int>(n, 0));
    for (int len = 2; len < n; len++) {
        for (int i = 1; i < n - len + 1; i++) {
            int j = i + len - 1;
            dp[i][j] = INT_MAX;
            for (int k = i; k < j; k++) { // k is the split point
                int cost = dp[i][k] + dp[k+1][j] + p[i-1]*p[k]*p[j];
                dp[i][j] = min(dp[i][j], cost);
            }
        }
    }
    return dp[1][n-1];
}
```
**Complexity:** O(N^3) Time | O(N^2) Space

### Bitmask DP Template (TSP Variant)
Pattern: Use bits to represent sets visited
**Key Insight:** A bitmask effectively compresses the state of "visited cities" into an integer, avoiding permutations in standard recursion.
```cpp
int tsp(int mask, int pos, int n, vector<vector<int>>& dist, vector<vector<int>>& dp) {
    if (mask == (1 << n) - 1) return dist[pos][0]; // Return to start
    if (dp[mask][pos] != -1) return dp[mask][pos];
    
    int ans = INT_MAX;
    for (int city = 0; city < n; city++) {
        if ((mask & (1 << city)) == 0) { // If unvisited (bit is 0)
            int newAns = dist[pos][city] + tsp(mask | (1 << city), city, n, dist, dp);
            ans = min(ans, newAns);
        }
    }
    return dp[mask][pos] = ans;
}
```
**Complexity:** O(N^2 * 2^N) Time | O(N * 2^N) Space
