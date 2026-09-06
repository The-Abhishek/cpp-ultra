# 03 — Sliding Window

### Best Time to Buy and Sell Stock (LC 121)
**Pattern:** Sliding Window (size 2) tracking Min
**Key Insight:** Track the minimum price seen so far. For each day, calculate profit if sold today and update max profit.
```cpp
int minPrice = INT_MAX, maxProfit = 0;
for (int price : prices) {
    minPrice = min(minPrice, price); // update lowest price seen so far
    maxProfit = max(maxProfit, price - minPrice);
}
return maxProfit;
```
**Complexity:** Time: O(N), Space: O(1)

### Longest Substring Without Repeating Characters (LC 3)
**Pattern:** Dynamic Window with Set/Map
**Key Insight:** Expand right. If a duplicate is seen, shrink from left until the duplicate is removed from the window.
```cpp
unordered_set<char> chars;
int l = 0, maxLen = 0;
for (int r = 0; r < s.length(); ++r) {
    while (chars.count(s[r])) {
        chars.erase(s[l++]); // shrink left until duplicate is removed
    }
    chars.insert(s[r]);
    maxLen = max(maxLen, r - l + 1);
}
return maxLen;
```
**Complexity:** Time: O(N), Space: O(1)

### Longest Repeating Character Replacement (LC 424)
**Pattern:** Dynamic Window with Max Frequency tracking
**Key Insight:** Window size minus max frequency is the number of replacements needed. Shrink window if it exceeds `k`.
```cpp
vector<int> counts(26, 0);
int l = 0, maxFreq = 0, maxLen = 0;
for (int r = 0; r < s.length(); ++r) {
    maxFreq = max(maxFreq, ++counts[s[r] - 'A']); // track max frequency in current window
    if ((r - l + 1) - maxFreq > k) { // if replacements needed > k, shrink window
        counts[s[l++] - 'A']--;
    }
    maxLen = max(maxLen, r - l + 1);
}
return maxLen;
```
**Complexity:** Time: O(N), Space: O(1)

### Minimum Window Substring (LC 76)
**Pattern:** Two Maps, Shrinkable Window
**Key Insight:** Use two frequency maps. Expand right until all required chars are in window, then shrink left to minimize length.
```cpp
vector<int> need(128, 0), have(128, 0);
for (char c : t) need[c]++;
int l = 0, required = t.length(), minLen = INT_MAX, minStart = 0;
for (int r = 0; r < s.length(); ++r) {
    if (++have[s[r]] <= need[s[r]]) required--;
    while (required == 0) { // shrink window while valid to find minimum
        if (r - l + 1 < minLen) {
            minLen = r - l + 1;
            minStart = l;
        }
        if (--have[s[l]] < need[s[l]]) required++;
        l++;
    }
}
return minLen == INT_MAX ? "" : s.substr(minStart, minLen);
```
**Complexity:** Time: O(N + M), Space: O(1)

### Permutation in String (LC 567)
**Pattern:** Fixed Sliding Window
**Key Insight:** Use a fixed-size sliding window of length `s1`. Compare character frequency arrays.
```cpp
if (s1.length() > s2.length()) return false;
vector<int> map1(26, 0), map2(26, 0);
for (int i = 0; i < s1.length(); ++i) {
    map1[s1[i] - 'a']++;
    map2[s2[i] - 'a']++;
}
for (int i = s1.length(); i < s2.length(); ++i) {
    if (map1 == map2) return true;
    map2[s2[i] - 'a']++; // add incoming char
    map2[s2[i - s1.length()] - 'a']--; // remove outgoing char from window
}
return map1 == map2;
```
**Complexity:** Time: O(N), Space: O(1)

### Sliding Window Maximum (LC 239)
**Pattern:** Monotonic Deque
**Key Insight:** Use a monotonic decreasing deque to store indices. The front always holds the max element for the current window.
```cpp
deque<int> dq; // stores indices
vector<int> res;
for (int i = 0; i < nums.size(); ++i) {
    if (!dq.empty() && dq.front() < i - k + 1) dq.pop_front(); // remove indices outside the window
    while (!dq.empty() && nums[dq.back()] < nums[i]) dq.pop_back(); // maintain monotonic decreasing order
    dq.push_back(i);
    if (i >= k - 1) res.push_back(nums[dq.front()]);
}
return res;
```
**Complexity:** Time: O(N), Space: O(K)

### Minimum Size Subarray Sum (LC 209)
**Pattern:** Dynamic Shrinkable Window
**Key Insight:** Expand right adding to sum. When sum >= target, record length and shrink from left to find smaller valid window.
```cpp
int l = 0, sum = 0, minLen = INT_MAX;
for (int r = 0; r < nums.size(); ++r) {
    sum += nums[r];
    while (sum >= target) {
        minLen = min(minLen, r - l + 1);
        sum -= nums[l++]; // shrink from left to find minimum valid length
    }
}
return minLen == INT_MAX ? 0 : minLen;
```
**Complexity:** Time: O(N), Space: O(1)

### Subarrays with K Different Integers (LC 992)
**Pattern:** atMost(K) - atMost(K-1) trick
**Key Insight:** Finding exactly K is hard. Instead, find `atMost(K) - atMost(K-1)`.
```cpp
auto atMost = [&](int k) {
    unordered_map<int, int> count;
    int res = 0, l = 0;
    for (int r = 0; r < nums.size(); ++r) {
        if (!count[nums[r]]++) k--;
        while (k < 0) {
            if (!--count[nums[l++]]) k++;
        }
        res += r - l + 1; // number of valid subarrays ending at r
    }
    return res;
};
return atMost(k) - atMost(k - 1);
```
**Complexity:** Time: O(N), Space: O(N)

### Fruit Into Baskets (LC 904)
**Pattern:** Longest subarray with <= 2 distinct elements
**Key Insight:** Longest subarray with at most 2 distinct elements. Shrink left when map size exceeds 2.
```cpp
unordered_map<int, int> count;
int l = 0, maxLen = 0;
for (int r = 0; r < fruits.size(); ++r) {
    count[fruits[r]]++;
    while (count.size() > 2) {
        if (--count[fruits[l]] == 0) count.erase(fruits[l]); // if count drops to 0, completely remove from map
        l++;
    }
    maxLen = max(maxLen, r - l + 1);
}
return maxLen;
```
**Complexity:** Time: O(N), Space: O(1)

### Maximum Number of Vowels in Substring (LC 1456)
**Pattern:** Fixed Sliding Window
**Key Insight:** Fixed window size `k`. When shifting, add the new char's vowel status and subtract the old char's status.
```cpp
auto isVowel = [](char c) { return c=='a'||c=='e'||c=='i'||c=='o'||c=='u'; };
int count = 0, maxCount = 0;
for (int i = 0; i < s.length(); ++i) {
    if (isVowel(s[i])) count++; // add incoming char if vowel
    if (i >= k && isVowel(s[i - k])) count--; // remove outgoing char if vowel
    maxCount = max(maxCount, count);
}
return maxCount;
```
**Complexity:** Time: O(N), Space: O(1)
