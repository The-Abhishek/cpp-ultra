# Design Logging Framework (like Log4j)

## 1. Requirements & Use Cases
- **Actors:** Application Code
- **Use Cases:** Log messages at different levels (DEBUG/INFO/WARN/ERROR/FATAL), output to multiple sinks (console, file, network), format customization.

## 2. Terminology & Entities
- **Sink [Destination for logs]:** e.g., Console, File.
- **Asynchronous Logging [Logging on a separate thread]:** Prevents I/O operations from blocking the main application thread.
- **Entities:** Logger, LogLevel, LogMessage, ILogSink, ILogFormatter.

## 3. Design Patterns & Principles
- **Singleton Pattern:** Ensure only one Logger instance exists. Why? Centralized configuration and resource management.
- **Chain of Responsibility Pattern:** For level filtering.
- **Observer Pattern:** Registering multiple sinks to the logger.
- **Strategy Pattern:** Formatting logic. Why? Allows dynamic formatting based on configuration.
- **SOLID Principles:** Dependency Inversion by using interfaces (`ILogSink`).

## 4. ASCII Class Diagram
```text
+-------------------+        +----------------+
|     Logger        |<>------|   ILogSink     |
+-------------------+        +----------------+
| - instance        |        | + write(msg)   |
| - sinks[]         |        +----------------+
| + log(level, msg) |               ^
+-------------------+               |
                                    +---------+
                                    |         |
                              +----------+ +--------+
                              |ConsoleSnk| |FileSnk |
                              +----------+ +--------+
```

## 5. Deep Dive
**Thread-Safe Singleton & Async Logging:**
- Modern C++ uses Meyer's Singleton (`static Logger instance;`) for thread-safe initialization.
- Async logging uses a background thread and a thread-safe queue (`std::queue` with `std::mutex` and `std::condition_variable`) to offload I/O.

## 6. Full Working C++ Implementation
```cpp
#include <iostream>
#include <string>
#include <vector>
#include <memory>
#include <mutex>

enum class LogLevel { DEBUG, INFO, WARN, ERROR, FATAL };

class ILogSink {
public:
    virtual void write(LogLevel level, const std::string& message) = 0;
    virtual ~ILogSink() = default;
};

class ConsoleSink : public ILogSink {
public:
    void write(LogLevel level, const std::string& message) override {
        // Simplified formatting for demo
        std::cout << "[" << static_cast<int>(level) << "] " << message << "\n";
    }
};

class Logger {
    std::vector<std::shared_ptr<ILogSink>> sinks;
    LogLevel minLevel;
    std::mutex mtx;
    
    Logger() : minLevel(LogLevel::INFO) {}
public:
    // Meyer's Singleton
    static Logger& getInstance() {
        static Logger instance;
        return instance;
    }
    
    // Delete copy/move
    Logger(const Logger&) = delete;
    Logger& operator=(const Logger&) = delete;
    
    void addSink(std::shared_ptr<ILogSink> sink) {
        std::lock_guard<std::mutex> lock(mtx);
        sinks.push_back(sink);
    }
    
    void log(LogLevel level, const std::string& message) {
        if (level < minLevel) return;
        
        std::lock_guard<std::mutex> lock(mtx);
        for (auto& sink : sinks) {
            sink->write(level, message);
        }
    }
};

int main() {
    Logger& logger = Logger::getInstance();
    logger.addSink(std::make_shared<ConsoleSink>());
    
    logger.log(LogLevel::DEBUG, "This will not print (default INFO level).");
    logger.log(LogLevel::INFO, "Application started.");
    logger.log(LogLevel::ERROR, "Database connection failed.");
    
    return 0;
}
```
