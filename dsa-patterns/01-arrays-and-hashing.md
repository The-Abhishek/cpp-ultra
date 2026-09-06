# 01 — Arrays & Hashing

### Two Sum (LC 1)
**Pattern:** One-pass Hash Map
**Key Insight:** For each element, check if its complement (target - num) was already seen. Single pass suffices since one of the pair will be seen second.
```cpp
unordered_map<int, int> numMap;
for (int i = 0; i < nums.size(); ++i) {
    if (numMap.count(target - nums[i])) // complement found
        return {numMap[target - nums[i]], i};
    numMap[nums[i]] = i;
}
return {};
```
**Complexity:** Time: O(N), Space: O(N)

### Contains Duplicate (LC 217)
**Pattern:** Hash Set tracking
**Key Insight:** Hash set stores seen elements. If an element cannot be inserted, it's a duplicate.
```cpp
unordered_set<int> seen;
for (int num : nums) {
    if (!seen.insert(num).second) // insert returns pair, .second is false if element exists
        return true;
}
return false;
```
**Complexity:** Time: O(N), Space: O(N)

### Valid Anagram (LC 242)
**Pattern:** Character Frequency Array
**Key Insight:** Use a fixed-size array to count character frequencies. Increment for one string, decrement for the other.
```cpp
if (s.length() != t.length()) return false;
vector<int> counts(26, 0);
for (int i = 0; i < s.length(); ++i) {
    counts[s[i] - 'a']++;
    counts[t[i] - 'a']--;
}
for (int c : counts) // check if all counts balanced to zero
    if (c != 0) return false;
return true;
```
**Complexity:** Time: O(N), Space: O(1)

### Group Anagrams (LC 49)
**Pattern:** Hash Map with sorted string key (or char count key)
**Key Insight:** Anagrams map to the same sorted string. Use the sorted string as the hash map key.
```cpp
unordered_map<string, vector<string>> groups;
for (const string& s : strs) {
    string key = s;
    sort(key.begin(), key.end()); // sort to create a canonical key
    groups[key].push_back(s);
}
vector<vector<string>> res;
for (auto& pair : groups)
    res.push_back(move(pair.second));
return res;
```
**Complexity:** Time: O(N * K log K), Space: O(N * K)

### Top K Frequent Elements (LC 347)
**Pattern:** Bucket Sort for frequencies
**Key Insight:** Count frequencies, then bucket sort where the index is the frequency. Iterate buckets backwards.
```cpp
unordered_map<int, int> counts;
for (int num : nums) counts[num]++;
vector<vector<int>> buckets(nums.size() + 1);
for (auto& [num, freq] : counts) 
    buckets[freq].push_back(num); // map frequency to list of numbers
vector<int> res;
for (int i = buckets.size() - 1; i >= 0 && res.size() < k; --i) { // iterate from highest frequency down
    for (int num : buckets[i]) {
        res.push_back(num);
        if (res.size() == k) break;
    }
}
return res;
```
**Complexity:** Time: O(N), Space: O(N)

### Product of Array Except Self (LC 238)
**Pattern:** Prefix and Suffix products (in place)
**Key Insight:** Multiply all elements to the left (prefix) and all elements to the right (suffix) without division.
```cpp
vector<int> res(nums.size(), 1);
int prefix = 1, suffix = 1;
for (int i = 0; i < nums.size(); ++i) {
    res[i] = prefix; // running prefix product
    prefix *= nums[i];
}
for (int i = nums.size() - 1; i >= 0; --i) {
    res[i] *= suffix; // multiply with running suffix product
    suffix *= nums[i];
}
return res;
```
**Complexity:** Time: O(N), Space: O(1) (excluding output array)

### Encode and Decode Strings (LC 271)
**Pattern:** Length prefix + delimiter
**Key Insight:** Prefix each string with its length and a delimiter (like #) so we know exactly how many characters to read.
```cpp
// Encode
string res = "";
for (const string& s : strs)
    res += to_string(s.length()) + "#" + s; // append length + delimiter + string
return res;

// Decode
vector<string> res;
int i = 0;
while (i < s.length()) {
    int j = s.find('#', i);
    int len = stoi(s.substr(i, j - i)); // read length before delimiter
    res.push_back(s.substr(j + 1, len));
    i = j + 1 + len;
}
return res;
```
**Complexity:** Time: O(N), Space: O(N)

### Longest Consecutive Sequence (LC 128)
**Pattern:** Hash Set interval building
**Key Insight:** Only start counting a sequence if the current number is the actual start (i.e., `num - 1` doesn't exist).
```cpp
unordered_set<int> numSet(nums.begin(), nums.end());
int longest = 0;
for (int num : numSet) {
    if (!numSet.count(num - 1)) { // only start if it's the beginning of a sequence
        int curr = num, len = 1;
        while (numSet.count(curr + 1)) {
            curr++; len++;
        }
        longest = max(longest, len);
    }
}
return longest;
```
**Complexity:** Time: O(N), Space: O(N)

### Valid Sudoku (LC 36)
**Pattern:** Coordinate mapping to blocks
**Key Insight:** Use 2D arrays to track seen numbers in rows, cols, and 3x3 blocks. Block index is `(r/3)*3 + c/3`.
```cpp
int rows[9][9] = {0}, cols[9][9] = {0}, blocks[9][9] = {0};
for(int r = 0; r < 9; ++r) {
    for(int c = 0; c < 9; ++c) {
        if(board[r][c] != '.') {
            int num = board[r][c] - '1';
            int b = (r / 3) * 3 + c / 3; // formula for 3x3 block index
            if(rows[r][num]++ || cols[c][num]++ || blocks[b][num]++)
                return false;
        }
    }
}
return true;
```
**Complexity:** Time: O(1), Space: O(1)

### Subarray Sum Equals K (LC 560)
**Pattern:** Prefix Sum Map
**Key Insight:** If `current_sum - k` exists in the prefix sum map, a subarray ending here sums to `k`.
```cpp
unordered_map<int, int> prefixCounts;
prefixCounts[0] = 1;
int sum = 0, count = 0;
for (int num : nums) {
    sum += num;
    if (prefixCounts.count(sum - k)) // check if required prefix sum occurred
        count += prefixCounts[sum - k];
    prefixCounts[sum]++;
}
return count;
```
**Complexity:** Time: O(N), Space: O(N)

### First Missing Positive (LC 41)
**Pattern:** Cyclic Sort / Index as Hash Key
**Key Insight:** Place each number `x` at index `x-1` (cyclic sort). The first index `i` not containing `i+1` is the missing positive.
```cpp
int n = nums.size();
for (int i = 0; i < n; ++i) {
    while (nums[i] > 0 && nums[i] <= n && nums[nums[i] - 1] != nums[i])
        swap(nums[i], nums[nums[i] - 1]); // swap num to its correct 0-based index if valid
}
for (int i = 0; i < n; ++i) {
    if (nums[i] != i + 1)
        return i + 1;
}
return n + 1;
```
**Complexity:** Time: O(N), Space: O(1)

### Majority Element (LC 169)
**Pattern:** Boyer-Moore Voting Algorithm
**Key Insight:** Boyer-Moore voting cancels out different elements. The majority element always survives.
```cpp
int count = 0, candidate = 0;
for (int num : nums) {
    if (count == 0) candidate = num; // reset candidate if count reaches 0
    count += (num == candidate) ? 1 : -1;
}
return candidate;
```
**Complexity:** Time: O(N), Space: O(1)

### Insert Delete GetRandom O(1) (LC 380)
**Pattern:** Hash Map mapping to Array Index
**Key Insight:** Array allows O(1) random access. Map allows O(1) lookups. On delete, swap the element with the last element in the array and pop.
```cpp
vector<int> vals;
unordered_map<int, int> valToIndex;
bool insert(int val) {
    if (valToIndex.count(val)) return false;
    vals.push_back(val);
    valToIndex[val] = vals.size() - 1;
    return true;
}
bool remove(int val) {
    if (!valToIndex.count(val)) return false;
    int lastVal = vals.back();
    int idx = valToIndex[val];
    vals[idx] = lastVal; // swap with last element to enable O(1) pop
    valToIndex[lastVal] = idx;
    vals.pop_back();
    valToIndex.erase(val);
    return true;
}
int getRandom() { return vals[rand() % vals.size()]; }
```
**Complexity:** Time: O(1) avg per operation, Space: O(N)
