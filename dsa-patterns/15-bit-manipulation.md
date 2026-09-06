# 15 — Bit Manipulation

## Essential Bit Tricks
```cpp
n & (n - 1);           // Clear lowest set bit
n & (-n);              // Isolate lowest set bit
x ^ x = 0; x ^ 0 = x;  // XOR properties
n & (n - 1) == 0;      // Check if power of 2 (for n > 0)
__builtin_popcount(n); // Count set bits (popcountll for long long)
n ^ (1 << i);          // Toggle i-th bit
n | (1 << i);          // Set i-th bit
n & ~(1 << i);         // Clear i-th bit
```

## Single Number (LC 136)
**Pattern:** XOR property
**Key Insight:** XORing a number by itself results in 0, so all paired numbers cancel out, leaving only the single number.
```cpp
int singleNumber(vector<int>& nums) {
    int res = 0;
    for (int n : nums) res ^= n; // Pairs cancel out to 0 (x ^ x = 0)
    return res;
}
```
**Complexity:** O(N) Time | O(1) Space

## Number of 1 Bits (LC 191)
**Pattern:** Clear lowest set bit
**Key Insight:** `n & (n - 1)` always flips the least significant 1-bit to 0, bounding iterations by the number of set bits instead of total bits.
```cpp
int hammingWeight(uint32_t n) {
    int count = 0;
    while (n) {
        n &= (n - 1); // Drops the lowest 1-bit
        count++;
    }
    return count;
}
```
**Complexity:** O(1) Time | O(1) Space

## Counting Bits (LC 338)
**Pattern:** DP with bits
**Key Insight:** The number of set bits in `i` is the same as `i >> 1` plus 1 if the lowest bit was set.
```cpp
vector<int> countBits(int n) {
    vector<int> dp(n + 1, 0);
    for (int i = 1; i <= n; i++) {
        dp[i] = dp[i >> 1] + (i & 1); // Right shift removes lowest bit; (i&1) checks if it was 1
    }
    return dp;
}
```
**Complexity:** O(N) Time | O(1) Extra Space

## Reverse Bits (LC 190)
**Pattern:** Bit shifting
**Key Insight:** Extract the lowest bit of `n` and push it into the result, then shift `n` right and result left.
```cpp
uint32_t reverseBits(uint32_t n) {
    uint32_t res = 0;
    for (int i = 0; i < 32; i++) {
        res = (res << 1) | (n & 1); // Extract lowest bit and append to res
        n >>= 1;
    }
    return res;
}
```
**Complexity:** O(1) Time | O(1) Space

## Missing Number (LC 268)
**Pattern:** XOR
**Key Insight:** XORing all indices and all array values together cancels out all present numbers, leaving only the missing index.
```cpp
int missingNumber(vector<int>& nums) {
    int res = nums.size();
    for (int i = 0; i < nums.size(); i++) {
        res ^= i ^ nums[i]; // XOR pairs index and value to cancel matches
    }
    return res;
}
```
**Complexity:** O(N) Time | O(1) Space

## Sum of Two Integers (LC 371)
**Pattern:** XOR for sum, AND for carry
**Key Insight:** XOR simulates addition without carries, while AND followed by a left shift isolates and propagates the carries.
```cpp
int getSum(int a, int b) {
    while (b != 0) {
        unsigned carry = a & b; // Identify positions that generate a carry
        a = a ^ b;              // Sum without carry
        b = carry << 1;         // Shift carry to the next significant position
    }
    return a;
}
```
**Complexity:** O(1) Time | O(1) Space

## Single Number II (LC 137)
**Pattern:** Count bits mod 3 (Logic design)
**Key Insight:** Using two bitmasks (ones and twos), we keep track of bits that appear once or twice, clearing them when they appear a third time.
```cpp
int singleNumber(vector<int>& nums) {
    int ones = 0, twos = 0;
    for (int x : nums) {
        ones = (ones ^ x) & ~twos; // Add to ones if not already in twos
        twos = (twos ^ x) & ~ones; // Add to twos if not already in ones
    }
    return ones;
}
```
**Complexity:** O(N) Time | O(1) Space

## Single Number III (LC 260)
**Pattern:** Isolate lowest set bit to partition
**Key Insight:** Isolating any set bit from the XOR of all elements allows partitioning the array into two groups that each contain exactly one unique number.
```cpp
vector<int> singleNumber(vector<int>& nums) {
    long xor2 = 0;
    for (int n : nums) xor2 ^= n;
    int diff_bit = xor2 & -xor2; // Extracts the rightmost 1-bit where the two answers differ
    vector<int> res(2, 0);
    for (int n : nums) {
        if (n & diff_bit) res[0] ^= n; // Group 1: bit is set
        else res[1] ^= n;              // Group 2: bit is unset
    }
    return res;
}
```
**Complexity:** O(N) Time | O(1) Space

## Power of Two (LC 231)
**Pattern:** Clear lowest set bit
**Key Insight:** A power of two has exactly one set bit, so clearing its lowest set bit must result in 0.
```cpp
bool isPowerOfTwo(int n) {
    return n > 0 && (n & (n - 1)) == 0; // Clears the only set bit if power of 2
}
```
**Complexity:** O(1) Time | O(1) Space

## Bitwise AND of Numbers Range (LC 201)
**Pattern:** Common prefix
**Key Insight:** The bitwise AND of a range reduces to the common MSB prefix of the left and right bounds, padded with zeros.
```cpp
int rangeBitwiseAnd(int left, int right) {
    int shifts = 0;
    while (left < right) {
        left >>= 1;  // Shift until prefixes match
        right >>= 1;
        shifts++;
    }
    return left << shifts; // Pad with zeros up to original length
}
```
**Complexity:** O(1) Time | O(1) Space
