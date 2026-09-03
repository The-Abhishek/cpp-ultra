# 04 — Stack & Monotonic Stack

## Basic Stack
### Valid Parentheses (LC 20)
**Pattern:** Stack for matching pairs
**Key Insight:** Push opening brackets; on closing, top must match. Empty stack at end = valid. O(1) lookup via direct char comparison.
```cpp
bool isValid(string s) {
    stack<char> st;
    for (char c : s) {
        if (c == '(') st.push(')');
        else if (c == '{') st.push('}');
        else if (c == '[') st.push(']');
        else if (st.empty() || st.top() != c) return false; // top must match closing
        else st.pop();
    }
    return st.empty();
}
```
**Complexity:** O(N) Time | O(N) Space

### Min Stack (LC 155)
**Pattern:** Pair stack for running minimum
**Key Insight:** Store the running minimum alongside each value in the stack to allow O(1) retrieval even after pops.
```cpp
stack<pair<int, int>> st; // {val, min_val}
void push(int val) {
    // current_min tracks the minimum element so far at this depth
    int current_min = st.empty() ? val : min(val, st.top().second);
    st.push({val, current_min});
}
int getMin() { return st.top().second; }
```
**Complexity:** O(1) Time | O(N) Space

### Evaluate Reverse Polish Notation (LC 150)
**Pattern:** Stack for evaluating postfix expressions
**Key Insight:** Push operands; on operator, pop last two, evaluate, and push result back.
```cpp
int evalRPN(vector<string>& tokens) {
    stack<long> st;
    for (const string& t : tokens) {
        if (t == "+" || t == "-" || t == "*" || t == "/") {
            long b = st.top(); st.pop(); // right operand popped first
            long a = st.top(); st.pop(); // left operand popped second
            if (t == "+") st.push(a + b);
            else if (t == "-") st.push(a - b);
            else if (t == "*") st.push(a * b);
            else st.push(a / b);
        } else {
            st.push(stoi(t));
        }
    }
    return st.top();
}
```
**Complexity:** O(N) Time | O(N) Space

### Asteroid Collision (LC 735)
**Pattern:** Stack for state tracking & cancellation
**Key Insight:** Collisions only happen when top is moving right ( > 0) and new is moving left ( < 0). Handle ties by destroying both.
```cpp
vector<int> asteroidCollision(vector<int>& asteroids) {
    vector<int> st; // vector as stack
    for (int a : asteroids) {
        bool destroyed = false;
        // while collision is possible (right-moving top, left-moving new)
        while (!st.empty() && a < 0 && st.back() > 0) {
            if (st.back() < -a) st.pop_back(); // top asteroid explodes
            else if (st.back() == -a) { st.pop_back(); destroyed = true; break; } // both explode
            else { destroyed = true; break; } // new asteroid explodes
        }
        if (!destroyed) st.push_back(a);
    }
    return st;
}
```
**Complexity:** O(N) Time | O(N) Space

### Basic Calculator II (LC 227)
**Pattern:** Stack for deferred evaluation
**Key Insight:** Delay addition/subtraction. Push term products/quotients to stack, then sum the stack at the end.
```cpp
int calculate(string s) {
    stack<int> st;
    char op = '+';
    int curr = 0;
    for (int i = 0; i <= s.length(); ++i) {
        if (i < s.length() && isdigit(s[i])) curr = curr * 10 + (s[i] - '0');
        if (i == s.length() || (!isdigit(s[i]) && s[i] != ' ')) {
            // evaluate * and / immediately, delay + and -
            if (op == '+') st.push(curr);
            else if (op == '-') st.push(-curr);
            else if (op == '*') { int top = st.top(); st.pop(); st.push(top * curr); }
            else if (op == '/') { int top = st.top(); st.pop(); st.push(top / curr); }
            op = s[i]; curr = 0;
        }
    }
    int res = 0;
    while (!st.empty()) { res += st.top(); st.pop(); }
    return res;
}
```
**Complexity:** O(N) Time | O(N) Space

### Decode String (LC 394)
**Pattern:** Stack for nested structures
**Key Insight:** Push current string and multiplier on `[`; on `]`, pop them and multiply inner string, then append to outer.
```cpp
string decodeString(string s) {
    stack<int> counts;
    stack<string> strings;
    string res = ""; int k = 0;
    for (char c : s) {
        if (isdigit(c)) k = k * 10 + (c - '0'); // accumulate multiplier
        else if (isalpha(c)) res += c;
        else if (c == '[') { 
            // save state before entering inner brackets
            counts.push(k); strings.push(res); res = ""; k = 0; 
        }
        else if (c == ']') {
            string tmp = res; res = strings.top(); strings.pop();
            for (int i = counts.top(); i > 0; --i) res += tmp;
            counts.pop();
        }
    }
    return res;
}
```
**Complexity:** O(N) Time | O(N) Space

## Monotonic Stack
### Daily Temperatures (LC 739)
**Pattern:** Monotonic decreasing stack (find next greater)
**Key Insight:** Keep indices of unresolved days in a decreasing stack; pop when finding a warmer day to resolve span.
```cpp
vector<int> dailyTemperatures(vector<int>& temperatures) {
    int n = temperatures.size();
    vector<int> res(n, 0);
    stack<int> st; // stores indices
    for (int i = 0; i < n; ++i) {
        // resolve previous colder days
        while (!st.empty() && temperatures[i] > temperatures[st.top()]) {
            res[st.top()] = i - st.top();
            st.pop();
        }
        st.push(i);
    }
    return res;
}
```
**Complexity:** O(N) Time | O(N) Space

### Next Greater Element I (LC 496)
**Pattern:** Monotonic decreasing stack + Hash Map
**Key Insight:** Map next greater element for all `nums2` using monotonic stack, then map answers for `nums1`.
```cpp
vector<int> nextGreaterElement(vector<int>& nums1, vector<int>& nums2) {
    unordered_map<int, int> next_greater;
    stack<int> st;
    for (int num : nums2) {
        while (!st.empty() && num > st.top()) {
            next_greater[st.top()] = num;
            st.pop();
        }
        st.push(num);
    }
    vector<int> res;
    for (int num : nums1) res.push_back(next_greater.count(num) ? next_greater[num] : -1);
    return res;
}
```
**Complexity:** O(N + M) Time | O(N) Space

### Next Greater Element II (LC 503)
**Pattern:** Monotonic decreasing stack + Circular array (iterate 2n)
**Key Insight:** Iterate `2n` times using modulo to simulate a circular array, maintaining a decreasing stack.
```cpp
vector<int> nextGreaterElements(vector<int>& nums) {
    int n = nums.size();
    vector<int> res(n, -1);
    stack<int> st;
    for (int i = 0; i < 2 * n; ++i) {
        // simulate circular array via modulo
        while (!st.empty() && nums[i % n] > nums[st.top()]) {
            res[st.top()] = nums[i % n];
            st.pop();
        }
        st.push(i % n);
    }
    return res;
}
```
**Complexity:** O(N) Time | O(N) Space

### Largest Rectangle in Histogram (LC 84)
**Pattern:** Monotonic increasing stack (find next/prev smaller)
**Key Insight:** Push indices of increasing heights. On drop, height of top forms a rectangle with width extending back to new top.
```cpp
int largestRectangleArea(vector<int>& heights) {
    heights.push_back(0); // Dummy for remaining elements
    stack<int> st;
    int max_area = 0;
    for (int i = 0; i < heights.size(); ++i) {
        while (!st.empty() && heights[i] < heights[st.top()]) {
            int h = heights[st.top()]; st.pop();
            // width = current index (i) - new top index - 1
            int w = st.empty() ? i : i - st.top() - 1;
            max_area = max(max_area, h * w);
        }
        st.push(i);
    }
    return max_area;
}
```
**Complexity:** O(N) Time | O(N) Space

### Maximal Rectangle (LC 85)
**Pattern:** DP array + Largest Rectangle in Histogram
**Key Insight:** Treat each row as a histogram base, accumulating heights of 1s, and apply largest rectangle logic per row.
```cpp
int maximalRectangle(vector<vector<char>>& matrix) {
    if (matrix.empty()) return 0;
    int n = matrix[0].size(), max_area = 0;
    vector<int> heights(n + 1, 0); // extra 0 for histogram
    for (const auto& row : matrix) {
        for (int i = 0; i < n; ++i) {
            // height drops to 0 if '0', else accumulates
            heights[i] = (row[i] == '1') ? heights[i] + 1 : 0;
        }
        stack<int> st;
        for (int i = 0; i <= n; ++i) {
            while (!st.empty() && heights[i] < heights[st.top()]) {
                int h = heights[st.top()]; st.pop();
                int w = st.empty() ? i : i - st.top() - 1;
                max_area = max(max_area, h * w);
            }
            st.push(i);
        }
    }
    return max_area;
}
```
**Complexity:** O(R * C) Time | O(C) Space

### Car Fleet (LC 853)
**Pattern:** Monotonic Stack on sorting/math
**Key Insight:** Sort by position descending. A fleet merges if a car's time to target is <= the fleet ahead's time.
```cpp
int carFleet(int target, vector<int>& position, vector<int>& speed) {
    vector<pair<int, double>> cars;
    for (int i = 0; i < position.size(); ++i) 
        cars.push_back({position[i], (double)(target - position[i]) / speed[i]});
    sort(cars.rbegin(), cars.rend()); // sort descending by position
    int fleets = 0;
    double max_time = 0;
    for (auto& car : cars) {
        // time = distance / speed
        if (car.second > max_time) { // slower car ahead, forms new fleet
            max_time = car.second;
            fleets++;
        }
    }
    return fleets;
}
```
**Complexity:** O(N log N) Time | O(N) Space

### Remove K Digits (LC 402)
**Pattern:** Monotonic increasing stack (greedy digit removal)
**Key Insight:** Greedily drop earlier larger digits to keep the number small, ensuring MSB is minimized.
```cpp
string removeKdigits(string num, int k) {
    string res = "";
    for (char c : num) {
        // drop larger previous digits while we can
        while (res.length() && res.back() > c && k > 0) { res.pop_back(); k--; }
        if (res.length() || c != '0') res.push_back(c); // avoid leading zero
    }
    while (res.length() && k-- > 0) res.pop_back();
    return res.empty() ? "0" : res;
}
```
**Complexity:** O(N) Time | O(N) Space

### Online Stock Span (LC 901)
**Pattern:** Monotonic decreasing stack with frequency/weight
**Key Insight:** Merge smaller/equal previous prices into current price, accumulating their spans in the stack.
```cpp
stack<pair<int, int>> st; // {price, span}
int next(int price) {
    int span = 1;
    // aggregate span of contiguous smaller/equal prices
    while (!st.empty() && st.top().first <= price) {
        span += st.top().second;
        st.pop();
    }
    st.push({price, span});
    return span;
}
```
**Complexity:** O(1) Amortized Time | O(N) Space
