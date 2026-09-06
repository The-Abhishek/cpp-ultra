# 12 — Dynamic Programming (1D)

## Fib / Stairs

### Fibonacci Pattern Template
**Pattern:** Track previous 2 states
**Key Insight:** To get the Nth Fibonacci number, we only need to remember the previous two numbers, optimizing space from O(N) to O(1).
```cpp
int fib(int n) {
    if (n < 2) return n;
    int a = 0, b = 1;
    for (int i = 2; i <= n; ++i) {
        int c = a + b; // Current state only depends on the last two states
        a = b;
        b = c;
    }
    return b;
}
```
**Complexity:** O(N) Time | O(1) Space

### Climbing Stairs (LC 70)
**Pattern:** Fibonacci sequence
**Key Insight:** To reach step N, you must have jumped from N-1 or N-2. The number of ways is exactly the Fibonacci sequence.
```cpp
int climbStairs(int n) {
    int a = 1, b = 1; // f(0), f(1)
    for (int i = 2; i <= n; ++i) {
        int c = a + b; // Ways to reach i is sum of ways to reach i-1 and i-2
        a = b; b = c;
    }
    return b;
}
```
**Complexity:** O(N) Time | O(1) Space

### Min Cost Climbing Stairs (LC 746)
**Pattern:** DP[i] = cost[i] + min(DP[i-1], DP[i-2])
**Key Insight:** The minimum cost to reach step `i` is the cost of step `i` plus the minimum cost of reaching either `i-1` or `i-2`.
```cpp
int minCostClimbingStairs(vector<int>& cost) {
    int n = cost.size();
    int dp1 = cost[0], dp2 = cost[1];
    for (int i = 2; i < n; ++i) {
        int cur = cost[i] + min(dp1, dp2); // Cost to land on current step + min cost to get here
        dp1 = dp2; dp2 = cur;
    }
    return min(dp1, dp2);
}
```
**Complexity:** O(N) Time | O(1) Space

## Pick / Skip (House Robber)

### House Robber (LC 198)
**Pattern:** Max of (skip current, pick current + skip previous)
**Key Insight:** For each house, you either rob it (adding its value to the max from two houses ago) or skip it (keeping the max from the previous house).
```cpp
int rob(vector<int>& nums) {
    int rob = 0, noRob = 0;
    for (int x : nums) {
        int newRob = noRob + x; // If we rob this house, we couldn't have robbed the previous one
        noRob = max(noRob, rob);
        rob = newRob;
    }
    return max(rob, noRob);
}
```
**Complexity:** O(N) Time | O(1) Space

### House Robber II (LC 213)
**Pattern:** Rob[0..n-2] OR Rob[1..n-1]
**Key Insight:** Houses are in a circle, so the first and last houses are adjacent. Solve for two linear ranges: [0, n-2] and [1, n-1] and take the max.
```cpp
int robRange(vector<int>& nums, int start, int end) {
    int r = 0, nr = 0;
    for (int i = start; i <= end; ++i) {
        int new_r = nr + nums[i];
        nr = max(nr, r);
        r = new_r;
    }
    return max(r, nr);
}
int rob(vector<int>& nums) {
    int n = nums.size();
    if (n == 1) return nums[0];
    return max(robRange(nums, 0, n - 2), robRange(nums, 1, n - 1));
}
```
**Complexity:** O(N) Time | O(1) Space

## Substrings / Subarrays

### Longest Palindromic Substring (LC 5)
**Pattern:** Expand from center
**Key Insight:** A palindrome mirrors around its center. Expand outwards from every possible center (2N-1 centers for odd/even lengths).
```cpp
string longestPalindrome(string s) {
    int maxLen = 0, start = 0;
    auto expand = [&](int l, int r) {
        while (l >= 0 && r < s.size() && s[l] == s[r]) l--, r++; // Expand as long as characters match
        if (r - l - 1 > maxLen) maxLen = r - l - 1, start = l + 1;
    };
    for (int i = 0; i < s.size(); ++i) {
        expand(i, i);     // odd length
        expand(i, i + 1); // even length
    }
    return s.substr(start, maxLen);
}
```
**Complexity:** O(N^2) Time | O(1) Space

### Palindromic Substrings (LC 647)
**Pattern:** Count expansions
**Key Insight:** Same as Longest Palindromic Substring, but we count every valid expansion step as a distinct palindrome.
```cpp
int countSubstrings(string s) {
    int res = 0;
    auto count = [&](int l, int r) {
        while (l >= 0 && r < s.size() && s[l] == s[r]) res++, l--, r++; // Every valid expansion is another palindrome substring
    };
    for (int i = 0; i < s.size(); ++i) {
        count(i, i);
        count(i, i + 1);
    }
    return res;
}
```
**Complexity:** O(N^2) Time | O(1) Space

### Maximum Product Subarray (LC 152)
**Pattern:** Track both min and max to handle negative signs
**Key Insight:** Multiplying by a negative number swaps the max and min. Keeping track of the current minimum handles negative signs correctly.
```cpp
int maxProduct(vector<int>& nums) {
    int res = nums[0], cMax = nums[0], cMin = nums[0];
    for (int i = 1; i < nums.size(); ++i) {
        int temp = max({nums[i], cMax * nums[i], cMin * nums[i]}); // Max can come from multiplying with previous min (if negative)
        cMin = min({nums[i], cMax * nums[i], cMin * nums[i]});
        cMax = temp;
        res = max(res, cMax);
    }
    return res;
}
```
**Complexity:** O(N) Time | O(1) Space

### Decode Ways (LC 91)
**Pattern:** DP based on 1-digit or 2-digit combinations
**Key Insight:** A string can be decoded by taking 1 digit (if != 0) or 2 digits (if between 10-26). Similar to Fibonacci with conditions.
```cpp
int numDecodings(string s) {
    if (s.empty() || s[0] == '0') return 0;
    int dp1 = 1, dp2 = 1;
    for (int i = 1; i < s.size(); ++i) {
        int dp = 0;
        if (s[i] != '0') dp += dp1; // Single digit decode is valid
        int two = stoi(s.substr(i - 1, 2));
        if (two >= 10 && two <= 26) dp += dp2;
        dp2 = dp1; dp1 = dp;
    }
    return dp1;
}
```
**Complexity:** O(N) Time | O(1) Space

### Word Break (LC 139)
**Pattern:** dp[i] is true if dp[j] and s[j:i] in dict
**Key Insight:** A string can be broken if a prefix up to `j` can be broken, and the remaining substring from `j` to `i` is in the dictionary.
```cpp
bool wordBreak(string s, vector<string>& wordDict) {
    unordered_set<string> dict(wordDict.begin(), wordDict.end());
    vector<bool> dp(s.size() + 1, false);
    dp[0] = true;
    for (int i = 1; i <= s.size(); ++i) {
        for (int j = 0; j < i; ++j) {
            if (dp[j] && dict.count(s.substr(j, i - j))) { // If prefix up to j is valid and rest is a word
                dp[i] = true; break;
            }
        }
    }
    return dp.back();
}
```
**Complexity:** O(N^3) Time | O(N) Space

## LIS & Knapsacks

### Longest Increasing Subsequence (LC 300)
**Pattern:** Patience sort / Binary Search DP
**Key Insight:** Keep an active array of smallest tail elements for all lengths. Binary search replaces elements to maintain the sequence optimally.
```cpp
int lengthOfLIS(vector<int>& nums) {
    vector<int> res;
    for (int x : nums) {
        auto it = lower_bound(res.begin(), res.end(), x); // Find the first element >= x to replace, keeping tails small
        if (it == res.end()) res.push_back(x);
        else *it = x;
    }
    return res.size();
}
```
**Complexity:** O(N log N) Time | O(N) Space

### 0/1 Knapsack Template (Partition Equal Subset Sum - LC 416)
**Pattern:** Backward 1D DP to prevent reuse
**Key Insight:** Iterate the target backwards. If we go forward, a coin could be reused in the same iteration, breaking the "use once" rule.
```cpp
bool canPartition(vector<int>& nums) {
    int sum = accumulate(nums.begin(), nums.end(), 0);
    if (sum % 2 != 0) return false;
    int target = sum / 2;
    vector<bool> dp(target + 1, false);
    dp[0] = true;
    for (int num : nums) {
        for (int i = target; i >= num; --i) { // Backwards traversal ensures each item is only used once
            dp[i] = dp[i] || dp[i - num];
        }
    }
    return dp[target];
}
```
**Complexity:** O(N * S) Time | O(S) Space

### Unbounded Knapsack Template (Coin Change - LC 322)
**Pattern:** Forward 1D DP (can reuse items)
**Key Insight:** Iterate the amount forwards. The state `dp[i]` depends on `dp[i-c]`, which we want to have already included the current coin (allowing reuse).
```cpp
int coinChange(vector<int>& coins, int amount) {
    vector<int> dp(amount + 1, amount + 1);
    dp[0] = 0;
    for (int c : coins) {
        for (int i = c; i <= amount; ++i) { // Forwards traversal allows reusing the same coin
            dp[i] = min(dp[i], dp[i - c] + 1);
        }
    }
    return dp[amount] > amount ? -1 : dp[amount];
}
```
**Complexity:** O(N * amount) Time | O(amount) Space

### Perfect Squares (LC 279)
**Pattern:** Unbounded knapsack (coins are perfect squares)
**Key Insight:** Identical to Coin Change, but the "coins" are dynamically generated perfect squares up to `n`.
```cpp
int numSquares(int n) {
    vector<int> dp(n + 1, n);
    dp[0] = 0;
    for (int i = 1; i <= n; ++i) {
        for (int j = 1; j * j <= i; ++j) { // Try all perfect squares less than or equal to current amount i
            dp[i] = min(dp[i], dp[i - j * j] + 1);
        }
    }
    return dp[n];
}
```
**Complexity:** O(N * sqrt(N)) Time | O(N) Space

## Jumps

### Jump Game (LC 55)
**Pattern:** Track furthest reachable
**Key Insight:** Greedily keep track of the maximum reachable index. If we encounter an index beyond our maximum reach, we can't reach the end.
```cpp
bool canJump(vector<int>& nums) {
    int reach = 0;
    for (int i = 0; i <= reach && i < nums.size(); ++i) {
        reach = max(reach, i + nums[i]); // Update the furthest index we can jump to
    }
    return reach >= nums.size() - 1;
}
```
**Complexity:** O(N) Time | O(1) Space

### Jump Game II (LC 45)
**Pattern:** BFS/Greedy tracking current level end
**Key Insight:** Implicit BFS. `curEnd` marks the end of the current jump level. When `i` reaches `curEnd`, we must jump, advancing to `curFarthest`.
```cpp
int jump(vector<int>& nums) {
    int jumps = 0, curEnd = 0, curFarthest = 0;
    for (int i = 0; i < nums.size() - 1; ++i) {
        curFarthest = max(curFarthest, i + nums[i]);
        if (i == curEnd) { // Reached the limit of current jump, must take another jump
            jumps++;
            curEnd = curFarthest;
        }
    }
    return jumps;
}
```
**Complexity:** O(N) Time | O(1) Space
