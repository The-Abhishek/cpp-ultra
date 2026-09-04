# Design In-Memory File System

## 1. Requirements Clarification
**Functional Requirements:**
- Support operations: `mkdir`, `ls`, `cd`, `touch`, `cat`, `write`, `rm`.
- Support both absolute and relative paths.
- Path resolution mechanism (parsing `/a/b/c`).

**Non-Functional Requirements:**
- Fast path traversal.
- Thread safety [Concurrency Control (mechanism to handle multiple threads accessing shared resources)] using locks.

> **Interview Tip**: Always clarify if the file system needs to handle concurrent reads/writes and if soft/hard links are supported. 

## 2. Actors & Use Cases
- **User/Client App**: Executes commands like `mkdir`, `write`, `cat`.
- **System**: Maintains the directory tree state and metadata.

## 3. Core Entities & Patterns
- **FileSystem**: Singleton or main orchestrator handling commands.
- **INode (Index Node)**: Abstract base class representing both directories and files.
- **Directory**: Extends INode. Contains a hash map of children.
- **File**: Extends INode. Contains string content.

**Design Patterns:**
- **Composite Pattern**: *Why?* Directories contain files and other directories. Treating them uniformly via an `INode` interface allows recursive operations (like getting total size or deleting) without checking concrete types.
- **Iterator Pattern**: *Why?* Useful when iterating over directory contents without exposing internal `std::unordered_map`.

## 4. Class Diagram & Architecture

```text
+-------------------+
|    FileSystem     |
|-------------------|
| - root: Directory*|
| - current: INode* |
|-------------------|
| + mkdir(path)     |
| + ls(path)        |
+---------+---------+
          | contains
          v
+-------------------+       +-----------------------+
| <<interface>>     |<------|       Directory       |
|      INode        |       |-----------------------|
|-------------------|       | - children: map       |
| + getName()       |<---+  | + add(INode*)         |
| + isDirectory()   |    |  | + getChild(name)      |
| + getSize()       |    |  +-----------------------+
+-------------------+    |
                         |  +-----------------------+
                         +--|         File          |
                            |-----------------------|
                            | - content: string     |
                            | + append(string)      |
                            +-----------------------+
```

## 5. Deep Dive: Path Resolution
Path resolution converts a string like `"/home/user/docs/file.txt"` into a reference to an `INode`.
- **Logic**: Split the string by `/`.
- **Base**: If the string starts with `/`, begin traversal at `root`. Otherwise, begin at `current_directory`.
- **Traversal**: Iteratively look up each segment in the current `Directory`'s hash map. If a segment doesn't exist and we are writing, we might need to create it (if `mkdir -p` behavior is requested).

| Comparison | Naive Traversal | Hash Map (Trie-like) Traversal |
| :--- | :--- | :--- |
| **Lookup Time** | O(N * D) where N is items per dir | O(D) where D is path depth |
| **Space** | O(1) extra | O(V) for directory entries |

## 6. Full C++ Implementation

```cpp
#include <iostream>
#include <string>
#include <unordered_map>
#include <vector>
#include <sstream>
#include <memory>
#include <mutex>
#include <stdexcept>

// SOLID Principle: Dependency Inversion (depending on abstractions)
class INode {
protected:
    std::string name;
public:
    INode(const std::string& n) : name(n) {}
    virtual ~INode() = default;
    virtual bool isDirectory() const = 0;
    virtual std::string getName() const { return name; }
    virtual int getSize() const = 0;
};

class File : public INode {
private:
    std::string content;
public:
    File(const std::string& n) : INode(n) {}
    bool isDirectory() const override { return false; }
    int getSize() const override { return content.size(); }
    void write(const std::string& data) { content = data; }
    std::string read() const { return content; }
};

class Directory : public INode {
private:
    std::unordered_map<std::string, std::shared_ptr<INode>> children;
public:
    Directory(const std::string& n) : INode(n) {}
    bool isDirectory() const override { return true; }
    int getSize() const override {
        int total = 0;
        for (const auto& pair : children) {
            total += pair.second->getSize();
        }
        return total;
    }
    
    void addNode(std::shared_ptr<INode> node) {
        children[node->getName()] = node;
    }
    
    std::shared_ptr<INode> getChild(const std::string& childName) {
        if (children.find(childName) != children.end()) {
            return children[childName];
        }
        return nullptr;
    }
    
    std::vector<std::string> listContents() const {
        std::vector<std::string> res;
        for (const auto& pair : children) {
            res.push_back(pair.first);
        }
        return res;
    }
};

class FileSystem {
private:
    std::shared_ptr<Directory> root;
    std::mutex fs_mtx; // Thread-safety for concurrent access

    std::vector<std::string> splitPath(const std::string& path) {
        std::vector<std::string> parts;
        std::stringstream ss(path);
        std::string item;
        while (std::getline(ss, item, '/')) {
            if (!item.empty()) parts.push_back(item);
        }
        return parts;
    }

    std::shared_ptr<Directory> navigateToParent(const std::string& path, std::string& targetName) {
        std::vector<std::string> parts = splitPath(path);
        if (parts.empty()) return root;
        
        targetName = parts.back();
        parts.pop_back();
        
        std::shared_ptr<Directory> curr = root;
        for (const auto& part : parts) {
            auto next = curr->getChild(part);
            if (!next || !next->isDirectory()) {
                throw std::runtime_error("Invalid path");
            }
            curr = std::dynamic_pointer_cast<Directory>(next);
        }
        return curr;
    }

public:
    FileSystem() {
        root = std::make_shared<Directory>("/");
    }

    void mkdir(const std::string& path) {
        std::lock_guard<std::mutex> lock(fs_mtx);
        std::string newDirName;
        auto parent = navigateToParent(path, newDirName);
        if (parent->getChild(newDirName)) {
            throw std::runtime_error("Directory already exists");
        }
        parent->addNode(std::make_shared<Directory>(newDirName));
    }

    void writeToFile(const std::string& path, const std::string& content) {
        std::lock_guard<std::mutex> lock(fs_mtx);
        std::string fileName;
        auto parent = navigateToParent(path, fileName);
        
        auto child = parent->getChild(fileName);
        std::shared_ptr<File> file;
        
        if (!child) {
            file = std::make_shared<File>(fileName);
            parent->addNode(file);
        } else if (!child->isDirectory()) {
            file = std::dynamic_pointer_cast<File>(child);
        } else {
            throw std::runtime_error("Path is a directory");
        }
        file->write(content);
    }

    std::string cat(const std::string& path) {
        std::lock_guard<std::mutex> lock(fs_mtx);
        std::string fileName;
        auto parent = navigateToParent(path, fileName);
        auto child = parent->getChild(fileName);
        
        if (!child || child->isDirectory()) {
            throw std::runtime_error("File not found");
        }
        return std::dynamic_pointer_cast<File>(child)->read();
    }
};

int main() {
    FileSystem fs;
    fs.mkdir("/home");
    fs.mkdir("/home/user");
    fs.writeToFile("/home/user/docs.txt", "Hello World!");
    std::cout << fs.cat("/home/user/docs.txt") << std::endl;
    return 0;
}
```
