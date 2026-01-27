# Technical Interview Questions - Answers

## Go Programming

### 1. Mutex and Channel

**Mutex (Mutual Exclusion Lock):**
- A synchronization primitive used to protect shared data from concurrent access
- Requires explicit locking and unlocking
- Lower-level primitive for protecting critical sections
- Used when you need to guard shared memory

**Channel:**
- A communication mechanism between goroutines
- Built on CSP (Communicating Sequential Processes) principles
- Provides both communication and synchronization
- "Don't communicate by sharing memory; share memory by communicating"

**Key Differences:**

| Aspect | Mutex | Channel |
|--------|-------|---------|
| Purpose | Protect shared state | Communication between goroutines |
| Philosophy | Shared memory | Message passing |
| Ownership | Multiple goroutines can lock/unlock | Data flows through channel |
| Complexity | Requires manual lock/unlock | Automatic synchronization |
| Deadlock Risk | Higher if not careful | Can happen with incorrect usage |
| Use Case | Protecting shared data structures | Coordinating work, pipelines |

**Example - Mutex:**
```go
type Counter struct {
    mu    sync.Mutex
    value int
}

func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}
```

**Example - Channel:**
```go
func worker(jobs <-chan int, results chan<- int) {
    for job := range jobs {
        results <- job * 2
    }
}
```

**When to use which:**
- Use **Mutex** when: Multiple goroutines need access to the same data structure, caching, counters, state management
- Use **Channel** when: Transferring data ownership, distributing work, signaling events, implementing pipelines

---

### 2. Producer-Consumer Pattern in Go

The Producer-Consumer pattern decouples data generation from data processing using channels.

**Basic Implementation:**

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// Producer generates data and sends it to the channel
func producer(id int, jobs chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for i := 0; i < 5; i++ {
        job := id*100 + i
        fmt.Printf("Producer %d: Producing job %d\n", id, job)
        jobs <- job
        time.Sleep(time.Millisecond * 100)
    }
}

// Consumer receives and processes data from the channel
func consumer(id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for job := range jobs {
        fmt.Printf("Consumer %d: Processing job %d\n", id, job)
        time.Sleep(time.Millisecond * 200)
        results <- job * 2
    }
}

func main() {
    const numProducers = 2
    const numConsumers = 3
    
    jobs := make(chan int, 10)    // Buffered channel for jobs
    results := make(chan int, 10) // Buffered channel for results
    
    var producerWg sync.WaitGroup
    var consumerWg sync.WaitGroup
    
    // Start producers
    for i := 1; i <= numProducers; i++ {
        producerWg.Add(1)
        go producer(i, jobs, &producerWg)
    }
    
    // Start consumers
    for i := 1; i <= numConsumers; i++ {
        consumerWg.Add(1)
        go consumer(i, jobs, results, &consumerWg)
    }
    
    // Wait for producers and close jobs channel
    go func() {
        producerWg.Wait()
        close(jobs)
    }()
    
    // Wait for consumers and close results channel
    go func() {
        consumerWg.Wait()
        close(results)
    }()
    
    // Collect results
    for result := range results {
        fmt.Printf("Result: %d\n", result)
    }
}
```

**Advanced Pattern with Context:**

```go
func producerWithContext(ctx context.Context, jobs chan<- int) {
    ticker := time.NewTicker(100 * time.Millisecond)
    defer ticker.Stop()
    
    jobID := 0
    for {
        select {
        case <-ctx.Done():
            fmt.Println("Producer: Context cancelled, stopping")
            return
        case <-ticker.C:
            select {
            case jobs <- jobID:
                fmt.Printf("Produced: %d\n", jobID)
                jobID++
            case <-ctx.Done():
                return
            }
        }
    }
}

func consumerWithContext(ctx context.Context, id int, jobs <-chan int) {
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("Consumer %d: Context cancelled\n", id)
            return
        case job, ok := <-jobs:
            if !ok {
                fmt.Printf("Consumer %d: Channel closed\n", id)
                return
            }
            fmt.Printf("Consumer %d processed job %d\n", id, job)
            time.Sleep(200 * time.Millisecond)
        }
    }
}
```

**Key Considerations:**
- Use buffered channels to prevent blocking
- Always close channels after producers finish
- Use `sync.WaitGroup` for coordination
- Consider using `context.Context` for cancellation
- Handle channel closure properly with `value, ok := <-channel`

---

### 3. Handling Out of Memory and High CPU Usage in Go Applications

#### **Memory Management:**

**Diagnosis Tools:**
```go
import (
    "runtime"
    "runtime/pprof"
    "os"
)

// Memory profiling
func profileMemory() {
    f, _ := os.Create("mem.prof")
    defer f.Close()
    runtime.GC() // Get up-to-date statistics
    pprof.WriteHeapProfile(f)
}

// Monitor memory stats
func printMemStats() {
    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    fmt.Printf("Alloc = %v MB", m.Alloc / 1024 / 1024)
    fmt.Printf("\tTotalAlloc = %v MB", m.TotalAlloc / 1024 / 1024)
    fmt.Printf("\tSys = %v MB", m.Sys / 1024 / 1024)
    fmt.Printf("\tNumGC = %v\n", m.NumGC)
}
```

**Prevention Strategies:**

1. **Use Object Pools:**
```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func process() {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer bufferPool.Put(buf)
    buf.Reset()
    // Use buffer
}
```

2. **Limit Goroutine Creation:**
```go
// Worker pool pattern
func workerPool(jobs <-chan Job, results chan<- Result, numWorkers int) {
    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- processJob(job)
            }
        }()
    }
    wg.Wait()
}
```

3. **Set Memory Limits:**
```go
// Set GOMEMLIMIT environment variable
// Or use runtime/debug
import "runtime/debug"

func init() {
    debug.SetMemoryLimit(1024 * 1024 * 1024) // 1GB
}
```

4. **Avoid Memory Leaks:**
```go
// Bad: goroutine leak
func badPattern(ctx context.Context) {
    ch := make(chan int)
    go func() {
        // This goroutine will never exit if nothing reads from ch
        ch <- computeValue()
    }()
}

// Good: proper cleanup
func goodPattern(ctx context.Context) {
    ch := make(chan int, 1) // Buffered channel
    go func() {
        select {
        case ch <- computeValue():
        case <-ctx.Done():
            return
        }
    }()
}
```

#### **CPU Usage Management:**

**Diagnosis Tools:**
```go
// CPU profiling
func profileCPU() {
    f, _ := os.Create("cpu.prof")
    defer f.Close()
    pprof.StartCPUProfile(f)
    defer pprof.StopCPUProfile()
    
    // Run your code here
}

// Use pprof HTTP endpoint
import _ "net/http/pprof"

func main() {
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
    // Your application code
}
```

**Optimization Strategies:**

1. **Control Goroutine Count:**
```go
// Use semaphore pattern
type Semaphore chan struct{}

func (s Semaphore) Acquire() {
    s <- struct{}{}
}

func (s Semaphore) Release() {
    <-s
}

sem := make(Semaphore, runtime.NumCPU())
for _, task := range tasks {
    sem.Acquire()
    go func(t Task) {
        defer sem.Release()
        processTask(t)
    }(task)
}
```

2. **Use GOMAXPROCS:**
```go
import "runtime"

func init() {
    // Limit CPU usage
    runtime.GOMAXPROCS(2) // Use only 2 CPU cores
}
```

3. **Avoid Busy Loops:**
```go
// Bad: busy loop
for {
    if condition {
        break
    }
}

// Good: use channels or timers
ticker := time.NewTicker(100 * time.Millisecond)
defer ticker.Stop()
for {
    select {
    case <-ticker.C:
        if condition {
            return
        }
    }
}
```

4. **Efficient Data Structures:**
```go
// Use sync.Map for concurrent access
var cache sync.Map

func getCached(key string) (value interface{}, ok bool) {
    return cache.Load(key)
}

func setCached(key string, value interface{}) {
    cache.Store(key, value)
}
```

**Monitoring and Alerting:**
```go
// Continuous monitoring
func monitorResources(ctx context.Context) {
    ticker := time.NewTicker(10 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            var m runtime.MemStats
            runtime.ReadMemStats(&m)
            
            // Alert if memory usage is high
            if m.Alloc > 500*1024*1024 { // 500MB
                log.Printf("WARNING: High memory usage: %v MB\n", 
                    m.Alloc/1024/1024)
            }
            
            // Check goroutine count
            if runtime.NumGoroutine() > 1000 {
                log.Printf("WARNING: High goroutine count: %d\n", 
                    runtime.NumGoroutine())
            }
        }
    }
}
```

---

## C++

### 1. Abstraction, Virtual Classes, VTable, and VPtr

#### **Abstraction:**
Abstraction is the concept of hiding implementation details and showing only essential features to the user.

```cpp
// Abstract class with pure virtual function
class Shape {
public:
    // Pure virtual function (makes class abstract)
    virtual double area() const = 0;
    virtual void draw() const = 0;
    virtual ~Shape() {} // Virtual destructor
};

class Circle : public Shape {
private:
    double radius;
public:
    Circle(double r) : radius(r) {}
    
    double area() const override {
        return 3.14159 * radius * radius;
    }
    
    void draw() const override {
        std::cout << "Drawing Circle\n";
    }
};
```

#### **Virtual Functions and Polymorphism:**

```cpp
class Base {
public:
    virtual void display() {
        std::cout << "Base display\n";
    }
    
    virtual ~Base() {}
};

class Derived : public Base {
public:
    void display() override {
        std::cout << "Derived display\n";
    }
};

int main() {
    Base* ptr = new Derived();
    ptr->display(); // Calls Derived::display() - Runtime polymorphism
    delete ptr;
}
```

#### **VTable (Virtual Table):**

The VTable is a lookup table of function pointers used to resolve virtual function calls at runtime.

**How it works:**
1. Each class with virtual functions has a VTable
2. VTable contains pointers to virtual functions
3. Compiler creates VTable at compile time
4. Each object has a hidden VPtr (virtual pointer)

```cpp
class Animal {
public:
    virtual void speak() { cout << "Animal speaks\n"; }
    virtual void move() { cout << "Animal moves\n"; }
    virtual ~Animal() {}
};

class Dog : public Animal {
public:
    void speak() override { cout << "Woof!\n"; }
    void move() override { cout << "Dog runs\n"; }
};

/*
Memory Layout:

Animal object:
+--------+
| VPtr   | -----> Animal VTable
+--------+        +----------+
| data   |        | &speak() |
+--------+        | &move()  |
                  | &~Animal|
                  +----------+

Dog object:
+--------+
| VPtr   | -----> Dog VTable
+--------+        +----------+
| data   |        | &speak() | (overridden)
+--------+        | &move()  | (overridden)
                  | &~Dog    |
                  +----------+
*/
```

#### **VPtr (Virtual Pointer):**

- Hidden pointer added by compiler as first member of class with virtual functions
- Points to class's VTable
- Initialized by constructor
- Size of VPtr = size of pointer (typically 8 bytes on 64-bit system)

```cpp
#include <iostream>
using namespace std;

class Base {
public:
    virtual void func1() { cout << "Base::func1\n"; }
    virtual void func2() { cout << "Base::func2\n"; }
    int data;
};

class Derived : public Base {
public:
    void func1() override { cout << "Derived::func1\n"; }
    // func2 not overridden
};

int main() {
    cout << "Size of Base: " << sizeof(Base) << endl;
    // Output: 16 (8 for vptr + 4 for int + 4 padding on 64-bit)
    
    Base* b = new Derived();
    b->func1();  // Calls Derived::func1 via VTable lookup
    b->func2();  // Calls Base::func2 via VTable lookup
    
    delete b;
}
```

**VTable Lookup Process:**
1. Compiler sees virtual function call
2. Gets VPtr from object
3. Uses VPtr to access VTable
4. Gets function pointer from VTable
5. Calls the function through pointer

**Performance Impact:**
- Virtual function call: One extra indirection (VPtr → VTable → Function)
- Negligible for most applications
- Can prevent inlining
- Each object has 8 extra bytes (VPtr overhead)

---

### 2. Memory Handling in C/C++ and Optimization Techniques

#### **Memory Segments:**

```cpp
// 1. Stack Memory (automatic storage)
void stackExample() {
    int x = 10;           // Stack allocated
    char arr[100];        // Stack allocated
    // Automatically deallocated when function returns
}

// 2. Heap Memory (dynamic storage)
void heapExample() {
    int* ptr = new int(20);           // Heap allocated
    int* arr = new int[100];          // Heap array
    delete ptr;                        // Must manually free
    delete[] arr;                      // Must use delete[]
}

// 3. Global/Static Memory
int global_var = 10;                   // Global memory
static int static_var = 20;            // Static memory

// 4. Code/Text Segment
void function() {                      // Code segment
    // Function code stored here
}

// 5. Constant Memory
const char* str = "Hello";             // String literal in const memory
```

#### **Manual Memory Management:**

```cpp
// C-style allocation
int* ptr = (int*)malloc(sizeof(int) * 10);
if (ptr == NULL) {
    // Handle allocation failure
}
// Use ptr
free(ptr);

// C++ style allocation
int* ptr = new int[10];
// Use ptr
delete[] ptr;

// Placement new (construct object in pre-allocated memory)
char buffer[sizeof(MyClass)];
MyClass* obj = new(buffer) MyClass();
obj->~MyClass();  // Must explicitly call destructor
```

#### **Smart Pointers (Modern C++):**

```cpp
#include <memory>

// 1. unique_ptr (exclusive ownership)
std::unique_ptr<int> ptr1 = std::make_unique<int>(10);
// ptr1.reset();  // Manually release
// Automatically freed when out of scope

// 2. shared_ptr (shared ownership with reference counting)
std::shared_ptr<int> ptr2 = std::make_shared<int>(20);
std::shared_ptr<int> ptr3 = ptr2;  // Both point to same object
// use_count() == 2
// Object deleted when last shared_ptr is destroyed

// 3. weak_ptr (non-owning reference)
std::weak_ptr<int> weak = ptr2;  // Doesn't increase ref count
if (auto locked = weak.lock()) {
    // Use locked (shared_ptr)
}
```

#### **Memory Optimization Techniques:**

**1. Object Pooling:**
```cpp
template<typename T>
class ObjectPool {
private:
    std::vector<T*> pool;
    std::vector<T*> available;
    
public:
    ObjectPool(size_t size) {
        for (size_t i = 0; i < size; i++) {
            T* obj = new T();
            pool.push_back(obj);
            available.push_back(obj);
        }
    }
    
    T* acquire() {
        if (available.empty()) {
            return new T();  // Expand pool
        }
        T* obj = available.back();
        available.pop_back();
        return obj;
    }
    
    void release(T* obj) {
        // Reset object state
        available.push_back(obj);
    }
    
    ~ObjectPool() {
        for (auto* obj : pool) {
            delete obj;
        }
    }
};
```

**2. Memory Arena/Linear Allocator:**
```cpp
class Arena {
private:
    char* buffer;
    size_t size;
    size_t offset;
    
public:
    Arena(size_t s) : size(s), offset(0) {
        buffer = new char[size];
    }
    
    void* allocate(size_t n) {
        if (offset + n > size) {
            throw std::bad_alloc();
        }
        void* ptr = buffer + offset;
        offset += n;
        return ptr;
    }
    
    void reset() {
        offset = 0;  // Reset without deallocating
    }
    
    ~Arena() {
        delete[] buffer;
    }
};
```

**3. Copy Elision and Move Semantics:**
```cpp
class BigObject {
private:
    int* data;
    size_t size;
    
public:
    // Constructor
    BigObject(size_t s) : size(s) {
        data = new int[size];
    }
    
    // Move constructor (efficient)
    BigObject(BigObject&& other) noexcept 
        : data(other.data), size(other.size) {
        other.data = nullptr;
        other.size = 0;
    }
    
    // Move assignment
    BigObject& operator=(BigObject&& other) noexcept {
        if (this != &other) {
            delete[] data;
            data = other.data;
            size = other.size;
            other.data = nullptr;
            other.size = 0;
        }
        return *this;
    }
    
    // Disable copy (force move)
    BigObject(const BigObject&) = delete;
    BigObject& operator=(const BigObject&) = delete;
    
    ~BigObject() {
        delete[] data;
    }
};

// Usage
BigObject createObject() {
    return BigObject(1000);  // Move, not copy
}
```

**4. Small String Optimization (SSO):**
```cpp
class String {
private:
    union {
        char* heap_ptr;        // For long strings
        char buffer[16];       // For short strings (SSO)
    };
    size_t size;
    size_t capacity;
    
    static const size_t SSO_SIZE = 15;
    
    bool is_small() const { return capacity <= SSO_SIZE; }
    
public:
    String(const char* s) {
        size = strlen(s);
        if (size <= SSO_SIZE) {
            strcpy(buffer, s);
            capacity = SSO_SIZE;
        } else {
            capacity = size + 1;
            heap_ptr = new char[capacity];
            strcpy(heap_ptr, s);
        }
    }
    
    ~String() {
        if (!is_small()) {
            delete[] heap_ptr;
        }
    }
};
```

**5. Cache-Friendly Data Structures:**
```cpp
// Bad: Array of pointers (pointer chasing)
struct BadNode {
    int* data;
    BadNode* next;
};

// Good: Data-oriented design
struct GoodNode {
    int data;        // Data embedded
    size_t next_idx; // Index instead of pointer
};

// Even better: Structure of Arrays (SoA)
struct Particles {
    std::vector<float> x;  // All x coordinates together
    std::vector<float> y;  // All y coordinates together
    std::vector<float> z;  // All z coordinates together
    // Better cache locality for operations on single component
};
```

**6. Alignment and Padding:**
```cpp
// Bad: Poor alignment
struct Bad {
    char a;    // 1 byte
    int b;     // 4 bytes (3 bytes padding before)
    char c;    // 1 byte (3 bytes padding after)
};  // Total: 12 bytes

// Good: Optimal alignment
struct Good {
    int b;     // 4 bytes
    char a;    // 1 byte
    char c;    // 1 byte (2 bytes padding after)
};  // Total: 8 bytes

// Force alignment
struct alignas(64) CacheLineAligned {
    int data;
    // Padded to 64 bytes
};
```

**7. Memory Leak Detection:**
```cpp
// Custom allocator with tracking
void* operator new(size_t size) {
    void* ptr = malloc(size);
    // Log allocation
    std::cout << "Allocated: " << size << " bytes at " << ptr << "\n";
    return ptr;
}

void operator delete(void* ptr) noexcept {
    // Log deallocation
    std::cout << "Freed: " << ptr << "\n";
    free(ptr);
}

// Use tools: Valgrind, AddressSanitizer, Dr. Memory
```

---

### 3. Memory Management: Calloc and Malloc Functions

#### **malloc() - Memory Allocation:**

```cpp
#include <stdlib.h>

void* malloc(size_t size);
```

**Characteristics:**
- Allocates specified number of bytes
- Returns pointer to allocated memory
- Memory is **uninitialized** (contains garbage values)
- Returns NULL if allocation fails
- Faster than calloc (no initialization)

**Example:**
```cpp
int* arr = (int*)malloc(10 * sizeof(int));
if (arr == NULL) {
    fprintf(stderr, "Memory allocation failed\n");
    exit(1);
}

// arr contains garbage values
for (int i = 0; i < 10; i++) {
    arr[i] = i;  // Must initialize manually
}

free(arr);
```

#### **calloc() - Contiguous Allocation:**

```cpp
void* calloc(size_t num, size_t size);
```

**Characteristics:**
- Allocates memory for array of elements
- Takes two parameters: number of elements and size of each
- **Initializes all bytes to zero**
- Returns pointer to allocated memory
- Returns NULL if allocation fails
- Slower than malloc due to initialization

**Example:**
```cpp
int* arr = (int*)calloc(10, sizeof(int));
if (arr == NULL) {
    fprintf(stderr, "Memory allocation failed\n");
    exit(1);
}

// arr is initialized to all zeros
// arr[0] = 0, arr[1] = 0, ... arr[9] = 0

free(arr);
```

#### **Comparison Table:**

| Feature | malloc | calloc |
|---------|--------|--------|
| Parameters | 1 (total size) | 2 (num elements, size each) |
| Initialization | Uninitialized (garbage) | Initialized to zero |
| Performance | Faster | Slower (zero initialization) |
| Use Case | When you'll initialize immediately | When zero initialization needed |
| Syntax | `malloc(n * sizeof(type))` | `calloc(n, sizeof(type))` |

#### **realloc() - Resize Allocation:**

```cpp
void* realloc(void* ptr, size_t new_size);
```

**Characteristics:**
- Resizes previously allocated memory
- May move memory block to new location
- If ptr is NULL, behaves like malloc
- If new_size is 0, behaves like free

**Example:**
```cpp
int* arr = (int*)malloc(5 * sizeof(int));
// Use arr...

// Need more space
arr = (int*)realloc(arr, 10 * sizeof(int));
if (arr == NULL) {
    // Handle error
}
// arr now has space for 10 integers
// First 5 values preserved (if not moved)
// Last 5 values uninitialized

free(arr);
```

#### **Best Practices:**

```cpp
// 1. Always check for NULL
int* ptr = (int*)malloc(size);
if (ptr == NULL) {
    perror("malloc failed");
    return -1;
}

// 2. Use sizeof for portability
int* arr = (int*)malloc(n * sizeof(*arr));  // Better than sizeof(int)

// 3. Avoid memory leaks
int* ptr1 = (int*)malloc(10 * sizeof(int));
ptr1 = (int*)malloc(20 * sizeof(int));  // LEAK! Original memory lost
// Should free(ptr1) before reassigning

// 4. Free memory when done
free(ptr);
ptr = NULL;  // Good practice to avoid dangling pointer

// 5. Don't free twice
free(ptr);
free(ptr);  // UNDEFINED BEHAVIOR!

// 6. Don't use after free
free(ptr);
*ptr = 10;  // UNDEFINED BEHAVIOR!

// 7. Use calloc for clarity when zeroing needed
int* counters = (int*)calloc(100, sizeof(int));  // All zeros

// 8. Check overflow in malloc
size_t n = 1000000000;
int* huge = (int*)malloc(n * sizeof(int));  // May overflow
// Better:
if (n > SIZE_MAX / sizeof(int)) {
    // Handle overflow
}
```

#### **Modern C++ Alternatives:**

```cpp
// Instead of malloc/calloc/free, use:

// 1. new/delete
int* ptr = new int(10);        // Single object
delete ptr;

int* arr = new int[10];        // Array
delete[] arr;

// 2. Smart pointers (preferred)
auto ptr = std::make_unique<int>(10);
auto arr = std::make_unique<int[]>(10);
// Automatic cleanup

// 3. Containers (most preferred)
std::vector<int> vec(10);      // Automatic memory management
std::array<int, 10> arr;       // Stack-based, fixed size
```

#### **Memory Allocation Implementation Details:**

```cpp
// Typical allocator structure
struct Block {
    size_t size;
    bool is_free;
    Block* next;
    // Actual data follows this header
};

// Simple allocator (conceptual)
void* my_malloc(size_t size) {
    // 1. Add header size
    size_t total = size + sizeof(Block);
    
    // 2. Find free block (First Fit strategy)
    Block* block = find_free_block(total);
    
    if (block) {
        block->is_free = false;
        return (void*)(block + 1);  // Return after header
    }
    
    // 3. Request from OS (sbrk/mmap)
    block = request_from_os(total);
    block->size = size;
    block->is_free = false;
    
    return (void*)(block + 1);
}

void my_free(void* ptr) {
    if (!ptr) return;
    
    // Get block header
    Block* block = (Block*)ptr - 1;
    block->is_free = true;
    
    // Coalesce with adjacent free blocks
    coalesce_blocks(block);
}
```

---

## Operating System

### 1. LRU Cache Implementation

**LRU (Least Recently Used) Cache** evicts the least recently accessed item when the cache is full.

#### **Implementation using HashMap + Doubly Linked List:**

```cpp
#include <unordered_map>
#include <list>

template<typename K, typename V>
class LRUCache {
private:
    size_t capacity;
    std::list<std::pair<K, V>> cache_list;  // Doubly linked list
    std::unordered_map<K, typename std::list<std::pair<K, V>>::iterator> cache_map;
    
public:
    LRUCache(size_t cap) : capacity(cap) {}
    
    V get(const K& key) {
        auto it = cache_map.find(key);
        if (it == cache_map.end()) {
            throw std::runtime_error("Key not found");
        }
        
        // Move accessed item to front (most recently used)
        cache_list.splice(cache_list.begin(), cache_list, it->second);
        return it->second->second;
    }
    
    void put(const K& key, const V& value) {
        auto it = cache_map.find(key);
        
        if (it != cache_map.end()) {
            // Key exists, update value and move to front
            it->second->second = value;
            cache_list.splice(cache_list.begin(), cache_list, it->second);
            return;
        }
        
        // Check capacity
        if (cache_list.size() >= capacity) {
            // Remove LRU item (back of list)
            auto lru = cache_list.back();
            cache_map.erase(lru.first);
            cache_list.pop_back();
        }
        
        // Insert new item at front
        cache_list.emplace_front(key, value);
        cache_map[key] = cache_list.begin();
    }
    
    size_t size() const {
        return cache_list.size();
    }
};

// Usage
int main() {
    LRUCache<int, std::string> cache(3);
    
    cache.put(1, "one");
    cache.put(2, "two");
    cache.put(3, "three");
    
    std::cout << cache.get(1) << "\n";  // "one", moves 1 to front
    
    cache.put(4, "four");  // Evicts 2 (LRU)
    
    try {
        cache.get(2);  // Throws exception
    } catch (const std::exception& e) {
        std::cout << "Key 2 not found (evicted)\n";
    }
    
    return 0;
}
```

#### **Time Complexity:**
- get(): O(1)
- put(): O(1)
- Space: O(n) where n is capacity

#### **Alternative: Using std::deque (simpler but less efficient):**

```cpp
template<typename K, typename V>
class SimpleLRUCache {
private:
    size_t capacity;
    std::deque<std::pair<K, V>> cache;
    
public:
    SimpleLRUCache(size_t cap) : capacity(cap) {}
    
    V get(const K& key) {
        for (auto it = cache.begin(); it != cache.end(); ++it) {
            if (it->first == key) {
                V value = it->second;
                cache.erase(it);
                cache.push_front({key, value});
                return value;
            }
        }
        throw std::runtime_error("Key not found");
    }
    
    void put(const K& key, const V& value) {
        // Remove if exists
        for (auto it = cache.begin(); it != cache.end(); ++it) {
            if (it->first == key) {
                cache.erase(it);
                break;
            }
        }
        
        // Add to front
        cache.push_front({key, value});
        
        // Evict if over capacity
        if (cache.size() > capacity) {
            cache.pop_back();
        }
    }
};
```

**Time Complexity (Simple version):**
- get(): O(n)
- put(): O(n)

---

### 2. Virtual Memory, Defragmentation, Thrashing, and Memory Management

#### **Virtual Memory:**

Virtual memory is an abstraction that provides each process with its own address space, independent of physical RAM.

**Key Concepts:**

```
Virtual Address Space (Process View):
+------------------------+ 0xFFFFFFFF
|    Kernel Space        |
+------------------------+ 0xC0000000
|    Stack               | (grows down)
|         ↓              |
|                        |
|    Free Space          |
|                        |
|         ↑              |
|    Heap                | (grows up)
+------------------------+
|    BSS (uninitialized) |
+------------------------+
|    Data (initialized)  |
+------------------------+
|    Text (code)         |
+------------------------+ 0x00000000

Physical Memory:
+------------------------+
|  Process A Pages       |
+------------------------+
|  Process B Pages       |
+------------------------+
|  Free Frames           |
+------------------------+
|  Kernel Pages          |
+------------------------+
```

**Benefits:**
1. **Isolation**: Each process has own address space
2. **More memory than RAM**: Via paging/swapping
3. **Security**: Process can't access other's memory
4. **Flexibility**: Non-contiguous physical memory

**Address Translation:**
```
Virtual Address = Page Number + Offset

Page Table Entry (PTE):
+-------+-----+-----+-----+-----+
| Frame | V   | R   | W   | X   |
+-------+-----+-----+-----+-----+
  ^       ^     ^     ^     ^
  |       |     |     |     |
  |       |     |     |     +-- Executable
  |       |     |     +-------- Writable
  |       |     +-------------- Readable
  |       +-------------------- Valid bit
  +---------------------------- Physical frame number

TLB (Translation Lookaside Buffer):
- Cache for page table entries
- Speeds up address translation
```

#### **Page Replacement Algorithms:**

```cpp
// 1. FIFO (First In First Out)
class FIFO {
    std::queue<int> pages;
    std::unordered_set<int> page_set;
    size_t capacity;
    
public:
    bool access(int page) {
        if (page_set.count(page)) {
            return true;  // Page hit
        }
        
        // Page fault
        if (pages.size() >= capacity) {
            int old = pages.front();
            pages.pop();
            page_set.erase(old);
        }
        
        pages.push(page);
        page_set.insert(page);
        return false;
    }
};

// 2. LRU (Least Recently Used) - Already covered above

// 3. Clock/Second Chance Algorithm
class ClockAlgorithm {
    struct Page {
        int id;
        bool reference_bit;
    };
    
    std::vector<Page> pages;
    size_t clock_hand;
    size_t capacity;
    
public:
    ClockAlgorithm(size_t cap) : capacity(cap), clock_hand(0) {}
    
    void access(int page_id) {
        // Check if page in memory
        for (auto& p : pages) {
            if (p.id == page_id) {
                p.reference_bit = true;
                return;
            }
        }
        
        // Page fault - find victim
        if (pages.size() < capacity) {
            pages.push_back({page_id, true});
            return;
        }
        
        // Clock algorithm to find victim
        while (true) {
            if (!pages[clock_hand].reference_bit) {
                // Replace this page
                pages[clock_hand] = {page_id, true};
                clock_hand = (clock_hand + 1) % capacity;
                break;
            }
            // Give second chance
            pages[clock_hand].reference_bit = false;
            clock_hand = (clock_hand + 1) % capacity;
        }
    }
};
```

#### **Thrashing:**

**Definition**: Excessive paging activity where system spends more time swapping pages than executing processes.

**Causes:**
1. Too many processes in memory
2. Insufficient RAM
3. Poor page replacement algorithm
4. Working set doesn't fit in memory

**Working Set Model:**
```
Working Set (WS) = Set of pages referenced in recent time window Δ

If: Σ(Working Sets) > Available Memory
Then: Thrashing occurs

Example:
Process A: WS = 100 pages
Process B: WS = 80 pages
Process C: WS = 120 pages
Total: 300 pages

If RAM has only 250 pages → Thrashing!
```

**Detection:**
```cpp
struct ThrashingDetector {
    double page_fault_rate_threshold = 0.5;  // 50%
    double cpu_utilization_threshold = 0.2;  // 20%
    
    bool is_thrashing(int page_faults, int memory_accesses, 
                      double cpu_util) {
        double fault_rate = (double)page_faults / memory_accesses;
        
        // High page fault rate AND low CPU utilization
        return (fault_rate > page_fault_rate_threshold && 
                cpu_util < cpu_utilization_threshold);
    }
};
```

**Prevention:**
1. **Working Set Model**: Keep only active working sets in memory
2. **Page Fault Frequency (PFF)**: Allocate frames based on fault rate
3. **Load Control**: Reduce multiprogramming degree
4. **Increase RAM**: Hardware solution

**Mitigation:**
```cpp
// Process suspension to reduce thrashing
void handle_thrashing() {
    if (detect_thrashing()) {
        // 1. Suspend low-priority processes
        suspend_processes(LOW_PRIORITY);
        
        // 2. Increase page allocation for remaining processes
        increase_page_allocation();
        
        // 3. Wait for situation to improve
        wait_for_stabilization();
        
        // 4. Resume processes gradually
        resume_processes_gradually();
    }
}
```

#### **Defragmentation (Memory Compaction):**

**Purpose**: Consolidate free memory by moving allocated blocks together.

**Types:**

**1. Internal Fragmentation:**
- Wasted space within allocated blocks
- Example: Request 10 bytes, allocate 16 bytes → 6 bytes wasted

**2. External Fragmentation:**
- Free memory scattered in small blocks
- Total free memory sufficient, but not contiguous

```
Before Compaction:
+------+------+------+------+------+
| Used | Free | Used | Free | Used |
| 100  | 50   | 200  | 80   | 150  |
+------+------+------+------+------+

After Compaction:
+------+------+------+------+------+
| Used | Used | Used | Free | Free |
| 100  | 200  | 150  | 50   | 80   | 
+------+------+------+------+------+
```

**Compaction Algorithm:**
```cpp
struct MemoryBlock {
    void* address;
    size_t size;
    bool is_allocated;
};

void compact_memory(std::vector<MemoryBlock>& blocks) {
    size_t write_pos = 0;
    
    // Move allocated blocks to front
    for (auto& block : blocks) {
        if (block.is_allocated) {
            if (block.address != (void*)write_pos) {
                // Move block
                memmove((void*)write_pos, block.address, block.size);
                block.address = (void*)write_pos;
            }
            write_pos += block.size;
        }
    }
    
    // Consolidate free space at end
    // Update free space pointer
}
```

**Challenges:**
- Pauses execution (stop-the-world)
- Updates all pointers/references
- Time-consuming for large memory

**Solutions:**
1. **Buddy System**: Reduces fragmentation by power-of-2 allocation
2. **Slab Allocator**: Pre-allocate objects of same size
3. **Paging**: Eliminates external fragmentation (fixed-size pages)

---

### 3. Locks, Semaphores, File Systems, Device Drivers, Deadlock, Race Conditions, Synchronization

#### **Locks (Mutex):**

**Purpose**: Provide mutual exclusion to protect critical sections.

```cpp
#include <mutex>

std::mutex mtx;
int shared_counter = 0;

void increment() {
    mtx.lock();
    shared_counter++;  // Critical section
    mtx.unlock();
}

// Better: Use lock_guard (RAII)
void increment_safe() {
    std::lock_guard<std::mutex> lock(mtx);
    shared_counter++;
    // Automatically unlocks on scope exit
}

// Even better: scoped_lock (C++17)
std::mutex mtx1, mtx2;
void safe_transfer() {
    std::scoped_lock lock(mtx1, mtx2);  // Deadlock-free
    // Access both resources
}
```

**Spinlock** (busy-waiting):
```cpp
class Spinlock {
    std::atomic_flag flag = ATOMIC_FLAG_INIT;
    
public:
    void lock() {
        while (flag.test_and_set(std::memory_order_acquire)) {
            // Spin (busy wait)
        }
    }
    
    void unlock() {
        flag.clear(std::memory_order_release);
    }
};

// Use when lock is held for very short time
// Avoids context switch overhead
```

**Read-Write Lock:**
```cpp
#include <shared_mutex>

std::shared_mutex rw_mtx;
int shared_data = 0;

void read_data() {
    std::shared_lock<std::shared_mutex> lock(rw_mtx);
    // Multiple readers can access simultaneously
    int value = shared_data;
}

void write_data(int value) {
    std::unique_lock<std::shared_mutex> lock(rw_mtx);
    // Exclusive access for writer
    shared_data = value;
}
```

#### **Semaphores:**

**Binary Semaphore** (like mutex):
```cpp
#include <semaphore>

std::binary_semaphore sem(1);  // C++20

void critical_section() {
    sem.acquire();  // P operation (wait)
    // Critical section
    sem.release();  // V operation (signal)
}
```

**Counting Semaphore**:
```cpp
// Limit concurrent access to resource pool
std::counting_semaphore<10> pool_sem(10);  // 10 resources

void use_resource() {
    pool_sem.acquire();  // Get resource
    // Use resource
    pool_sem.release();  // Return resource
}
```

**Classic Problems:**

**1. Producer-Consumer:**
```cpp
std::counting_semaphore<0> items(0);      // Count of items
std::counting_semaphore<BUFFER_SIZE> spaces(BUFFER_SIZE);  // Empty spaces
std::mutex mtx;

void producer() {
    while (true) {
        Item item = produce();
        spaces.acquire();      // Wait for space
        mtx.lock();
        buffer.push(item);
        mtx.unlock();
        items.release();       // Signal item available
    }
}

void consumer() {
    while (true) {
        items.acquire();       // Wait for item
        mtx.lock();
        Item item = buffer.pop();
        mtx.unlock();
        spaces.release();      // Signal space available
        consume(item);
    }
}
```

**2. Readers-Writers:**
```cpp
int readers = 0;
std::mutex mtx;
std::binary_semaphore wrt(1);

void reader() {
    mtx.lock();
    readers++;
    if (readers == 1) {
        wrt.acquire();  // First reader locks writers
    }
    mtx.unlock();
    
    // Read data
    
    mtx.lock();
    readers--;
    if (readers == 0) {
        wrt.release();  // Last reader unlocks writers
    }
    mtx.unlock();
}

void writer() {
    wrt.acquire();
    // Write data
    wrt.release();
}
```

**3. Dining Philosophers:**
```cpp
const int N = 5;
std::binary_semaphore forks[N] = {1, 1, 1, 1, 1};
std::mutex mtx;

void philosopher(int i) {
    while (true) {
        think();
        
        // Pick up forks (avoid deadlock by ordering)
        int first = std::min(i, (i + 1) % N);
        int second = std::max(i, (i + 1) % N);
        
        forks[first].acquire();
        forks[second].acquire();
        
        eat();
        
        forks[first].release();
        forks[second].release();
    }
}
```

#### **Deadlock:**

**Necessary Conditions:**
1. **Mutual Exclusion**: Resources can't be shared
2. **Hold and Wait**: Process holds resources while waiting for more
3. **No Preemption**: Resources can't be forcibly taken
4. **Circular Wait**: Circular chain of processes waiting for resources

**Resource Allocation Graph:**
```
Process P1 → Resource R1 → Process P2 → Resource R2 → Process P1
(Cycle = Deadlock)
```

**Prevention:**

```cpp
// 1. Break circular wait - order resources
void prevent_circular_wait() {
    std::lock(mtx1, mtx2);  // Atomic lock both
    std::lock_guard<std::mutex> l1(mtx1, std::adopt_lock);
    std::lock_guard<std::mutex> l2(mtx2, std::adopt_lock);
}

// 2. Timeout-based acquisition
bool try_acquire_with_timeout() {
    if (mtx.try_lock_for(std::chrono::seconds(1))) {
        // Got lock
        return true;
    }
    // Timeout - back off
    return false;
}

// 3. Resource ordering
class OrderedLock {
    static std::atomic<int> next_id;
    int id;
    std::mutex mtx;
    
public:
    OrderedLock() : id(next_id++) {}
    
    static void lock_multiple(OrderedLock& l1, OrderedLock& l2) {
        if (l1.id < l2.id) {
            l1.mtx.lock();
            l2.mtx.lock();
        } else {
            l2.mtx.lock();
            l1.mtx.lock();
        }
    }
};
```

**Detection:**
```cpp
struct Resource {
    int id;
    std::vector<int> held_by;
    std::vector<int> requested_by;
};

bool has_cycle(std::vector<Resource>& resources) {
    // Build wait-for graph
    std::map<int, std::vector<int>> graph;
    
    for (auto& res : resources) {
        for (int holder : res.held_by) {
            for (int requester : res.requested_by) {
                graph[requester].push_back(holder);
            }
        }
    }
    
    // Detect cycle using DFS
    return detect_cycle_dfs(graph);
}
```

**Recovery:**
- Abort one or more processes
- Preempt resources
- Rollback to safe state (checkpoints)

#### **Race Conditions:**

**Example of Race Condition:**
```cpp
int balance = 1000;

// Thread 1
void withdraw(int amount) {
    if (balance >= amount) {
        // Context switch here!
        balance -= amount;
    }
}

// Thread 2
void withdraw(int amount) {
    if (balance >= amount) {
        balance -= amount;
    }
}

// Both threads may withdraw even if balance < 2 * amount
```

**Solutions:**

```cpp
// 1. Mutex protection
std::mutex mtx;
void withdraw_safe(int amount) {
    std::lock_guard<std::mutex> lock(mtx);
    if (balance >= amount) {
        balance -= amount;
    }
}

// 2. Atomic operations
std::atomic<int> balance{1000};
void withdraw_atomic(int amount) {
    int expected = balance.load();
    while (expected >= amount) {
        if (balance.compare_exchange_weak(expected, expected - amount)) {
            break;
        }
    }
}

// 3. Transaction memory (conceptual)
void withdraw_transactional(int amount) {
    __transaction_atomic {
        if (balance >= amount) {
            balance -= amount;
        }
    }
}
```

**Memory Ordering Issues:**
```cpp
// Without proper synchronization
int data = 0;
bool ready = false;

// Thread 1
void producer() {
    data = 42;
    ready = true;  // May be reordered before data = 42!
}

// Thread 2
void consumer() {
    while (!ready) {}
    std::cout << data;  // May print 0!
}

// Solution: Memory barriers
std::atomic<bool> ready{false};

void producer() {
    data = 42;
    ready.store(true, std::memory_order_release);
}

void consumer() {
    while (!ready.load(std::memory_order_acquire)) {}
    std::cout << data;  // Guaranteed to see 42
}
```

#### **File Systems:**

**Basic Structure:**
```
Disk Layout:
+-------------+-------------+-------------+-------------+
| Boot Block  | Super Block | Inode Table | Data Blocks |
+-------------+-------------+-------------+-------------+

Inode (Index Node):
- File metadata (permissions, timestamps, size)
- Pointers to data blocks
- No filename (stored in directory entries)

Directory Entry:
- Filename
- Inode number
```

**Operations:**
```cpp
// File descriptor operations
int fd = open("file.txt", O_RDWR | O_CREAT, 0644);
write(fd, buffer, size);
read(fd, buffer, size);
lseek(fd, offset, SEEK_SET);
close(fd);

// Memory-mapped I/O
void* addr = mmap(NULL, length, PROT_READ | PROT_WRITE, 
                  MAP_SHARED, fd, 0);
// Access file via memory
munmap(addr, length);

// Synchronization
fsync(fd);  // Flush file to disk
sync();     // Flush all buffers
```

**Buffering:**
```cpp
// User space → Page cache → Disk

// Direct I/O (bypass cache)
int fd = open("file.txt", O_RDWR | O_DIRECT);

// Asynchronous I/O
struct aiocb aio;
aio.aio_fildes = fd;
aio.aio_buf = buffer;
aio.aio_nbytes = size;
aio_read(&aio);
// Do other work
aio_suspend(&aio, 1, NULL);  // Wait for completion
```

#### **Device Drivers:**

**Kernel Module Structure:**
```cpp
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

static int __init driver_init(void) {
    printk(KERN_INFO "Driver loaded\n");
    register_chrdev(MAJOR_NUM, "mydevice", &fops);
    return 0;
}

static void __exit driver_exit(void) {
    printk(KERN_INFO "Driver unloaded\n");
    unregister_chrdev(MAJOR_NUM, "mydevice");
}

module_init(driver_init);
module_exit(driver_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Author");
MODULE_DESCRIPTION("My Device Driver");
```

**File Operations:**
```cpp
struct file_operations fops = {
    .owner = THIS_MODULE,
    .open = device_open,
    .release = device_release,
    .read = device_read,
    .write = device_write,
    .unlocked_ioctl = device_ioctl,
};

static ssize_t device_read(struct file *filp, char __user *buffer,
                           size_t length, loff_t *offset) {
    // Read from hardware
    copy_to_user(buffer, kernel_buffer, length);
    return length;
}

static ssize_t device_write(struct file *filp, const char __user *buffer,
                            size_t length, loff_t *offset) {
    // Write to hardware
    copy_from_user(kernel_buffer, buffer, length);
    return length;
}
```

**Interrupt Handling:**
```cpp
irqreturn_t irq_handler(int irq, void *dev_id) {
    // Top half - minimal processing
    acknowledge_interrupt();
    schedule_work(&bottom_half_work);
    return IRQ_HANDLED;
}

void bottom_half_handler(struct work_struct *work) {
    // Bottom half - heavy processing
    process_data();
}

// Register interrupt
request_irq(IRQ_NUM, irq_handler, IRQF_SHARED, "mydevice", dev_id);
```

---

### 4. Process Management, Semaphores, and Inter-Process Communication

#### **Process Management:**

**Process States:**
```
        +-----+
        | New |
        +-----+
           |
           v
     +----------+
     |  Ready   | <----+
     +----------+      |
           |           |
           v           |
     +----------+      |
     | Running  | -----+
     +----------+
       |     |
       v     v
  +------+ +-------+
  | Wait | | Exit  |
  +------+ +-------+
```

**Process Control Block (PCB):**
```cpp
struct PCB {
    int pid;
    ProcessState state;
    int priority;
    
    // CPU context
    RegisterSet registers;
    ProgramCounter pc;
    StackPointer sp;
    
    // Memory management
    PageTable* page_table;
    
    // I/O status
    std::vector<OpenFile> open_files;
    
    // Scheduling
    int cpu_time;
    int wait_time;
    
    // Parent/child relationships
    int parent_pid;
    std::vector<int> child_pids;
};
```

**Process Creation:**
```cpp
pid_t pid = fork();

if (pid < 0) {
    // Error
    perror("fork failed");
} else if (pid == 0) {
    // Child process
    execl("/bin/ls", "ls", "-l", NULL);
    perror("execl failed");
    exit(1);
} else {
    // Parent process
    int status;
    waitpid(pid, &status, 0);
    printf("Child exited with status %d\n", WEXITSTATUS(status));
}
```

**Process Scheduling Algorithms:**

```cpp
// 1. First Come First Serve (FCFS)
void fcfs_schedule(std::queue<Process>& ready_queue) {
    while (!ready_queue.empty()) {
        Process p = ready_queue.front();
        ready_queue.pop();
        execute(p);
    }
}

// 2. Shortest Job First (SJF)
void sjf_schedule(std::vector<Process>& processes) {
    std::sort(processes.begin(), processes.end(),
        [](const Process& a, const Process& b) {
            return a.burst_time < b.burst_time;
        });
    
    for (auto& p : processes) {
        execute(p);
    }
}

// 3. Round Robin
void round_robin(std::queue<Process>& ready_queue, int time_quantum) {
    while (!ready_queue.empty()) {
        Process p = ready_queue.front();
        ready_queue.pop();
        
        int exec_time = std::min(time_quantum, p.remaining_time);
        execute(p, exec_time);
        p.remaining_time -= exec_time;
        
        if (p.remaining_time > 0) {
            ready_queue.push(p);
        }
    }
}

// 4. Priority Scheduling
struct PriorityProcess {
    int priority;
    Process process;
    
    bool operator<(const PriorityProcess& other) const {
        return priority > other.priority;  // Higher priority first
    }
};

void priority_schedule(std::priority_queue<PriorityProcess>& pq) {
    while (!pq.empty()) {
        PriorityProcess pp = pq.top();
        pq.pop();
        execute(pp.process);
    }
}

// 5. Multilevel Feedback Queue
class MLFQueue {
    std::vector<std::queue<Process>> queues;
    std::vector<int> time_quantums;
    
public:
    void add_process(Process p) {
        queues[0].push(p);  // Start at highest priority
    }
    
    void schedule() {
        for (int i = 0; i < queues.size(); i++) {
            if (!queues[i].empty()) {
                Process p = queues[i].front();
                queues[i].pop();
                
                int exec_time = std::min(time_quantums[i], 
                                        p.remaining_time);
                execute(p, exec_time);
                p.remaining_time -= exec_time;
                
                if (p.remaining_time > 0) {
                    // Move to lower priority queue
                    int next_queue = std::min(i + 1, 
                                            (int)queues.size() - 1);
                    queues[next_queue].push(p);
                }
                return;
            }
        }
    }
};
```

#### **Inter-Process Communication (IPC):**

**1. Pipes:**
```cpp
// Anonymous pipe (related processes)
int pipefd[2];
pipe(pipefd);  // pipefd[0] = read, pipefd[1] = write

if (fork() == 0) {
    // Child - writer
    close(pipefd[0]);
    write(pipefd[1], "Hello", 5);
    close(pipefd[1]);
    exit(0);
} else {
    // Parent - reader
    close(pipefd[1]);
    char buffer[100];
    int n = read(pipefd[0], buffer, sizeof(buffer));
    buffer[n] = '\0';
    close(pipefd[0]);
    wait(NULL);
}

// Named pipe (FIFO) - unrelated processes
mkfifo("/tmp/myfifo", 0666);

// Writer
int fd = open("/tmp/myfifo", O_WRONLY);
write(fd, "Hello", 5);
close(fd);

// Reader
int fd = open("/tmp/myfifo", O_RDONLY);
char buffer[100];
read(fd, buffer, sizeof(buffer));
close(fd);
```

**2. Message Queues:**
```cpp
#include <sys/msg.h>

struct message {
    long msg_type;
    char msg_text[100];
};

// Create/get message queue
int msgid = msgget(IPC_PRIVATE, 0666 | IPC_CREAT);

// Send message
struct message msg;
msg.msg_type = 1;
strcpy(msg.msg_text, "Hello");
msgsnd(msgid, &msg, sizeof(msg.msg_text), 0);

// Receive message
struct message received;
msgrcv(msgid, &received, sizeof(received.msg_text), 1, 0);

// Remove queue
msgctl(msgid, IPC_RMID, NULL);
```

**3. Shared Memory:**
```cpp
#include <sys/shm.h>

// Create shared memory
int shmid = shmget(IPC_PRIVATE, 1024, 0666 | IPC_CREAT);

// Attach to address space
char* shm = (char*)shmat(shmid, NULL, 0);

// Writer
strcpy(shm, "Hello from shared memory");

// Reader
printf("%s\n", shm);

// Detach
shmdt(shm);

// Remove
shmctl(shmid, IPC_RMID, NULL);

// With synchronization
std::mutex* mtx = new (shm) std::mutex();  // Construct in shared memory
mtx->lock();
// Access shared data
mtx->unlock();
```

**4. Sockets:**
```cpp
// Server
int server_fd = socket(AF_INET, SOCK_STREAM, 0);
struct sockaddr_in address;
address.sin_family = AF_INET;
address.sin_addr.s_addr = INADDR_ANY;
address.sin_port = htons(8080);

bind(server_fd, (struct sockaddr*)&address, sizeof(address));
listen(server_fd, 3);

int client_fd = accept(server_fd, NULL, NULL);
char buffer[1024];
read(client_fd, buffer, sizeof(buffer));
write(client_fd, "Hello from server", 17);
close(client_fd);

// Client
int sock = socket(AF_INET, SOCK_STREAM, 0);
struct sockaddr_in serv_addr;
serv_addr.sin_family = AF_INET;
serv_addr.sin_port = htons(8080);
inet_pton(AF_INET, "127.0.0.1", &serv_addr.sin_addr);

connect(sock, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
send(sock, "Hello from client", 17, 0);
read(sock, buffer, sizeof(buffer));
close(sock);
```

**5. Signals:**
```cpp
#include <signal.h>

void signal_handler(int signum) {
    printf("Received signal %d\n", signum);
}

int main() {
    // Register handler
    signal(SIGINT, signal_handler);
    signal(SIGUSR1, signal_handler);
    
    pid_t child_pid = fork();
    
    if (child_pid == 0) {
        // Child waits
        pause();
        exit(0);
    } else {
        // Parent sends signal
        sleep(1);
        kill(child_pid, SIGUSR1);
        wait(NULL);
    }
}
```

**6. Memory-Mapped Files:**
```cpp
#include <sys/mman.h>

// Open file
int fd = open("shared_file.txt", O_RDWR | O_CREAT, 0666);
ftruncate(fd, 1024);  // Set size

// Map to memory
void* addr = mmap(NULL, 1024, PROT_READ | PROT_WRITE, 
                  MAP_SHARED, fd, 0);

// Access like memory
strcpy((char*)addr, "Hello");

// Sync changes
msync(addr, 1024, MS_SYNC);

// Unmap
munmap(addr, 1024);
close(fd);
```

**Comparison:**

| IPC Method | Speed | Complexity | Use Case |
|------------|-------|------------|----------|
| Pipes | Fast | Low | Parent-child communication |
| Message Queues | Medium | Medium | Structured messaging |
| Shared Memory | Fastest | High | Large data exchange |
| Sockets | Medium | Medium | Network/local communication |
| Signals | Fast | Low | Simple notifications |
| Memory-Mapped Files | Fast | Medium | File-based sharing |

---

## AI/ML/LLM

### 1. Training Loop, Transformer Architecture, and System Design

#### **Training Loop:**

The training loop is the core process of machine learning model optimization.

**Basic Training Loop:**
```python
import torch
import torch.nn as nn
import torch.optim as optim

def train_model(model, train_loader, val_loader, epochs, device):
    criterion = nn.CrossEntropyLoss()
    optimizer = optim.Adam(model.parameters(), lr=0.001)
    
    for epoch in range(epochs):
        # Training phase
        model.train()
        train_loss = 0.0
        train_correct = 0
        train_total = 0
        
        for batch_idx, (inputs, targets) in enumerate(train_loader):
            inputs, targets = inputs.to(device), targets.to(device)
            
            # Forward pass
            outputs = model(inputs)
            loss = criterion(outputs, targets)
            
            # Backward pass
            optimizer.zero_grad()  # Clear gradients
            loss.backward()        # Compute gradients
            optimizer.step()       # Update weights
            
            # Metrics
            train_loss += loss.item()
            _, predicted = outputs.max(1)
            train_total += targets.size(0)
            train_correct += predicted.eq(targets).sum().item()
            
            if batch_idx % 100 == 0:
                print(f'Epoch: {epoch}, Batch: {batch_idx}, '
                      f'Loss: {loss.item():.4f}')
        
        # Calculate epoch metrics
        epoch_loss = train_loss / len(train_loader)
        epoch_acc = 100. * train_correct / train_total
        
        # Validation phase
        val_loss, val_acc = validate(model, val_loader, criterion, device)
        
        print(f'Epoch {epoch}: '
              f'Train Loss: {epoch_loss:.4f}, Train Acc: {epoch_acc:.2f}%, '
              f'Val Loss: {val_loss:.4f}, Val Acc: {val_acc:.2f}%')
        
        # Save checkpoint
        if epoch % 5 == 0:
            torch.save({
                'epoch': epoch,
                'model_state_dict': model.state_dict(),
                'optimizer_state_dict': optimizer.state_dict(),
                'train_loss': epoch_loss,
            }, f'checkpoint_epoch_{epoch}.pth')

def validate(model, val_loader, criterion, device):
    model.eval()
    val_loss = 0.0
    correct = 0
    total = 0
    
    with torch.no_grad():
        for inputs, targets in val_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            outputs = model(inputs)
            loss = criterion(outputs, targets)
            
            val_loss += loss.item()
            _, predicted = outputs.max(1)
            total += targets.size(0)
            correct += predicted.eq(targets).sum().item()
    
    return val_loss / len(val_loader), 100. * correct / total
```

**Advanced Training Loop with Mixed Precision and Gradient Accumulation:**
```python
from torch.cuda.amp import autocast, GradScaler

def advanced_training_loop(model, train_loader, epochs, device):
    optimizer = optim.AdamW(model.parameters(), lr=1e-4, weight_decay=0.01)
    scheduler = optim.lr_scheduler.CosineAnnealingLR(optimizer, epochs)
    scaler = GradScaler()  # For mixed precision
    
    accumulation_steps = 4  # Gradient accumulation
    
    for epoch in range(epochs):
        model.train()
        optimizer.zero_grad()
        
        for batch_idx, (inputs, targets) in enumerate(train_loader):
            inputs, targets = inputs.to(device), targets.to(device)
            
            # Mixed precision training
            with autocast():
                outputs = model(inputs)
                loss = criterion(outputs, targets)
                loss = loss / accumulation_steps  # Scale loss
            
            # Backward pass with scaling
            scaler.scale(loss).backward()
            
            # Update weights every N steps
            if (batch_idx + 1) % accumulation_steps == 0:
                # Gradient clipping
                scaler.unscale_(optimizer)
                torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
                
                # Optimizer step
                scaler.step(optimizer)
                scaler.update()
                optimizer.zero_grad()
        
        scheduler.step()  # Update learning rate
```

#### **Transformer Architecture:**

**Core Components:**

**1. Self-Attention Mechanism:**
```python
import torch.nn.functional as F

class SelfAttention(nn.Module):
    def __init__(self, embed_dim, num_heads):
        super().__init__()
        self.embed_dim = embed_dim
        self.num_heads = num_heads
        self.head_dim = embed_dim // num_heads
        
        assert self.head_dim * num_heads == embed_dim
        
        self.q_linear = nn.Linear(embed_dim, embed_dim)
        self.k_linear = nn.Linear(embed_dim, embed_dim)
        self.v_linear = nn.Linear(embed_dim, embed_dim)
        self.out_linear = nn.Linear(embed_dim, embed_dim)
        
    def forward(self, x, mask=None):
        batch_size, seq_len, embed_dim = x.shape
        
        # Linear projections
        Q = self.q_linear(x)  # (batch, seq_len, embed_dim)
        K = self.k_linear(x)
        V = self.v_linear(x)
        
        # Split into multiple heads
        Q = Q.view(batch_size, seq_len, self.num_heads, self.head_dim)
        K = K.view(batch_size, seq_len, self.num_heads, self.head_dim)
        V = V.view(batch_size, seq_len, self.num_heads, self.head_dim)
        
        # Transpose for attention: (batch, num_heads, seq_len, head_dim)
        Q = Q.transpose(1, 2)
        K = K.transpose(1, 2)
        V = V.transpose(1, 2)
        
        # Scaled dot-product attention
        scores = torch.matmul(Q, K.transpose(-2, -1)) / (self.head_dim ** 0.5)
        
        if mask is not None:
            scores = scores.masked_fill(mask == 0, -1e9)
        
        attention_weights = F.softmax(scores, dim=-1)
        attention_output = torch.matmul(attention_weights, V)
        
        # Concatenate heads
        attention_output = attention_output.transpose(1, 2).contiguous()
        attention_output = attention_output.view(batch_size, seq_len, embed_dim)
        
        # Final linear projection
        output = self.out_linear(attention_output)
        
        return output, attention_weights
```

**2. Feed-Forward Network:**
```python
class FeedForward(nn.Module):
    def __init__(self, embed_dim, ff_dim, dropout=0.1):
        super().__init__()
        self.linear1 = nn.Linear(embed_dim, ff_dim)
        self.linear2 = nn.Linear(ff_dim, embed_dim)
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, x):
        x = F.gelu(self.linear1(x))
        x = self.dropout(x)
        x = self.linear2(x)
        return x
```

**3. Transformer Encoder Block:**
```python
class TransformerEncoderBlock(nn.Module):
    def __init__(self, embed_dim, num_heads, ff_dim, dropout=0.1):
        super().__init__()
        self.attention = SelfAttention(embed_dim, num_heads)
        self.norm1 = nn.LayerNorm(embed_dim)
        self.ff = FeedForward(embed_dim, ff_dim, dropout)
        self.norm2 = nn.LayerNorm(embed_dim)
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, x, mask=None):
        # Multi-head attention with residual connection
        attn_output, _ = self.attention(x, mask)
        x = self.norm1(x + self.dropout(attn_output))
        
        # Feed-forward with residual connection
        ff_output = self.ff(x)
        x = self.norm2(x + self.dropout(ff_output))
        
        return x
```

**4. Transformer Decoder Block:**
```python
class TransformerDecoderBlock(nn.Module):
    def __init__(self, embed_dim, num_heads, ff_dim, dropout=0.1):
        super().__init__()
        self.self_attention = SelfAttention(embed_dim, num_heads)
        self.cross_attention = SelfAttention(embed_dim, num_heads)
        self.ff = FeedForward(embed_dim, ff_dim, dropout)
        
        self.norm1 = nn.LayerNorm(embed_dim)
        self.norm2 = nn.LayerNorm(embed_dim)
        self.norm3 = nn.LayerNorm(embed_dim)
        self.dropout = nn.Dropout(dropout)
        
    def forward(self, x, encoder_output, src_mask=None, tgt_mask=None):
        # Masked self-attention
        attn_output, _ = self.self_attention(x, tgt_mask)
        x = self.norm1(x + self.dropout(attn_output))
        
        # Cross-attention with encoder output
        cross_attn, _ = self.cross_attention(x, encoder_output, src_mask)
        x = self.norm2(x + self.dropout(cross_attn))
        
        # Feed-forward
        ff_output = self.ff(x)
        x = self.norm3(x + self.dropout(ff_output))
        
        return x
```

**5. Complete Transformer:**
```python
class Transformer(nn.Module):
    def __init__(self, vocab_size, embed_dim, num_heads, num_layers, 
                 ff_dim, max_seq_len, dropout=0.1):
        super().__init__()
        
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        self.pos_encoding = PositionalEncoding(embed_dim, max_seq_len)
        
        self.encoder_layers = nn.ModuleList([
            TransformerEncoderBlock(embed_dim, num_heads, ff_dim, dropout)
            for _ in range(num_layers)
        ])
        
        self.decoder_layers = nn.ModuleList([
            TransformerDecoderBlock(embed_dim, num_heads, ff_dim, dropout)
            for _ in range(num_layers)
        ])
        
        self.output_linear = nn.Linear(embed_dim, vocab_size)
        
    def forward(self, src, tgt, src_mask=None, tgt_mask=None):
        # Encoder
        src_embed = self.pos_encoding(self.embedding(src))
        encoder_output = src_embed
        for layer in self.encoder_layers:
            encoder_output = layer(encoder_output, src_mask)
        
        # Decoder
        tgt_embed = self.pos_encoding(self.embedding(tgt))
        decoder_output = tgt_embed
        for layer in self.decoder_layers:
            decoder_output = layer(decoder_output, encoder_output, 
                                  src_mask, tgt_mask)
        
        # Output projection
        output = self.output_linear(decoder_output)
        return output

class PositionalEncoding(nn.Module):
    def __init__(self, embed_dim, max_seq_len):
        super().__init__()
        pe = torch.zeros(max_seq_len, embed_dim)
        position = torch.arange(0, max_seq_len).unsqueeze(1)
        div_term = torch.exp(torch.arange(0, embed_dim, 2) * 
                            -(torch.log(torch.tensor(10000.0)) / embed_dim))
        
        pe[:, 0::2] = torch.sin(position * div_term)
        pe[:, 1::2] = torch.cos(position * div_term)
        
        self.register_buffer('pe', pe.unsqueeze(0))
        
    def forward(self, x):
        return x + self.pe[:, :x.size(1)]
```

#### **Encoder vs Decoder-Based Models:**

**Encoder-only (BERT-style):**
- Bidirectional context
- Use cases: Classification, NER, Q&A
- Masked language modeling objective

```python
class BERTModel(nn.Module):
    def __init__(self, vocab_size, embed_dim, num_heads, num_layers):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        self.encoders = nn.ModuleList([
            TransformerEncoderBlock(embed_dim, num_heads, embed_dim * 4)
            for _ in range(num_layers)
        ])
        self.mlm_head = nn.Linear(embed_dim, vocab_size)
        
    def forward(self, input_ids, attention_mask=None):
        x = self.embedding(input_ids)
        for encoder in self.encoders:
            x = encoder(x, attention_mask)
        return self.mlm_head(x)
```

**Decoder-only (GPT-style):**
- Unidirectional (causal) context
- Use cases: Text generation, completion
- Autoregressive language modeling

```python
class GPTModel(nn.Module):
    def __init__(self, vocab_size, embed_dim, num_heads, num_layers):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, embed_dim)
        self.decoders = nn.ModuleList([
            TransformerDecoderBlock(embed_dim, num_heads, embed_dim * 4)
            for _ in range(num_layers)
        ])
        self.lm_head = nn.Linear(embed_dim, vocab_size)
        
    def forward(self, input_ids):
        seq_len = input_ids.size(1)
        causal_mask = torch.tril(torch.ones(seq_len, seq_len))
        
        x = self.embedding(input_ids)
        for decoder in self.decoders:
            x = decoder(x, None, None, causal_mask)
        return self.lm_head(x)
```

**Encoder-Decoder (T5-style):**
- Full transformer architecture
- Use cases: Translation, summarization
- Seq2seq tasks

#### **System Design for LLM Serving:**

```
Architecture:

┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────┐
│   API Gateway / Load Balancer│
└──────────┬──────────────────┘
           │
     ┌─────┴─────┐
     ▼           ▼
┌─────────┐ ┌─────────┐
│ Server1 │ │ Server2 │ ...
└────┬────┘ └────┬────┘
     │           │
     ▼           ▼
┌─────────────────────┐
│ Model Cache (Redis) │
└─────────────────────┘
           │
           ▼
┌─────────────────────┐
│ Model Storage (S3)  │
└─────────────────────┘

Components:
1. Request preprocessing
2. Token batching for efficiency
3. Model inference
4. Response streaming
5. Caching frequent requests
6. Rate limiting
```

**Key Considerations:**
- **Batching**: Combine multiple requests for GPU efficiency
- **KV Cache**: Cache key-value pairs for faster generation
- **Quantization**: 4-bit/8-bit quantization for memory efficiency
- **Model Parallelism**: Split model across GPUs
- **Streaming**: Stream tokens as generated
- **Scaling**: Horizontal scaling with load balancer

---

### 2. Langchain and Langsmith

#### **Langchain:**

Langchain is a framework for building applications with LLMs, providing abstractions for:
- Prompts
- Chains
- Agents
- Memory
- Retrieval

**Basic Usage:**

```python
from langchain.llms import OpenAI
from langchain.prompts import PromptTemplate
from langchain.chains import LLMChain

# 1. Simple LLM call
llm = OpenAI(temperature=0.7)
response = llm("What is the capital of France?")

# 2. Prompt template
template = """
You are a helpful assistant. Answer the following question:
Question: {question}
Answer:"""

prompt = PromptTemplate(template=template, input_variables=["question"])

# 3. Chain
chain = LLMChain(llm=llm, prompt=prompt)
result = chain.run(question="What is machine learning?")
```

**Advanced Patterns:**

**Sequential Chains:**
```python
from langchain.chains import SimpleSequentialChain

# Chain 1: Generate synopsis
synopsis_chain = LLMChain(
    llm=llm,
    prompt=PromptTemplate(
        template="Write a synopsis for: {title}",
        input_variables=["title"]
    )
)

# Chain 2: Write review
review_chain = LLMChain(
    llm=llm,
    prompt=PromptTemplate(
        template="Write a review for this synopsis:\n{synopsis}",
        input_variables=["synopsis"]
    )
)

# Combine chains
overall_chain = SimpleSequentialChain(
    chains=[synopsis_chain, review_chain]
)

result = overall_chain.run("The Matrix")
```

**RAG (Retrieval Augmented Generation):**
```python
from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import Chroma
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.chains import RetrievalQA
from langchain.document_loaders import TextLoader

# Load documents
loader = TextLoader("document.txt")
documents = loader.load()

# Split into chunks
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200
)
chunks = text_splitter.split_documents(documents)

# Create embeddings and vector store
embeddings = OpenAIEmbeddings()
vectorstore = Chroma.from_documents(chunks, embeddings)

# Create retrieval chain
qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    chain_type="stuff",
    retriever=vectorstore.as_retriever(search_kwargs={"k": 3}),
    return_source_documents=True
)

# Query
result = qa_chain({"query": "What is the main topic?"})
print(result["result"])
print(result["source_documents"])
```

**Agents:**
```python
from langchain.agents import initialize_agent, Tool
from langchain.agents import AgentType
from langchain.tools import DuckDuckGoSearchRun

# Define tools
search = DuckDuckGoSearchRun()
tools = [
    Tool(
        name="Search",
        func=search.run,
        description="Useful for searching the internet"
    ),
    Tool(
        name="Calculator",
        func=lambda x: eval(x),
        description="Useful for math calculations"
    )
]

# Initialize agent
agent = initialize_agent(
    tools,
    llm,
    agent=AgentType.ZERO_SHOT_REACT_DESCRIPTION,
    verbose=True
)

# Run agent
result = agent.run("What is 10 * 25 + 50?")
```

**Memory:**
```python
from langchain.memory import ConversationBufferMemory
from langchain.chains import ConversationChain

memory = ConversationBufferMemory()
conversation = ConversationChain(
    llm=llm,
    memory=memory,
    verbose=True
)

response1 = conversation.predict(input="Hi, I'm Alice")
response2 = conversation.predict(input="What's my name?")
# "Your name is Alice"
```

#### **Langsmith:**

Langsmith is a platform for debugging, testing, and monitoring LLM applications.

**Key Features:**
1. **Tracing**: Track all LLM calls and chains
2. **Evaluation**: Test prompts and chains
3. **Monitoring**: Production monitoring
4. **Datasets**: Manage test datasets

**Setup:**
```python
import os
from langsmith import Client

os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your-api-key"
os.environ["LANGCHAIN_PROJECT"] = "my-project"

client = Client()
```

**Tracing:**
```python
from langchain.callbacks.tracers import LangChainTracer

tracer = LangChainTracer(project_name="my-project")

# Chains automatically traced
chain.run(
    {"question": "What is AI?"},
    callbacks=[tracer]
)

# View traces in Langsmith UI
```

**Evaluation:**
```python
from langsmith import evaluate

dataset_name = "qa-dataset"

# Create dataset
examples = [
    {"question": "What is 2+2?", "answer": "4"},
    {"question": "What is the capital of France?", "answer": "Paris"}
]

client.create_dataset(dataset_name, description="QA pairs")
for ex in examples:
    client.create_example(
        inputs={"question": ex["question"]},
        outputs={"answer": ex["answer"]},
        dataset_name=dataset_name
    )

# Evaluate
def predict(inputs):
    return {"answer": chain.run(inputs["question"])}

def score(outputs, reference_outputs):
    return {"correct": outputs["answer"] == reference_outputs["answer"]}

results = evaluate(
    predict,
    data=dataset_name,
    evaluators=[score],
    experiment_prefix="test-run"
)

print(f"Accuracy: {results['correct'].mean()}")
```

**Custom Evaluators:**
```python
from langchain.evaluation import StringEvaluator

class RelevanceEvaluator(StringEvaluator):
    def __init__(self):
        self.llm = OpenAI()
        
    def evaluate_strings(self, prediction, reference, input):
        prompt = f"""
        Question: {input}
        Expected: {reference}
        Actual: {prediction}
        
        Is the actual answer relevant? (Yes/No)
        """
        result = self.llm(prompt).strip().lower()
        return {"score": 1 if "yes" in result else 0}

evaluator = RelevanceEvaluator()
results = evaluate(
    predict,
    data=dataset_name,
    evaluators=[evaluator]
)
```

---

## Computer Vision

### 1. Anomaly Detection

Anomaly detection identifies unusual patterns that don't conform to expected behavior.

#### **Statistical Methods:**

**1. Z-Score Method:**
```python
import numpy as np

def zscore_anomaly_detection(data, threshold=3):
    mean = np.mean(data)
    std = np.std(data)
    z_scores = np.abs((data - mean) / std)
    anomalies = z_scores > threshold
    return anomalies

# Usage
data = np.random.normal(0, 1, 1000)
data[50] = 10  # Insert anomaly
anomalies = zscore_anomaly_detection(data)
```

**2. Isolation Forest:**
```python
from sklearn.ensemble import IsolationForest

detector = IsolationForest(contamination=0.1, random_state=42)
predictions = detector.fit_predict(data.reshape(-1, 1))
anomalies = predictions == -1
```

#### **Deep Learning Methods:**

**1. Autoencoder:**
```python
import torch
import torch.nn as nn

class Autoencoder(nn.Module):
    def __init__(self, input_dim, encoding_dim):
        super().__init__()
        # Encoder
        self.encoder = nn.Sequential(
            nn.Linear(input_dim, 128),
            nn.ReLU(),
            nn.Linear(128, 64),
            nn.ReLU(),
            nn.Linear(64, encoding_dim)
        )
        
        # Decoder
        self.decoder = nn.Sequential(
            nn.Linear(encoding_dim, 64),
            nn.ReLU(),
            nn.Linear(64, 128),
            nn.ReLU(),
            nn.Linear(128, input_dim),
            nn.Sigmoid()
        )
        
    def forward(self, x):
        encoded = self.encoder(x)
        decoded = self.decoder(encoded)
        return decoded

# Training
autoencoder = Autoencoder(input_dim=784, encoding_dim=32)
criterion = nn.MSELoss()
optimizer = torch.optim.Adam(autoencoder.parameters(), lr=0.001)

# Train on normal data only
for epoch in range(100):
    for batch in normal_data_loader:
        optimizer.zero_grad()
        reconstructed = autoencoder(batch)
        loss = criterion(reconstructed, batch)
        loss.backward()
        optimizer.step()

# Detection
def detect_anomaly(image, threshold=0.05):
    with torch.no_grad():
        reconstructed = autoencoder(image)
        reconstruction_error = torch.mean((image - reconstructed) ** 2)
        return reconstruction_error > threshold
```

**2. Variational Autoencoder (VAE):**
```python
class VAE(nn.Module):
    def __init__(self, input_dim, latent_dim):
        super().__init__()
        # Encoder
        self.fc1 = nn.Linear(input_dim, 256)
        self.fc_mu = nn.Linear(256, latent_dim)
        self.fc_logvar = nn.Linear(256, latent_dim)
        
        # Decoder
        self.fc2 = nn.Linear(latent_dim, 256)
        self.fc3 = nn.Linear(256, input_dim)
        
    def encode(self, x):
        h = F.relu(self.fc1(x))
        return self.fc_mu(h), self.fc_logvar(h)
    
    def reparameterize(self, mu, logvar):
        std = torch.exp(0.5 * logvar)
        eps = torch.randn_like(std)
        return mu + eps * std
    
    def decode(self, z):
        h = F.relu(self.fc2(z))
        return torch.sigmoid(self.fc3(h))
    
    def forward(self, x):
        mu, logvar = self.encode(x)
        z = self.reparameterize(mu, logvar)
        return self.decode(z), mu, logvar

def vae_loss(reconstructed, original, mu, logvar):
    # Reconstruction loss
    recon_loss = F.binary_cross_entropy(reconstructed, original, reduction='sum')
    
    # KL divergence
    kl_loss = -0.5 * torch.sum(1 + logvar - mu.pow(2) - logvar.exp())
    
    return recon_loss + kl_loss
```

**3. One-Class SVM:**
```python
from sklearn.svm import OneClassSVM

# Train on normal data
oc_svm = OneClassSVM(kernel='rbf', nu=0.1)
oc_svm.fit(normal_training_data)

# Predict (-1 for anomaly, 1 for normal)
predictions = oc_svm.predict(test_data)
anomalies = predictions == -1
```

#### **Image Anomaly Detection:**

**ConvAutoencoder:**
```python
class ConvAutoencoder(nn.Module):
    def __init__(self):
        super().__init__()
        # Encoder
        self.encoder = nn.Sequential(
            nn.Conv2d(3, 64, kernel_size=3, stride=2, padding=1),
            nn.ReLU(),
            nn.Conv2d(64, 128, kernel_size=3, stride=2, padding=1),
            nn.ReLU(),
            nn.Conv2d(128, 256, kernel_size=3, stride=2, padding=1),
            nn.ReLU()
        )
        
        # Decoder
        self.decoder = nn.Sequential(
            nn.ConvTranspose2d(256, 128, kernel_size=3, stride=2, 
                              padding=1, output_padding=1),
            nn.ReLU(),
            nn.ConvTranspose2d(128, 64, kernel_size=3, stride=2, 
                              padding=1, output_padding=1),
            nn.ReLU(),
            nn.ConvTranspose2d(64, 3, kernel_size=3, stride=2, 
                              padding=1, output_padding=1),
            nn.Sigmoid()
        )
        
    def forward(self, x):
        encoded = self.encoder(x)
        decoded = self.decoder(encoded)
        return decoded

# Anomaly detection
def detect_image_anomaly(image, model, threshold=0.1):
    with torch.no_grad():
        reconstructed = model(image.unsqueeze(0))
        error = torch.mean((image - reconstructed[0]) ** 2)
        
        # Compute anomaly map
        anomaly_map = torch.mean((image - reconstructed[0]) ** 2, dim=0)
        
        return error > threshold, anomaly_map
```

**Feature-based Anomaly Detection:**
```python
from torchvision import models
from sklearn.neighbors import LocalOutlierFactor

# Extract features using pre-trained network
feature_extractor = models.resnet50(pretrained=True)
feature_extractor.fc = nn.Identity()  # Remove final layer
feature_extractor.eval()

def extract_features(images):
    with torch.no_grad():
        features = feature_extractor(images)
    return features.numpy()

# Detect anomalies in feature space
features = extract_features(normal_images)
lof = LocalOutlierFactor(n_neighbors=20)
lof.fit(features)

test_features = extract_features(test_images)
anomaly_scores = lof.score_samples(test_features)
```

---

### 2. YOLO (You Only Look Once)

YOLO is a real-time object detection algorithm that frames detection as a regression problem.

#### **Architecture Evolution:**

**YOLOv1 → YOLOv3 → YOLOv5 → YOLOv8 → YOLOv11 (latest)**

#### **Core Concept:**

```
Input Image
     ↓
Single CNN
     ↓
Grid cells (S×S)
     ↓
Each cell predicts:
- B bounding boxes
- Confidence scores
- C class probabilities
     ↓
Output: Detections
```

#### **YOLOv5 Implementation:**

```python
import torch

# Install: pip install ultralytics

from ultralytics import YOLO

# 1. Load pre-trained model
model = YOLO('yolov5s.pt')  # Small model

# 2. Inference
results = model('image.jpg')

# 3. Process results
for result in results:
    boxes = result.boxes  # Bounding boxes
    for box in boxes:
        x1, y1, x2, y2 = box.xyxy[0]  # Coordinates
        confidence = box.conf[0]       # Confidence
        class_id = box.cls[0]          # Class ID
        class_name = model.names[int(class_id)]
        
        print(f'{class_name}: {confidence:.2f} at ({x1}, {y1}, {x2}, {y2})')

# 4. Show results
result.show()
result.save('output.jpg')
```

#### **Training Custom YOLO:**

```python
from ultralytics import YOLO

# 1. Prepare dataset in YOLO format
"""
dataset/
  images/
    train/
    val/
  labels/
    train/
    val/
  data.yaml
"""

# data.yaml content:
"""
train: dataset/images/train
val: dataset/images/val
nc: 80  # number of classes
names: ['person', 'car', ...]  # class names
"""

# 2. Train model
model = YOLO('yolov5s.pt')

results = model.train(
    data='data.yaml',
    epochs=100,
    imgsz=640,
    batch=16,
    name='my_model',
    pretrained=True,
    optimizer='Adam',
    lr0=0.01,
    device=0  # GPU
)

# 3. Validate
metrics = model.val()
print(f"mAP50: {metrics.box.map50}")
print(f"mAP50-95: {metrics.box.map}")

# 4. Export
model.export(format='onnx')  # ONNX, TensorRT, CoreML, etc.
```

#### **YOLO Architecture Details:**

**Backbone:**
- CSPDarknet53 (YOLOv5)
- Extracts features at multiple scales

**Neck:**
- FPN (Feature Pyramid Network)
- PAN (Path Aggregation Network)
- Merges features from different scales

**Head:**
- Detection head
- Predicts boxes, confidence, classes

```python
# Simplified YOLOv5 Head
class DetectionHead(nn.Module):
    def __init__(self, nc=80, anchors=()):
        super().__init__()
        self.nc = nc  # Number of classes
        self.no = nc + 5  # Outputs per anchor (x, y, w, h, conf, ...classes)
        self.na = len(anchors)  # Number of anchors
        
        self.m = nn.Conv2d(128, self.no * self.na, 1)  # Prediction conv
        
    def forward(self, x):
        bs, _, ny, nx = x.shape  # Batch, channels, height, width
        
        x = self.m(x)
        x = x.view(bs, self.na, self.no, ny, nx).permute(0, 1, 3, 4, 2)
        
        if not self.training:
            # Inference mode
            xy = torch.sigmoid(x[..., 0:2])  # Center coordinates
            wh = torch.exp(x[..., 2:4])       # Width, height
            conf = torch.sigmoid(x[..., 4:5]) # Confidence
            cls = torch.sigmoid(x[..., 5:])   # Classes
            
            return torch.cat((xy, wh, conf, cls), -1)
        
        return x
```

#### **Loss Function:**

```python
def yolo_loss(predictions, targets, anchors):
    # predictions: (batch, anchors, grid_h, grid_w, 5+nc)
    # targets: (num_targets, 6)  # [batch_idx, class, x, y, w, h]
    
    # 1. Objectness loss (binary cross entropy)
    obj_mask = ...  # Cells with objects
    obj_loss = BCE(predictions[..., 4], obj_mask)
    
    # 2. Box regression loss (IoU/GIoU/DIoU/CIoU)
    pred_boxes = decode_boxes(predictions[..., :4], anchors)
    target_boxes = targets[:, 2:6]
    box_loss = 1 - bbox_iou(pred_boxes, target_boxes)
    
    # 3. Classification loss
    cls_loss = BCE(predictions[..., 5:], target_classes)
    
    # Total loss
    loss = obj_loss + box_loss + cls_loss
    return loss
```

#### **Post-processing (NMS):**

```python
def non_max_suppression(predictions, conf_threshold=0.25, 
                       iou_threshold=0.45, max_det=300):
    output = []
    
    for pred in predictions:  # Per image
        # Filter by confidence
        pred = pred[pred[:, 4] > conf_threshold]
        
        if not pred.shape[0]:
            continue
        
        # Multiply confidence by class probability
        pred[:, 5:] *= pred[:, 4:5]
        
        # Box coordinates (center to corner)
        box = xywh2xyxy(pred[:, :4])
        
        # Best class for each detection
        conf, cls = pred[:, 5:].max(1, keepdim=True)
        pred = torch.cat((box, conf, cls.float()), 1)
        
        # Filter by class
        pred = pred[conf.view(-1) > conf_threshold]
        
        # NMS
        keep = torchvision.ops.nms(pred[:, :4], pred[:, 4], iou_threshold)
        output.append(pred[keep[:max_det]])
    
    return output
```

#### **Real-time Inference:**

```python
import cv2

# Video processing
cap = cv2.VideoCapture(0)  # Webcam
model = YOLO('yolov5s.pt')

while True:
    ret, frame = cap.read()
    if not ret:
        break
    
    # Inference
    results = model(frame, verbose=False)
    
    # Draw boxes
    annotated = results[0].plot()
    
    # Display
    cv2.imshow('YOLO', annotated)
    
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

cap.release()
cv2.destroyAllWindows()
```

#### **Optimizations:**

**1. TensorRT Acceleration:**
```python
model.export(format='engine', half=True)  # FP16
model = YOLO('model.engine')
```

**2. Batch Inference:**
```python
images = ['img1.jpg', 'img2.jpg', 'img3.jpg']
results = model(images, batch=True)
```

**3. Model Pruning:**
```python
# Use smaller model
model = YOLO('yolov5n.pt')  # Nano (fastest)
# yolov5s.pt - Small
# yolov5m.pt - Medium
# yolov5l.pt - Large
# yolov5x.pt - XLarge (most accurate)
```

---

## Summary

This comprehensive guide covers:
- **Go**: Concurrency primitives, memory/CPU management
- **C++**: OOP concepts, memory management, optimization
- **OS**: Caching, virtual memory, synchronization, IPC
- **AI/ML**: Training loops, transformers, Langchain
- **Computer Vision**: Anomaly detection, YOLO

All topics include theory, implementation examples, and best practices for technical interviews.
