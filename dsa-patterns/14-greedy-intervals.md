# 14 — Greedy & Intervals

## Arrays & Subarrays

### Maximum Subarray / Kadane's (LC 53)
Pattern: Local vs Global maximum
**Key Insight:** If the running sum becomes negative, it hurts future sums, so reset it to the current element.
```cpp
int maxSubArray(vector<int>& nums) {
    int curMax = 0, globalMax = INT_MIN;
    for (int x : nums) {
        curMax = max(x, curMax + x); // extend subarray or start new one
        globalMax = max(globalMax, curMax);
    }
    return globalMax;
}
```
**Complexity:** O(N) Time | O(1) Space

### Gas Station (LC 134)
Pattern: Start reset based on cumulative deficit
**Key Insight:** If you can't reach station `B` from `A`, no station between `A` and `B` can reach `B` either. Set start to `B+1`.
```cpp
int canCompleteCircuit(vector<int>& gas, vector<int>& cost) {
    int total = 0, cur = 0, start = 0;
    for (int i = 0; i < gas.size(); ++i) {
        total += gas[i] - cost[i];
        cur += gas[i] - cost[i];
        if (cur < 0) { // unable to reach current station
            start = i + 1; // start from next station
            cur = 0;
        }
    }
    return total >= 0 ? start : -1;
}
```
**Complexity:** O(N) Time | O(1) Space

### Hand of Straights (LC 846)
Pattern: Ordered map counting
**Key Insight:** Always try to build a group starting from the smallest available card. Use an ordered map to fetch the smallest automatically.
```cpp
bool isNStraightHand(vector<int>& hand, int groupSize) {
    if (hand.size() % groupSize != 0) return false;
    map<int, int> count; // ordered map to process smallest cards first
    for (int card : hand) count[card]++;
    for (auto [card, freq] : count) {
        if (freq == 0) continue;
        for (int i = 0; i < groupSize; ++i) {
            if (count[card + i] < freq) return false; // missing consecutive card
            count[card + i] -= freq;
        }
    }
    return true;
}
```
**Complexity:** O(N log N) Time | O(N) Space

### Partition Labels (LC 763)
Pattern: Last seen index
**Key Insight:** A partition must include the last occurrence of every character within it. We can track the running maximum `last seen` index.
```cpp
vector<int> partitionLabels(string s) {
    vector<int> last(26, 0), res;
    for (int i = 0; i < s.size(); ++i) last[s[i] - 'a'] = i; // record last occurrence
    int curEnd = 0, curStart = 0;
    for (int i = 0; i < s.size(); ++i) {
        curEnd = max(curEnd, last[s[i] - 'a']); // extend boundary if needed
        if (i == curEnd) { // reached the boundary for this partition
            res.push_back(curEnd - curStart + 1);
            curStart = i + 1;
        }
    }
    return res;
}
```
**Complexity:** O(N) Time | O(1) Space

### Valid Parenthesis String (LC 678)
Pattern: Track valid ranges of open brackets
**Key Insight:** Track the `min` and `max` possible open brackets. If `max` becomes negative, it's invalid. Cap `min` at 0.
```cpp
bool checkValidString(string s) {
    int cMin = 0, cMax = 0;
    for (char c : s) {
        if (c == '(') cMin++, cMax++;
        else if (c == ')') cMin--, cMax--;
        else cMin--, cMax++; // '*' acts as ')' or '('
        
        if (cMax < 0) return false; // too many closing brackets
        cMin = max(0, cMin); // we can't have negative open brackets
    }
    return cMin == 0;
}
```
**Complexity:** O(N) Time | O(1) Space

## Intervals

### Merge Intervals (LC 56)
Pattern: Sort by start time, merge overlapping
**Key Insight:** Sorting by start time ensures overlapping intervals are adjacent. Merge if current start <= previous end.
```cpp
vector<vector<int>> merge(vector<vector<int>>& intervals) {
    sort(intervals.begin(), intervals.end()); // sort by start time
    vector<vector<int>> res = {intervals[0]};
    for (auto& iv : intervals) {
        if (iv[0] <= res.back()[1]) res.back()[1] = max(res.back()[1], iv[1]); // merge
        else res.push_back(iv);
    }
    return res;
}
```
**Complexity:** O(N log N) Time | O(N) Space

### Insert Interval (LC 57)
Pattern: Left, Overlapping, Right loops
**Key Insight:** Process intervals in 3 phases: non-overlapping before, merging overlapping, and non-overlapping after.
```cpp
vector<vector<int>> insert(vector<vector<int>>& intervals, vector<int>& newInterval) {
    vector<vector<int>> res;
    int i = 0, n = intervals.size();
    while (i < n && intervals[i][1] < newInterval[0]) res.push_back(intervals[i++]); // Left
    while (i < n && intervals[i][0] <= newInterval[1]) { // Overlapping
        newInterval[0] = min(newInterval[0], intervals[i][0]);
        newInterval[1] = max(newInterval[1], intervals[i][1]);
        i++;
    }
    res.push_back(newInterval);
    while (i < n) res.push_back(intervals[i++]); // Right
    return res;
}
```
**Complexity:** O(N) Time | O(N) Space

### Non-Overlapping Intervals (LC 435) / Minimum Number of Arrows (LC 452) / Activity Selection Pattern
Pattern: Sort by END time, count non-overlapping
**Key Insight:** Sorting by end time leaves maximum room for subsequent intervals, minimizing overlaps greedily.
```cpp
int eraseOverlapIntervals(vector<vector<int>>& intervals) {
    sort(intervals.begin(), intervals.end(), [](auto& a, auto& b){ return a[1] < b[1]; }); // sort by end time
    int count = 0, end = INT_MIN;
    for (auto& iv : intervals) {
        if (iv[0] >= end) end = iv[1]; // non-overlapping, keep it
        else count++; // overlapping, need to remove
    }
    return count;
}
```
**Complexity:** O(N log N) Time | O(1) Space

### Meeting Rooms (LC 252)
Pattern: Check overlapping consecutive
**Key Insight:** If you sort by start time, conflicts only happen if a meeting starts before the previous one ends.
```cpp
bool canAttendMeetings(vector<vector<int>>& intervals) {
    sort(intervals.begin(), intervals.end()); // sort by start time
    for (int i = 1; i < intervals.size(); ++i) {
        if (intervals[i][0] < intervals[i-1][1]) return false; // overlap found
    }
    return true;
}
```
**Complexity:** O(N log N) Time | O(1) Space

### Meeting Rooms II (LC 253)
Pattern: Sweep line OR Min-heap tracking end times
**Key Insight:** Sweep line treats start times as `+1` room and end times as `-1` room. Track the maximum prefix sum.
```cpp
// Sweep Line Pattern
int minMeetingRooms(vector<vector<int>>& intervals) {
    map<int, int> sweep; // maps time to room delta
    for (auto& iv : intervals) {
        sweep[iv[0]]++; // +1 room needed
        sweep[iv[1]]--; // -1 room freed
    }
    int maxRooms = 0, curRooms = 0;
    for (auto& [time, delta] : sweep) {
        curRooms += delta;
        maxRooms = max(maxRooms, curRooms); // track max concurrency
    }
    return maxRooms;
}
```
**Complexity:** O(N log N) Time | O(N) Space

## Miscellaneous Greedy

### Job Scheduling with Deadlines Pattern
Pattern: Sort by profit desc, place in latest available slot
**Key Insight:** Always try to schedule the most profitable jobs first, and push them as late as possible before their deadline to save earlier slots.
```cpp
struct Job { int id, dead, profit; };
int jobScheduling(vector<Job>& jobs) {
    sort(jobs.begin(), jobs.end(), [](Job& a, Job& b) { return a.profit > b.profit; });
    int maxDead = 0;
    for (auto& j : jobs) maxDead = max(maxDead, j.dead);
    
    vector<int> slots(maxDead + 1, -1); // keeps track of used slots
    int totalProfit = 0;
    for (auto& j : jobs) {
        for (int i = j.dead; i > 0; --i) { // find latest available slot
            if (slots[i] == -1) {
                slots[i] = j.id;
                totalProfit += j.profit;
                break;
            }
        }
    }
    return totalProfit;
}
```
**Complexity:** O(N * maxDead) Time | O(maxDead) Space
