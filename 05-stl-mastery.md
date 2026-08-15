# Module 05 — The Standard Template Library (STL) Mastery

The Standard Template Library (STL) is the heart of C++. It provides a set of generalized, generic components—containers, iterators, and algorithms—that work seamlessly together to form a powerful, high-performance library for common programming tasks. Understanding the STL is critical for writing idiomatic, fast, and robust C++ code.

---

## 1. Containers

Containers are objects that store collections of other objects. The STL provides different types of containers optimized for various operations and use cases.

### Sequence Containers

Sequence containers store elements in a linear arrangement.

#### `std::array` (C++11)
A fixed-size array wrapper. Unlike C-style arrays, it doesn't decay to pointers and knows its size.
- **Internals:** Contiguous memory block allocated on the stack (usually). No dynamic allocation overhead.
- **Time Complexity:** Access `O(1)`.
- **Use case:** When size is known at compile-time and fixed.

```cpp
#include <array>
#include <iostream>

int main() {
    std::array<int, 5> arr = {1, 2, 3, 4, 5};
    // Safe bounds-checked access
    try {
        std::cout << arr.at(5) << '\n'; // Throws std::out_of_range
    } catch(const std::exception& e) {
        std::cerr << e.what() << '\n';
    }
    return 0;
}
```

#### `std::vector`
A dynamic array. The default container for most use cases in C++.
- **Internals:** Manages a dynamically allocated contiguous array. When capacity is exceeded, it allocates a larger chunk (typically 1.5x or 2x the old capacity), copies/moves elements, and frees the old chunk.
- **Time Complexity:**
  - Access: `O(1)`
  - Push back: Amortized `O(1)` (worst-case `O(N)` during reallocation)
  - Insertion/Removal anywhere except the end: `O(N)`
- **Use case:** When you need dynamic sizing, fast random access, and cache friendliness.

```cpp
#include <vector>
#include <iostream>

int main() {
    std::vector<int> v;
    v.reserve(10); // 💡 Tip: Preallocate to avoid reallocations!
    for (int i = 0; i < 10; ++i) {
        v.push_back(i);
    }
    return 0;
}
```

#### `std::deque` (Double-Ended Queue)
- **Internals:** Often implemented as a fixed-size array of pointers to fixed-size memory chunks (pages). It provides non-contiguous but random access.
- **Time Complexity:** Access `O(1)` (slightly slower than vector), Push/Pop front and back: `O(1)`.
- **Use case:** Queue-like behavior where you need fast insertion/deletion at both ends.

#### `std::list` and `std::forward_list`
- **Internals:** `std::list` is a doubly-linked list. `std::forward_list` is a singly-linked list (optimized for minimal overhead).
- **Time Complexity:** Access `O(N)`, Insertion/Removal `O(1)` *if* you have an iterator to the position.
- **Use case:** Heavy insertion/removal in the middle of the collection. Splicing elements between lists.

### Associative Containers

Associative containers automatically sort their elements. They are typically implemented as **Red-Black Trees**, a type of self-balancing binary search tree.

#### `std::set` and `std::map`
- `set`: Unique elements.
- `map`: Unique keys, stores key-value pairs.
- **Time Complexity:** Search, Insert, Erase: `O(log N)`.

```cpp
#include <map>
#include <string>
#include <iostream>

struct CustomCompare {
    bool operator()(const std::string& a, const std::string& b) const {
        return a.length() < b.length(); // Sort by length
    }
};

int main() {
    std::map<std::string, int, CustomCompare> myMap;
    myMap["apple"] = 1;
    myMap["banana"] = 2;
    myMap["kiwi"] = 3;
    // Iterates in order of string length
    for(const auto& [key, val] : myMap) {
        std::cout << key << ": " << val << '\n';
    }
}
```

#### `std::multiset` and `std::multimap`
Like `set` and `map`, but allow duplicate elements/keys.

### Unordered Containers (C++11)

Implemented as **Hash Tables**.
- **Internals:** Uses an array of "buckets." Elements are hashed, and the hash determines the bucket. If multiple elements land in the same bucket (collision), they are chained (e.g., linked list). When the `load_factor` (elements / buckets) exceeds the `max_load_factor`, the table is rehashed (bucket array grows, and elements are reassigned).
- **Containers:** `unordered_set`, `unordered_map`, `unordered_multiset`, `unordered_multimap`.
- **Time Complexity:** Search, Insert, Erase: Average `O(1)`, Worst-case `O(N)` (if all elements collide).

```cpp
#include <unordered_map>
#include <string>

// Custom hash function
struct Point { int x, y; };
struct PointHash {
    std::size_t operator()(const Point& p) const {
        return std::hash<int>()(p.x) ^ (std::hash<int>()(p.y) << 1);
    }
};
struct PointEqual {
    bool operator()(const Point& a, const Point& b) const {
        return a.x == b.x && a.y == b.y;
    }
};

int main() {
    std::unordered_map<Point, std::string, PointHash, PointEqual> points;
    points[{1, 2}] = "Origin";
}
```
**Ordered vs Unordered:** Use unordered when you only need fast lookup. Use ordered when you need range queries, ordered iteration, or `O(log N)` guaranteed worst-case performance.

### Container Adaptors
Wrappers around other containers to provide restricted interfaces.
- `std::stack`: LIFO (Uses `deque` by default).
- `std::queue`: FIFO (Uses `deque` by default).
- `std::priority_queue`: Max-heap structure (Uses `vector` by default). `O(log N)` insertions/removals.

### Flat Containers (C++23)
`std::flat_map` and `std::flat_set` provide map/set interfaces but are backed by sorted vectors.
- **Benefits:** Better cache locality than trees, faster iteration, lower memory overhead.
- **Trade-offs:** Slower insertions/deletions (`O(N)`).

---

## 2. Iterators

Iterators are objects that point to elements in a container. They act as the bridge between containers and algorithms.

### Iterator Categories
From least to most powerful:
1. **Input Iterator:** Read forward once (`istream_iterator`).
2. **Output Iterator:** Write forward once (`ostream_iterator`).
3. **Forward Iterator:** Read/Write forward multiple times (`forward_list`).
4. **Bidirectional Iterator:** Read/Write forward and backward (`list`, `set`, `map`).
5. **Random Access Iterator:** Read/Write, `+` and `-` integer offsets in `O(1)` (`deque`).
6. **Contiguous Iterator (C++20):** Random access, plus elements are guaranteed contiguous in memory (`array`, `vector`, `string`).

### Iterator Invalidation Rules (⚠️ CRITICAL)
- `vector`: Reallocation invalidates **all** iterators, pointers, and references. Insert/erase invalidates iterators after the point of modification.
- `deque`: Insert at middle invalidates **all** iterators and references. Insert at ends invalidates iterators, but references remain valid. Erase invalidates iterators depending on position.
- `list`, `forward_list`: Iterators remain valid unless the specific element they point to is erased.
- `set`, `map`: Iterators remain valid unless the specific element is erased.
- `unordered_map`: Rehashing invalidates **all** iterators (but not references). Erase invalidates only the erased element's iterator.

### Iterator Adaptors
- `std::reverse_iterator`: Reverses iteration direction (`rbegin()`, `rend()`).
- `std::move_iterator`: Dereferences to an rvalue reference, facilitating move semantics.
- `std::back_inserter`: Appends elements (`push_back`).

---

## 3. Algorithms

The STL provides >100 algorithms in `<algorithm>` and `<numeric>`.

### Non-Modifying
```cpp
std::vector<int> v = {1, 2, 3, 4, 5};
bool has_even = std::any_of(v.begin(), v.end(), [](int n){ return n % 2 == 0; });
int count_threes = std::count(v.begin(), v.end(), 3);
auto it = std::find(v.begin(), v.end(), 4);
```

### Modifying
```cpp
// Erase-Remove idiom (Pre C++20)
v.erase(std::remove(v.begin(), v.end(), 3), v.end());

// C++20 standardizes this
std::erase(v, 3); 

// Transform
std::vector<int> out(v.size());
std::transform(v.begin(), v.end(), out.begin(), [](int x) { return x * 2; });
```

### Sorting and Searching
- `std::sort`: `O(N log N)` Introsort.
- `std::stable_sort`: Preserves relative order of equal elements.
- `std::partial_sort`: Sorts only the top N elements.
- `std::nth_element`: Finds the Nth element as if the container was sorted.
- **Binary Search** (Requires sorted range): `std::lower_bound`, `std::upper_bound`, `std::binary_search`.

### Parallel Algorithms (C++17)
```cpp
#include <execution>
#include <algorithm>
#include <vector>

std::vector<int> v = { /* huge data */ };
// Sort in parallel
std::sort(std::execution::par, v.begin(), v.end());
```

---

## 4. Ranges (C++20)

Ranges drastically simplify STL usage. A "range" is an object with a `begin()` and `end()`.

### Range Views and Pipelines
Views are non-owning, lazy-evaluated adaptors.
```cpp
#include <iostream>
#include <vector>
#include <ranges>

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5, 6};

    auto even_squares = v 
        | std::views::filter([](int n){ return n % 2 == 0; })
        | std::views::transform([](int n){ return n * n; });

    // Output: 4 16 36
    for (int i : even_squares) {
        std::cout << i << " ";
    }
}
```

### Projections
Ranges algorithms accept "projections" to transform elements before operating on them.
```cpp
struct Person { std::string name; int age; };
std::vector<Person> people = {{"Alice", 30}, {"Bob", 20}};
// Sort by age using a projection
std::ranges::sort(people, std::less{}, &Person::age);
```

---

## 5. Strings

### `std::string` and SSO
- **SSO (Small String Optimization):** `std::string` implementations avoid dynamic heap allocation for small strings (typically < 15 or 23 chars) by storing them directly in the string object's footprint.
- **`std::string_view` (C++17):** A non-owning, read-only view of a string. Extremely cheap to copy. Pass `std::string_view` by value instead of `const std::string&` when you don't need ownership.

```cpp
#include <string_view>
#include <iostream>

void print_len(std::string_view sv) {
    std::cout << sv.length() << '\n';
}

int main() {
    print_len("Hello"); // No allocation!
    std::string s = "World";
    print_len(s); // Cheap conversion
}
```

---

## 6. Utility

### `std::pair` and `std::tuple`
Bundling diverse data types.
```cpp
#include <tuple>
std::tuple<int, double, std::string> get_data() {
    return {1, 3.14, "hello"};
}
// C++17 Structured Binding
auto [i, d, s] = get_data();
```

### Modern Wrappers (C++17)
- `std::optional<T>`: Represents an object that might or might not contain a value.
- `std::variant<T...>`: Type-safe union.
- `std::any`: Can hold any type.

```cpp
#include <optional>
std::optional<int> divide(int a, int b) {
    if (b == 0) return std::nullopt;
    return a / b;
}
```

---

## 7. Interview Questions

1. **How does `std::vector` grow internally?**
   **Answer:** When capacity is full, it allocates a new larger buffer (usually 1.5x or 2x the current size), moves/copies elements to the new buffer, and deallocates the old buffer. If element types are `noexcept` movable, it uses move constructors; otherwise, it copies.

2. **Why should you use `std::vector::reserve()`?**
   **Answer:** To pre-allocate memory. It prevents multiple costly reallocations and element copying/moving when adding many elements.

3. **What is iterator invalidation?**
   **Answer:** It happens when underlying memory is reallocated or shifted. Pointers, references, and iterators to elements become dangling (pointing to invalid memory).

4. **Which container would you use for frequent insertions and deletions at both ends?**
   **Answer:** `std::deque`.

5. **Why might `std::vector` be faster than `std::list` even for insertions in the middle?**
   **Answer:** Cache locality. `vector` elements are contiguous, so they fit cleanly in CPU caches. `list` nodes are scattered in memory, causing cache misses. Shifting elements in a `vector` in L1 cache is often faster than traversing pointers in RAM.

6. **Explain Small String Optimization (SSO).**
   **Answer:** `std::string` objects have a small internal buffer (e.g., 15 bytes). If a string fits in this buffer, no dynamic heap allocation is performed, greatly improving performance.

7. **What is the complexity of `std::sort`?**
   **Answer:** `O(N log N)`. It typically uses Introsort (QuickSort, switching to HeapSort if recursion depth gets too deep, and InsertionSort for small chunks).

8. **When would you use `std::unordered_map` vs `std::map`?**
   **Answer:** Use `unordered_map` (Hash table) when you need `O(1)` average lookup and don't care about order. Use `map` (Red-Black Tree) when you need elements sorted, range-based queries (e.g., `lower_bound`), or guaranteed `O(log N)` worst-case time (hash table worst case is `O(N)`).

9. **What happens if a hash function for `std::unordered_map` always returns 1?**
   **Answer:** All elements hash to the same bucket, causing massive collisions. The map degrades into a linked list, and lookup becomes `O(N)`.

10. **Explain the Erase-Remove idiom.**
    **Answer:** `std::remove` doesn't actually delete elements; it shifts non-removed elements to the front and returns an iterator to the new logical end. You must call `container.erase(new_end, container.end())` to physically remove them. C++20 added `std::erase` to simplify this.

11. **What is `std::string_view`?**
    **Answer:** A lightweight, non-owning view over a character sequence. It consists of a pointer and a length. It avoids heap allocations when creating substrings or passing string literals to functions.

12. **How do you write a custom hash function for `std::unordered_map`?**
    **Answer:** Provide a struct that implements `std::size_t operator()(const T& key) const`, and optionally a custom equality operator struct.

13. **What is the difference between `std::set` and `std::unordered_set`?**
    **Answer:** `set` is ordered, backed by a tree, `O(log N)` access. `unordered_set` is not ordered, backed by a hash table, `O(1)` average access.

14. **Why use `std::make_shared` over `std::shared_ptr<T>(new T())`?**
    **Answer:** `make_shared` performs a single memory allocation for both the object and the control block. `new T()` followed by `shared_ptr` constructor performs two allocations.

15. **What does `std::move` actually do?**
    **Answer:** It performs a static cast to an rvalue reference (`static_cast<T&&>`). It does not move anything itself; it just enables the move constructor or move assignment operator to take over.

16. **How does `std::deque` maintain fast insertions at the front?**
    **Answer:** It's implemented as a "map" of pointers to fixed-size memory chunks. It can allocate a new chunk at the front and update the map without moving existing elements.

17. **What is the complexity of `std::distance`?**
    **Answer:** `O(1)` for random access iterators (like vector), and `O(N)` for input/forward/bidirectional iterators (like list).

18. **Why is `std::list::size()` `O(1)` in C++11 and later?**
    **Answer:** The standard mandated `O(1)` size for lists, meaning implementations must maintain an internal size counter, at the cost of making `splice()` operations `O(N)` if elements are moved between lists and their count needs to be known.

19. **What are the benefits of C++20 Ranges?**
    **Answer:** Composability (pipelines via `|`), lazy evaluation (views only compute when iterated), and prevention of dangling iterators.

20. **Can you modify the key of an element in a `std::set`?**
    **Answer:** No, keys are effectively `const`. Modifying a key would break the internal tree structure. You must extract the node (C++17), modify it, and reinsert it, or erase and insert a new element.
