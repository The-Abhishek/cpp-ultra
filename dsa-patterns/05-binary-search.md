# 05 — Binary Search

## Basic Binary Search
### Binary Search (LC 704)
**Pattern:** Standard Binary Search
**Key Insight:** Halve search space based on mid comparison until pointers cross.
```cpp
int search(vector<int>& nums, int target) {
    int l = 0, r = nums.size() - 1;
    while (l <= r) {
        int m = l + (r - l) / 2; // prevent overflow
        if (nums[m] == target) return m;
        if (nums[m] < target) l = m + 1;
        else r = m - 1;
    }
    return -1;
}
```
**Complexity:** O(log N) Time | O(1) Space

### Search Insert Position (LC 35)
**Pattern:** Lower bound search
**Key Insight:** Normal binary search, but return `l` (left pointer) which perfectly aligns with the first element >= target.
```cpp
int searchInsert(vector<int>& nums, int target) {
    int l = 0, r = nums.size() - 1;
    while (l <= r) {
        int m = l + (r - l) / 2;
        if (nums[m] < target) l = m + 1;
        else r = m - 1;
    }
    return l; // l becomes the lower bound index
}
```
**Complexity:** O(log N) Time | O(1) Space

### Find First and Last Position (LC 34)
**Pattern:** Binary search for boundaries
**Key Insight:** Use `lower_bound` for first occurrence and `prev(upper_bound)` for the last occurrence.
```cpp
vector<int> searchRange(vector<int>& nums, int target) {
    auto l = lower_bound(nums.begin(), nums.end(), target);
    auto r = upper_bound(nums.begin(), nums.end(), target);
    if (l == nums.end() || *l != target) return {-1, -1};
    return {(int)distance(nums.begin(), l), (int)distance(nums.begin(), prev(r))};
}
// Custom implementation: use `searchInsert` logic twice.
```
**Complexity:** O(log N) Time | O(1) Space

### Search a 2D Matrix (LC 74)
**Pattern:** 2D mapping to 1D array
**Key Insight:** Treat 2D matrix as flattened 1D array using `mid / n` for row and `mid % n` for column.
```cpp
bool searchMatrix(vector<vector<int>>& matrix, int target) {
    if (matrix.empty()) return false;
    int m = matrix.size(), n = matrix[0].size();
    int l = 0, r = m * n - 1;
    while (l <= r) {
        int mid = l + (r - l) / 2;
        int val = matrix[mid / n][mid % n]; // 1D to 2D mapping
        if (val == target) return true;
        if (val < target) l = mid + 1;
        else r = mid - 1;
    }
    return false;
}
```
**Complexity:** O(log(M * N)) Time | O(1) Space

### Time Based Key-Value Store (LC 981)
**Pattern:** Binary search on sorted timestamps (Upper bound)
**Key Insight:** Since timestamps are strictly increasing, use binary search to find the largest timestamp <= target.
```cpp
unordered_map<string, vector<pair<int, string>>> m;
void set(string key, string value, int timestamp) { m[key].push_back({timestamp, value}); }
string get(string key, int timestamp) {
    if (!m.count(key)) return "";
    const auto& vec = m[key];
    int l = 0, r = vec.size() - 1;
    string res = "";
    while (l <= r) {
        int mid = l + (r - l) / 2;
        // valid candidate, but check right for larger valid timestamp
        if (vec[mid].first <= timestamp) { res = vec[mid].second; l = mid + 1; }
        else { r = mid - 1; }
    }
    return res;
}
```
**Complexity:** O(log N) Time for get | O(N) Space

## Rotated / Peak Arrays
### Find Minimum in Rotated Sorted Array (LC 153)
**Pattern:** Compare mid with right boundary
**Key Insight:** Compare mid to right bound; if mid > right, the drop is to the right. Else, min is at or before mid.
```cpp
int findMin(vector<int>& nums) {
    int l = 0, r = nums.size() - 1;
    while (l < r) {
        int m = l + (r - l) / 2;
        if (nums[m] > nums[r]) l = m + 1; // min must be in right unsorted half
        else r = m; // min is at mid or to the left
    }
    return nums[l];
}
```
**Complexity:** O(log N) Time | O(1) Space

### Search in Rotated Sorted Array (LC 33)
**Pattern:** Identify sorted half
**Key Insight:** One half is always sorted. Check if target lies within the sorted half's range to decide which way to step.
```cpp
int search(vector<int>& nums, int target) {
    int l = 0, r = nums.size() - 1;
    while (l <= r) {
        int m = l + (r - l) / 2;
        if (nums[m] == target) return m;
        // Left half is perfectly sorted
        if (nums[l] <= nums[m]) {
            if (target >= nums[l] && target < nums[m]) r = m - 1;
            else l = m + 1;
        } 
        // Right half is perfectly sorted
        else {
            if (target > nums[m] && target <= nums[r]) l = m + 1;
            else r = m - 1;
        }
    }
    return -1;
}
```
**Complexity:** O(log N) Time | O(1) Space

### Find Peak Element (LC 162)
**Pattern:** Follow the rising slope
**Key Insight:** Follow the rising slope: if `nums[m] < nums[m+1]`, a peak must exist to the right.
```cpp
int findPeakElement(vector<int>& nums) {
    int l = 0, r = nums.size() - 1;
    while (l < r) {
        int m = l + (r - l) / 2;
        if (nums[m] < nums[m + 1]) l = m + 1; // Rising slope, Peak is to the right
        else r = m; // Peak is at m or left
    }
    return l;
}
```
**Complexity:** O(log N) Time | O(1) Space

## Binary Search on Answer
### Koko Eating Bananas (LC 875)
**Pattern:** Binary search over solution space
**Key Insight:** Search the answer space `[1, max_pile]`. If a speed works, try slower; if it fails, go faster.
```cpp
int minEatingSpeed(vector<int>& piles, int h) {
    int l = 1, r = *max_element(piles.begin(), piles.end());
    while (l <= r) {
        int m = l + (r - l) / 2;
        long long hours = 0;
        for (int p : piles) hours += (p + m - 1) / m; // ceil(p/m)
        if (hours <= h) r = m - 1; // can eat slower
        else l = m + 1; // must eat faster
    }
    return l;
}
```
**Complexity:** O(N log M) Time | O(1) Space

### Capacity To Ship Packages (LC 1011)
**Pattern:** Binary search on answer (like Koko)
**Key Insight:** Search space is `[max_weight, sum(weights)]`. Validate if capacity `m` can group packages in `days`.
```cpp
int shipWithinDays(vector<int>& weights, int days) {
    int l = *max_element(weights.begin(), weights.end());
    int r = accumulate(weights.begin(), weights.end(), 0);
    while (l <= r) {
        int m = l + (r - l) / 2;
        int d = 1, curr = 0;
        for (int w : weights) {
            if (curr + w > m) { d++; curr = 0; }
            curr += w;
        }
        if (d <= days) r = m - 1;
        else l = m + 1;
    }
    return l;
}
```
**Complexity:** O(N log W) Time | O(1) Space

### Split Array Largest Sum (LC 410)
**Pattern:** Binary search on answer (identical to LC 1011)
**Key Insight:** Mathematically identical to shipping packages—find minimum max-chunk sum by binary searching the sum range.
```cpp
int splitArray(vector<int>& nums, int k) {
    int l = *max_element(nums.begin(), nums.end());
    int r = accumulate(nums.begin(), nums.end(), 0);
    while (l <= r) {
        int m = l + (r - l) / 2;
        int chunks = 1, curr = 0;
        for (int n : nums) {
            if (curr + n > m) { chunks++; curr = 0; }
            curr += n;
        }
        if (chunks <= k) r = m - 1;
        else l = m + 1;
    }
    return l;
}
```
**Complexity:** O(N log S) Time | O(1) Space

### Aggressive Cows / Magnetic Balls pattern (LC 1552)
**Pattern:** Maximize the minimum distance
**Key Insight:** Sort positions. Binary search the distance `d`; greedily place items `d` apart to check feasibility.
```cpp
int maxDistance(vector<int>& position, int m) {
    sort(position.begin(), position.end());
    int l = 1, r = position.back() - position.front();
    while (l <= r) {
        int mid = l + (r - l) / 2;
        int count = 1, last = position[0];
        for (int i = 1; i < position.size(); ++i) {
            if (position[i] - last >= mid) { count++; last = position[i]; }
        }
        if (count >= m) l = mid + 1; // Can place more spaced out
        else r = mid - 1;
    }
    return r; // r is the max valid distance
}
```
**Complexity:** O(N log D) Time | O(1) Space

## Binary Search on Partition
### Median of Two Sorted Arrays (LC 4)
**Pattern:** Binary search on smaller array to find valid partition
**Key Insight:** Binary search on the smaller array's partition index. A valid partition has max(left halves) <= min(right halves).
```cpp
double findMedianSortedArrays(vector<int>& nums1, vector<int>& nums2) {
    if (nums1.size() > nums2.size()) return findMedianSortedArrays(nums2, nums1);
    int m = nums1.size(), n = nums2.size();
    int l = 0, r = m, total = (m + n + 1) / 2;
    while (l <= r) {
        int i = l + (r - l) / 2, j = total - i;
        int l1 = (i > 0) ? nums1[i - 1] : INT_MIN;
        int r1 = (i < m) ? nums1[i] : INT_MAX;
        int l2 = (j > 0) ? nums2[j - 1] : INT_MIN;
        int r2 = (j < n) ? nums2[j] : INT_MAX;
        
        if (l1 <= r2 && l2 <= r1) { // correct partition found
            if ((m + n) % 2 == 0) return (max(l1, l2) + min(r1, r2)) / 2.0;
            return max(l1, l2);
        } else if (l1 > r2) {
            r = i - 1;
        } else {
            l = i + 1;
        }
    }
    return 0.0;
}
```
**Complexity:** O(log(min(M, N))) Time | O(1) Space
