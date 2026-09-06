# 09 — Trie (Prefix Tree)

### Implement Trie (Prefix Tree) (LC 208)
**Pattern:** Basic Trie Node with `isWord`
**Key Insight:** A tree where each path down represents a string. We use an array of size 26 for quick child node lookups.
```cpp
struct TrieNode {
    TrieNode* children[26] = {};
    bool isWord = false; // Marks end of a valid word
};
TrieNode* root = new TrieNode();

void insert(string word) {
    TrieNode* node = root;
    for (char c : word) {
        if (!node->children[c - 'a']) node->children[c - 'a'] = new TrieNode(); // Create node if missing
        node = node->children[c - 'a'];
    }
    node->isWord = true;
}
```
**Complexity:** Time: `O(L)` | Space: `O(N * L)` where L is word length

### Design Add and Search Words Data Structure (LC 211)
**Pattern:** Trie + DFS (for wildcard '.')
**Key Insight:** A normal search handles letters directly, but for '.', we must DFS explore all existing children.
```cpp
bool dfs(string& word, int idx, TrieNode* node) {
    for (int i = idx; i < word.length(); i++) {
        char c = word[i];
        if (c == '.') {
            for (auto child : node->children) { // Try all possible letters
                if (child && dfs(word, i + 1, child)) return true;
            }
            return false;
        }
        if (!node->children[c - 'a']) return false;
        node = node->children[c - 'a'];
    }
    return node->isWord;
}
```
**Complexity:** Time: `O(26^L)` worst-case with '.' | Space: `O(N * L)`

### Word Search II (LC 212)
**Pattern:** Trie + DFS Backtracking
**Key Insight:** Put all target words in a Trie. Traverse the board and the Trie simultaneously, which drastically prunes invalid prefixes.
```cpp
void dfs(int r, int c, TrieNode* node, string word) {
    if (r < 0 || c < 0 || r >= R || c >= C || !node || board[r][c] == '#') return;
    char ch = board[r][c];
    node = node->children[ch - 'a'];
    if (!node) return; // Prune if no prefix matches
    word += ch;
    if (node->isWord) { res.insert(word); node->isWord = false; /* dedup */ }
    
    board[r][c] = '#'; // mark visited
    dfs(r+1, c, node, word); dfs(r-1, c, node, word);
    dfs(r, c+1, node, word); dfs(r, c-1, node, word);
    board[r][c] = ch; // backtrack
}
```
**Complexity:** Time: `O(M * N * 4^L)` | Space: `O(Total Letters)`

### Longest Word in Dictionary (LC 720)
**Pattern:** Trie + BFS/DFS for longest valid path
**Key Insight:** Only traverse to children that are marked as complete words (`isWord = true`), ensuring we build words one character at a time.
```cpp
// After inserting all words into Trie
string res = "";
void dfs(TrieNode* node, string word) {
    // Update res if we find a longer word or lexicographically smaller one of the same length
    if (word.length() > res.length() || (word.length() == res.length() && word < res)) {
        res = word;
    }
    for (int i = 0; i < 26; i++) {
        if (node->children[i] && node->children[i]->isWord) { // Only extend valid words
            dfs(node->children[i], word + (char)('a' + i));
        }
    }
}
```
**Complexity:** Time: `O(Sum of L)` | Space: `O(Sum of L)`

### Search Suggestions System (LC 1268)
**Pattern:** Trie storing Top 3 results at each node
**Key Insight:** Instead of DFS at query time, precompute and store the top 3 lexicographical strings at each Trie node during insertion.
```cpp
struct TrieNode {
    TrieNode* children[26] = {};
    vector<string> top3; // Cache best suggestions at this prefix node
};
// On insert, add word to node->top3, keeping it sorted and max size 3
// Then search prefix simply traverses and returns node->top3
```
*(Alternative without Trie: sort + binary search `O(N log N + M log N)`)*
**Complexity:** Time: `O(N log N + Sum of L)` | Space: `O(Sum of L)`

### Replace Words (LC 648)
**Pattern:** Shortest Prefix Match (Trie)
**Key Insight:** Traverse the Trie and return early at the very first `isWord` node encountered, replacing the original word with its shortest root.
```cpp
string getShortestPrefix(string word, TrieNode* root) {
    TrieNode* node = root;
    string prefix = "";
    for (char c : word) {
        if (!node->children[c - 'a']) break;
        prefix += c;
        node = node->children[c - 'a'];
        if (node->isWord) return prefix; // Found shortest root
    }
    return word; // no replacement
}
```
**Complexity:** Time: `O(N * L + S)` | Space: `O(N * L)`

### Maximum XOR of Two Numbers in an Array (LC 421)
**Pattern:** Bitwise Trie (Insert MSB to LSB)
**Key Insight:** To maximize XOR, we insert numbers into a Trie bit-by-bit (MSB first) and always greedily try to traverse the opposite bit.
```cpp
struct BitNode { BitNode* child[2] = {}; };
// Insert nums in 31 to 0 bit order
int maxXor = 0;
for (int num : nums) {
    BitNode* node = root;
    int currXor = 0;
    for (int i = 31; i >= 0; i--) {
        int bit = (num >> i) & 1;
        if (node->child[1 - bit]) { // Greedily pick opposite bit for max XOR
            currXor |= (1 << i);
            node = node->child[1 - bit];
        } else {
            node = node->child[bit]; // Forced to take same bit
        }
    }
    maxXor = max(maxXor, currXor);
}
```
**Complexity:** Time: `O(N)` | Space: `O(N)`
