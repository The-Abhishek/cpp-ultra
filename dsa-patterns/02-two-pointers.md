# 02 — Two Pointers

### Valid Palindrome (LC 125)
**Pattern:** Two Pointers from ends
**Key Insight:** Two pointers start at ends, skipping non-alphanumeric characters, closing inward until they meet.
```cpp
int l = 0, r = s.length() - 1;
while (l < r) {
    if (!isalnum(s[l])) l++; // skip non-alphanumeric
    else if (!isalnum(s[r])) r--;
    else if (tolower(s[l++]) != tolower(s[r--])) return false;
}
return true;
```
**Complexity:** Time: O(N), Space: O(1)

### Two Sum II Sorted (LC 167)
**Pattern:** Directional Two Pointers
**Key Insight:** Since array is sorted, if sum is too small, increment left pointer; if too large, decrement right pointer.
```cpp
int l = 0, r = numbers.size() - 1;
while (l < r) {
    int sum = numbers[l] + numbers[r];
    if (sum == target) return {l + 1, r + 1};
    if (sum < target) l++;
    else r--; // shrink from right if sum too large
}
return {};
```
**Complexity:** Time: O(N), Space: O(1)

### 3Sum (LC 15)
**Pattern:** Sort + Two Pointers with Deduplication
**Key Insight:** Sort first. Fix one element, then use Two Sum II approach for the remaining two. Skip duplicates to avoid duplicate triplets.
```cpp
vector<vector<int>> res;
sort(nums.begin(), nums.end());
for (int i = 0; i < nums.size(); ++i) {
    if (i > 0 && nums[i] == nums[i-1]) continue; // skip duplicate fixed elements
    int l = i + 1, r = nums.size() - 1;
    while (l < r) {
        int sum = nums[i] + nums[l] + nums[r];
        if (sum > 0) r--;
        else if (sum < 0) l++;
        else {
            res.push_back({nums[i], nums[l], nums[r]});
            l++;
            while (l < r && nums[l] == nums[l-1]) l++; // skip duplicate second elements
        }
    }
}
return res;
```
**Complexity:** Time: O(N^2), Space: O(1) or O(N) for sort

### Container With Most Water (LC 11)
**Pattern:** Greedy Two Pointers closing inward
**Key Insight:** Start with widest container. Shrink width by moving the pointer pointing to the shorter line, hoping for a taller line.
```cpp
int maxArea = 0, l = 0, r = height.size() - 1;
while (l < r) {
    int area = min(height[l], height[r]) * (r - l);
    maxArea = max(maxArea, area);
    if (height[l] < height[r]) l++; // move the shorter height inward
    else r--;
}
return maxArea;
```
**Complexity:** Time: O(N), Space: O(1)

### Trapping Rain Water (LC 42)
**Pattern:** Two Pointers with max tracking
**Key Insight:** Track max heights from left and right. The bottleneck is the smaller of the two max heights, so process that side.
```cpp
int l = 0, r = height.size() - 1;
int maxL = 0, maxR = 0, trapped = 0;
while (l <= r) {
    if (maxL <= maxR) { // process the side with the smaller max bound
        maxL = max(maxL, height[l]);
        trapped += maxL - height[l++];
    } else {
        maxR = max(maxR, height[r]);
        trapped += maxR - height[r--];
    }
}
return trapped;
```
**Complexity:** Time: O(N), Space: O(1)

### Remove Duplicates from Sorted Array (LC 26)
**Pattern:** Slow/Fast Pointers
**Key Insight:** Use a slow pointer to track the position of the next unique element, and a fast pointer to scan.
```cpp
if (nums.empty()) return 0;
int l = 1;
for (int r = 1; r < nums.size(); ++r) {
    if (nums[r] != nums[r - 1]) {
        nums[l++] = nums[r]; // write next unique element at slow pointer
    }
}
return l;
```
**Complexity:** Time: O(N), Space: O(1)

### Move Zeroes (LC 283)
**Pattern:** Partitioning Two Pointers
**Key Insight:** Fast pointer scans array; slow pointer tracks where the next non-zero should go. Swap them.
```cpp
int l = 0;
for (int r = 0; r < nums.size(); ++r) {
    if (nums[r] != 0) {
        swap(nums[l++], nums[r]); // swap non-zero to the front partition
    }
}
```
**Complexity:** Time: O(N), Space: O(1)

### Sort Colors / Dutch National Flag (LC 75)
**Pattern:** 3-Way Partitioning Pointers
**Key Insight:** 3-way partition (Dutch National Flag). 0s go to left, 2s go to right, 1s stay in middle.
```cpp
int l = 0, curr = 0, r = nums.size() - 1;
while (curr <= r) {
    if (nums[curr] == 0) {
        swap(nums[l++], nums[curr++]); // swap 0 to left partition
    } else if (nums[curr] == 2) {
        swap(nums[curr], nums[r--]); // swap 2 to right partition, don't increment curr
    } else {
        curr++;
    }
}
```
**Complexity:** Time: O(N), Space: O(1)

### Next Permutation (LC 31)
**Pattern:** Suffix Scanning Pointers
**Key Insight:** Find first decreasing element from right. Swap it with next larger element to its right, then reverse the rest.
```cpp
int i = nums.size() - 2;
while (i >= 0 && nums[i] >= nums[i + 1]) i--; // find first dip from right
if (i >= 0) {
    int j = nums.size() - 1;
    while (nums[j] <= nums[i]) j--;
    swap(nums[i], nums[j]);
}
reverse(nums.begin() + i + 1, nums.end()); // reverse suffix to make it the smallest possible
```
**Complexity:** Time: O(N), Space: O(1)

### 4Sum (LC 18)
**Pattern:** K-Sum Generic (2 loops + 2 pointers)
**Key Insight:** Generalization of 3Sum. Use two loops to fix two elements, then two pointers for the rest. Skip duplicates.
```cpp
vector<vector<int>> res;
sort(nums.begin(), nums.end());
for(int i = 0; i < nums.size(); ++i) {
    if (i > 0 && nums[i] == nums[i-1]) continue;
    for (int j = i + 1; j < nums.size(); ++j) {
        if (j > i + 1 && nums[j] == nums[j-1]) continue;
        int l = j + 1, r = nums.size() - 1;
        while (l < r) {
            long sum = (long)nums[i] + nums[j] + nums[l] + nums[r]; // cast to long to prevent overflow
            if (sum > target) r--;
            else if (sum < target) l++;
            else {
                res.push_back({nums[i], nums[j], nums[l], nums[r]});
                l++;
                while (l < r && nums[l] == nums[l-1]) l++; // skip duplicate elements
            }
        }
    }
}
return res;
```
**Complexity:** Time: O(N^3), Space: O(1) or O(N) for sort
