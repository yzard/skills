---
name: cpp-coding
description: Write C++ code following project coding guidelines. Use when the user asks to write C++ code, create classes, modify headers, or implement C++ features.
allowed-tools: Read, Edit, Write, Glob, Grep, Bash(cmake:*), Bash(clang-format:*)
---

# C++ Coding Guidelines

## What This Skill Does

Guides writing C++ code following the project's established conventions and best practices.

## Naming Conventions

### Classes and Namespaces - PascalCase

```cpp
// GOOD
class MessageConnection;
class WorkerManager;
namespace scaler::top {}

// BAD
class message_connection;
class workerManager;
```

### Functions and Variables - camelCase

```cpp
// GOOD
void processMessage(const std::vector<uint8_t>& payload);
int messageCount;
std::string workerAddress;

// BAD
void ProcessMessage();
void process_message();
int message_count;
```

### Abbreviations in Names

**PascalCase/camelCase (classes, functions, variables):** Abbreviations stay UPPERCASE.

```cpp
// GOOD - abbreviations uppercase in PascalCase (classes)
class TCPAcceptor;      // TCP is abbreviation
class IOSocket;         // IO is abbreviation
class HTTPClient;       // HTTP is abbreviation
class YMQSubscriber;    // YMQ is abbreviation

// GOOD - abbreviations uppercase in camelCase (functions/variables)
void sendHTTPRequest();
void processYMQMessage();
int maxTCPConnections;
std::string apiURL;

// BAD
class TcpAcceptor;      // TCP should be uppercase
class YmqSubscriber;    // YMQ should be uppercase
void sendHttpRequest(); // HTTP should be uppercase
int maxTcpConnections;  // TCP should be uppercase
```

**snake_case (file names):** Abbreviations are lowercase.

```cpp
// GOOD - file names use snake_case with lowercase abbreviations
tcp_acceptor.h
ymq_subscriber.cpp
http_client.h

// BAD
TCP_acceptor.h    // should be tcp_acceptor.h
YMQ_subscriber.cpp // should be ymq_subscriber.cpp
```

### Global Variables and Macros - UPPERCASE

```cpp
// GOOD
static const int MAX_BUFFER_SIZE = 1024;
#define SCALER_VERSION "1.0.0"

// BAD
static const int maxBufferSize = 1024;
#define scalerVersion "1.0.0"
```

### File Names - snake_case

```cpp
// GOOD
message_connection.h
message_connection.cpp
scheduler_state.h
ymq_subscriber.cpp

// BAD
MessageConnection.h
messageConnection.cpp
SchedulerState.h
```

## Namespace Rules

**Never use `using namespace` directives.**

```cpp
// GOOD - fully qualified names
std::cout << "Hello" << std::endl;
std::vector<uint8_t> buffer;
std::string address;

// BAD - using namespace directives
using namespace std;
cout << "Hello" << endl;
```

## Include Rules

### Order of Includes

1. System library includes
2. Standard library includes
3. Third-party library includes
4. In-library includes

```cpp
// GOOD - proper ordering
#include <sys/socket.h>      // system

#include <atomic>            // standard library
#include <memory>
#include <string>
#include <vector>

#include <ftxui/component/component.hpp>  // third-party

#include "scaler/ymq/io_context.h"        // in-library
#include "ui/top_screen.h"
```

### Explicit Includes - No Transitive Dependencies

Always include headers that directly define the symbols you use, even if they're already transitively included.

```cpp
// GOOD - explicit includes
#include <string>
#include <vector>
#include "some_module.h"  // even if some_module.h includes <string>

// BAD - relying on transitive includes
#include "some_module.h"  // hoping <string> comes through
```

### Remove Unused Includes

Delete includes when none of their symbols are used in the file.

### Direct Includes Over Indirect Includes

When a symbol is declared in a specific header, include that header directly. Do not include a different header just because it includes the needed one.

```cpp
// GOOD - include the declaring header directly
#include "a.h"  // declares foobar()

// BAD - indirect include via another header
#include "b.h"  // includes a.h, but foobar() is declared in a.h
```

## Struct vs Class

Use `struct` for passive data with minimal helpers. Use `class` for stateful types with encapsulated members.

```cpp
// GOOD - struct for passive data
struct Address {
    std::string domain;
    int port;
};

struct WorkerStatus {
    std::string workerId;
    std::string groupId;
    double totalCpu;
    uint64_t totalRss;
};

// GOOD - class for stateful types with encapsulation
class Client {
public:
    void send(const std::vector<uint8_t>& buffer);
    int getMessageCount() const;

private:
    int _messageCount;
    std::unique_ptr<Connection> _connection;
};
```

## Class Layout

Order class members as follows:
1. public, then protected, then private
2. Within each section: using types, constructors, member functions, member variables

```cpp
class Foobar {
public:
    // 1. Using types first
    using Callback = std::function<void(int)>;

    // 2. Constructors and destructor
    Foobar();
    Foobar(const std::string& name);
    ~Foobar();

    // 3. Member functions
    void processData();
    const std::string& getName() const;

    // 4. Member variables (public only if struct-like)

protected:
    void someProtectedMethod();

    int _protectedVariable;

private:
    void somePrivateMethod();

    std::string _name;
    int _count;
};
```

## OOP Rules

### Subtyping Allowed, Inheritance Disallowed

- **Subtyping (allowed)**: Override member functions that have no implementations (pure virtual / interfaces).
- **Inheritance (disallowed)**: Do not override functions that already provide implementations.

```cpp
// GOOD - interface with pure virtual methods (subtyping)
class Subscriber {
public:
    virtual ~Subscriber() = default;
    virtual bool connect() = 0;
    virtual std::vector<uint8_t> recv() = 0;
    virtual void requestStop() = 0;
};

class YMQSubscriber : public Subscriber {
public:
    bool connect() override;
    std::vector<uint8_t> recv() override;
    void requestStop() override;
};

// BAD - overriding implemented methods (inheritance)
class BaseProcessor {
public:
    virtual void process() {
        // default implementation
    }
};

class CustomProcessor : public BaseProcessor {
public:
    void process() override {  // BAD - overriding implementation
        // different implementation
    }
};
```

## Constants and Magic Numbers

No magic numbers. Every constant should be assigned to a well-named variable.

```cpp
// GOOD
static const int DEFAULT_TIMEOUT_MS = 5000;
static const int MAX_RETRY_COUNT = 3;
static const double CPU_THRESHOLD = 0.95;

void connect() {
    socket.setTimeout(DEFAULT_TIMEOUT_MS);
    for (int i = 0; i < MAX_RETRY_COUNT; ++i) {
        // ...
    }
}

// BAD - magic numbers
void connect() {
    socket.setTimeout(5000);  // what does 5000 mean?
    for (int i = 0; i < 3; ++i) {
        // ...
    }
}
```

## Formatting

Use the `.clang-format` file in the repo root for consistent formatting.

```bash
clang-format -i src/cpp/**/*.cpp src/cpp/**/*.h
```

## C++ Checklist

When writing C++ code:

1. [ ] PascalCase for classes/namespaces, camelCase for functions/variables
2. [ ] Abbreviations stay uppercase (TCP, IO, HTTP, YMQ)
3. [ ] snake_case for file names with .h/.cpp extensions
4. [ ] No `using namespace` - use fully qualified names
5. [ ] Includes ordered: system, standard, third-party, in-library
6. [ ] Explicit includes - don't rely on transitive dependencies
7. [ ] struct for passive data, class for stateful types
8. [ ] Class layout: public/protected/private, using types first
9. [ ] No magic numbers - use named constants
10. [ ] Subtyping allowed, inheritance disallowed
11. [ ] Remove unused includes
