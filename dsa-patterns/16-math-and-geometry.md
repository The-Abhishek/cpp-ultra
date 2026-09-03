# 16 — Math & Geometry

## Rotate Image (LC 48)
**Pattern:** Transpose + Reverse rows
**Key Insight:** Rotating 90 degrees clockwise is mathematically equivalent to transposing the matrix across the main diagonal and then reversing each row.
```cpp
void rotate(vector<vector<int>>& matrix) {
    int n = matrix.size();
    for (int i = 0; i < n; ++i)
        for (int j = i + 1; j < n; ++j)
            swap(matrix[i][j], matrix[j][i]); // Transpose across main diagonal
    for (int i = 0; i < n; ++i)
        reverse(matrix[i].begin(), matrix[i].end()); // Reverse rows for clockwise rotation
}
```
**Complexity:** O(N^2) Time | O(1) Space

## Spiral Matrix (LC 54)
**Pattern:** 4 boundaries simulation
**Key Insight:** Maintain four boundary pointers (top, bottom, left, right) and peel off the outer layers sequentially, shrinking the boundaries inward.
```cpp
vector<int> spiralOrder(vector<vector<int>>& matrix) {
    vector<int> res;
    int top = 0, bottom = matrix.size() - 1;
    int left = 0, right = matrix[0].size() - 1;
    while (top <= bottom && left <= right) {
        for (int j = left; j <= right; ++j) res.push_back(matrix[top][j]);
        top++;
        for (int i = top; i <= bottom; ++i) res.push_back(matrix[i][right]);
        right--;
        if (top <= bottom) { // Check if remaining row exists
            for (int j = right; j >= left; --j) res.push_back(matrix[bottom][j]);
            bottom--;
        }
        if (left <= right) { // Check if remaining col exists
            for (int i = bottom; i >= top; --i) res.push_back(matrix[i][left]);
            left++;
        }
    }
    return res;
}
```
**Complexity:** O(M*N) Time | O(1) Space

## Set Matrix Zeroes (LC 73)
**Pattern:** Use first row/col as markers
**Key Insight:** Instead of allocating a new array, use the first row and column to store zero states for the rest of the matrix, then process them backwards.
```cpp
void setZeroes(vector<vector<int>>& matrix) {
    int m = matrix.size(), n = matrix[0].size();
    bool firstRowZero = false, firstColZero = false;
    for (int i = 0; i < m; i++) if (matrix[i][0] == 0) firstColZero = true;
    for (int j = 0; j < n; j++) if (matrix[0][j] == 0) firstRowZero = true;
    
    for (int i = 1; i < m; i++)
        for (int j = 1; j < n; j++)
            if (matrix[i][j] == 0) matrix[i][0] = matrix[0][j] = 0; // Use first row/col as O(1) storage
            
    for (int i = 1; i < m; i++)
        for (int j = 1; j < n; j++)
            if (matrix[i][0] == 0 || matrix[0][j] == 0) matrix[i][j] = 0;
            
    if (firstColZero) for (int i = 0; i < m; i++) matrix[i][0] = 0;
    if (firstRowZero) for (int j = 0; j < n; j++) matrix[0][j] = 0;
}
```
**Complexity:** O(M*N) Time | O(1) Space

## Happy Number (LC 202)
**Pattern:** Floyd's Cycle Detection (Tortoise and Hare)
**Key Insight:** The sequence of digit square sums will either reach 1 or fall into a predictable cycle, allowing standard cycle detection via slow and fast pointers.
```cpp
int getNext(int n) {
    int sum = 0;
    while (n) {
        int d = n % 10;
        sum += d * d;
        n /= 10;
    }
    return sum;
}
bool isHappy(int n) {
    int slow = n, fast = getNext(n);
    while (fast != 1 && slow != fast) {
        slow = getNext(slow);
        fast = getNext(getNext(fast)); // Hare moves 2 steps, Tortoise moves 1
    }
    return fast == 1;
}
```
**Complexity:** O(log N) Time | O(1) Space

## Plus One (LC 66)
**Pattern:** Carry propagation
**Key Insight:** Traverse from right to left; if a digit is less than 9, increment and return. If all digits are 9, prepend a 1.
```cpp
vector<int> plusOne(vector<int>& digits) {
    for (int i = digits.size() - 1; i >= 0; --i) {
        if (digits[i] < 9) {
            digits[i]++;
            return digits;
        }
        digits[i] = 0; // 9 becomes 0, carry over 1 to next iteration
    }
    digits.insert(digits.begin(), 1); // If loop finishes, we had all 9s (e.g. 999 -> 1000)
    return digits;
}
```
**Complexity:** O(N) Time | O(1) Extra Space

## Pow(x, n) (LC 50)
**Pattern:** Fast Exponentiation (Binary Exponentiation)
**Key Insight:** Compute power in O(log n) by squaring the base and halving the exponent at each step, multiplying into the result when the exponent is odd.
```cpp
double myPow(double x, int n) {
    long long nn = n;
    if (nn < 0) { x = 1 / x; nn = -nn; }
    double res = 1;
    while (nn) {
        if (nn & 1) res *= x; // Multiply into result if current exponent bit is odd
        x *= x;               // Square the base
        nn >>= 1;             // Halve the exponent
    }
    return res;
}
```
**Complexity:** O(log N) Time | O(1) Space

## Multiply Strings (LC 43)
**Pattern:** Grade-school multiplication
**Key Insight:** The product of digits at indices `i` and `j` always gets placed at indices `i+j` and `i+j+1` in the result array.
```cpp
string multiply(string num1, string num2) {
    if (num1 == "0" || num2 == "0") return "0";
    int n = num1.size(), m = num2.size();
    vector<int> res(n + m, 0);
    for (int i = n - 1; i >= 0; --i) {
        for (int j = m - 1; j >= 0; --j) {
            int mul = (num1[i] - '0') * (num2[j] - '0');
            int p1 = i + j, p2 = i + j + 1; // p1 acts as carry, p2 is current digit
            int sum = mul + res[p2];
            res[p2] = sum % 10;
            res[p1] += sum / 10;
        }
    }
    string s = "";
    int i = 0;
    while (i < res.size() && res[i] == 0) i++;
    while (i < res.size()) s += to_string(res[i++]);
    return s;
}
```
**Complexity:** O(M*N) Time | O(M+N) Space

## Detect Squares (LC 2013)
**Pattern:** Diagonal matching + count map
**Key Insight:** Iterate through all points to find diagonals; if a point forms a valid diagonal with the query point, check if the other two corners exist to form a square.
```cpp
class DetectSquares {
    map<pair<int, int>, int> pts;
public:
    void add(vector<int> point) {
        pts[{point[0], point[1]}]++;
    }
    int count(vector<int> point) {
        int res = 0, px = point[0], py = point[1];
        for (auto& [pt, cnt] : pts) {
            int x = pt.first, y = pt.second;
            if (abs(px - x) != 0 && abs(px - x) == abs(py - y)) { // Check if it forms a valid diagonal
                if (pts.count({x, py}) && pts.count({px, y})) {   // Check for other two corners
                    res += cnt * pts[{x, py}] * pts[{px, y}];
                }
            }
        }
        return res;
    }
};
```
**Complexity:** O(N) Query Time | O(N) Space

## GCD / LCM Pattern
**Pattern:** Euclidean Algorithm
**Key Insight:** The GCD of a and b is the same as the GCD of b and a%b, reducing the problem size logarithmically until remainder is 0.
```cpp
int gcd(int a, int b) {
    return b == 0 ? a : gcd(b, a % b); // Logarithmic step reduction
}
int lcm(int a, int b) {
    return (a / gcd(a, b)) * b;
}
```
**Complexity:** O(log(min(a, b))) Time | O(1) Space

## Sieve of Eratosthenes (LC 204)
**Pattern:** Mark multiples
**Key Insight:** Iteratively mark the multiples of each uncrossed number as composite, efficiently isolating primes up to N.
```cpp
int countPrimes(int n) {
    if (n <= 2) return 0;
    vector<bool> isPrime(n, true);
    int count = 0;
    for (int p = 2; p < n; p++) {
        if (isPrime[p]) {
            count++;
            for (int i = 2 * p; i < n; i += p) // Mark all multiples of p as non-prime
                isPrime[i] = false;            // or i = p * p if avoiding overflow
        }
    }
    return count;
}
```
**Complexity:** O(N log log N) Time | O(N) Space

## Fast Modular Exponentiation
**Pattern:** Binary exponentiation modulo
**Key Insight:** Apply the modulo at each multiplication step to prevent integer overflow while retaining the logarithmic time complexity of fast exponentiation.
```cpp
long long modPow(long long base, long long exp, int mod) {
    long long res = 1;
    base %= mod;
    while (exp > 0) {
        if (exp % 2 == 1) res = (res * base) % mod; // Mod after multiplying
        base = (base * base) % mod;
        exp /= 2;
    }
    return res;
}
```
**Complexity:** O(log(exp)) Time | O(1) Space

## nCr mod p
**Pattern:** Combinatorics with modular inverse
**Key Insight:** Division modulo `p` requires multiplying by the modular multiplicative inverse, computed via Fermat's Little Theorem using fast exponentiation.
```cpp
long long nCr(int n, int r, int p) {
    if (r == 0 || n == r) return 1;
    long long num = 1, den = 1;
    for (int i = 0; i < r; i++) {
        num = (num * (n - i)) % p;
        den = (den * (i + 1)) % p;
    }
    return (num * modPow(den, p - 2, p)) % p; // Fermat's Little Theorem for modulo inverse
}
```
**Complexity:** O(R + log P) Time | O(1) Space

## Matrix Exponentiation
**Pattern:** Fast Fibonacci O(log n)
**Key Insight:** A linear recurrence like Fibonacci can be expressed as a matrix multiplied by a state vector. Exponentiating this base matrix quickly gives the Nth term.
```cpp
void multiply(long long F[2][2], long long M[2][2]) {
    long long x =  F[0][0]*M[0][0] + F[0][1]*M[1][0];
    long long y =  F[0][0]*M[0][1] + F[0][1]*M[1][1];
    long long z =  F[1][0]*M[0][0] + F[1][1]*M[1][0];
    long long w =  F[1][0]*M[0][1] + F[1][1]*M[1][1];
    F[0][0] = x; F[0][1] = y; F[1][0] = z; F[1][1] = w;
}
void power(long long F[2][2], int n) {
    if(n == 0 || n == 1) return;
    long long M[2][2] = {{1, 1}, {1, 0}};
    power(F, n / 2);
    multiply(F, F);
    if (n % 2 != 0) multiply(F, M); // Apply base matrix if power is odd
}
```
**Complexity:** O(log N) Time | O(log N) Space
