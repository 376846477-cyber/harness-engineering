# C++并发规范详细参考

## 1. std::thread 基础

### 1.1 创建与管理

```cpp
#include <thread>
#include <iostream>

void worker(int id) {
    std::cout << "Worker " << id << " running\n";
}

void example_thread() {
    std::thread t1(worker, 1);
    std::thread t2([](int x) {
        std::cout << "Lambda thread: " << x << "\n";
    }, 42);
    
    if (t1.joinable()) {
        t1.join();
    }
    if (t2.joinable()) {
        t2.join();
    }
}
```

### 1.2 std::jthread (C++20) - 推荐使用

```cpp
#include <thread>

void task(std::stop_token token, int id) {
    while (!token.stop_requested()) {
        std::cout << "Task " << id << " working\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
    std::cout << "Task " << id << " stopped\n";
}

void example_jthread() {
    std::jthread t(task, 1);
    std::this_thread::sleep_for(std::chrono::milliseconds(500));
    t.request_stop();
}
```

### 1.3 传递引用参数

```cpp
#include <thread>
#include <functional>

void modify(int& value) {
    value *= 2;
}

void example_ref() {
    int data = 10;
    std::thread t(modify, std::ref(data));
    t.join();
}
```

## 2. 互斥锁 (Mutex)

### 2.1 基本互斥锁

```cpp
#include <mutex>
#include <vector>

class ThreadSafeCounter {
private:
    std::mutex mtx_;
    int count_ = 0;
    
public:
    void increment() {
        std::lock_guard<std::mutex> lock(mtx_);
        ++count_;
    }
    
    int get() const {
        std::lock_guard<std::mutex> lock(mtx_);
        return count_;
    }
};
```

### 2.2 std::lock_guard - RAII锁管理

```cpp
#include <mutex>
#include <list>

template<typename T>
class ThreadSafeList {
private:
    std::mutex mtx_;
    std::list<T> data_;
    
public:
    void push_back(const T& value) {
        std::lock_guard<std::mutex> lock(mtx_);
        data_.push_back(value);
    }
    
    bool empty() const {
        std::lock_guard<std::mutex> lock(mtx_);
        return data_.empty();
    }
};
```

### 2.3 std::unique_lock - 灵活锁管理

```cpp
#include <mutex>
#include <condition_variable>

class BoundedBuffer {
private:
    std::mutex mtx_;
    std::condition_variable not_full_;
    std::condition_variable not_empty_;
    std::vector<int> buffer_;
    size_t head_ = 0, tail_ = 0, count_ = 0;
    
public:
    BoundedBuffer(size_t capacity) : buffer_(capacity) {}
    
    void put(int value) {
        std::unique_lock<std::mutex> lock(mtx_);
        not_full_.wait(lock, [this] { return count_ < buffer_.size(); });
        buffer_[tail_] = value;
        tail_ = (tail_ + 1) % buffer_.size();
        ++count_;
        not_empty_.notify_one();
    }
    
    int take() {
        std::unique_lock<std::mutex> lock(mtx_);
        not_empty_.wait(lock, [this] { return count_ > 0; });
        int value = buffer_[head_];
        head_ = (head_ + 1) % buffer_.size();
        --count_;
        not_full_.notify_one();
        return value;
    }
};
```

### 2.4 std::scoped_lock (C++17) - 多锁同时获取

```cpp
#include <mutex>

class BankAccount {
public:
    std::mutex mtx_;
    int balance_;
    
    BankAccount(int b) : balance_(b) {}
};

void transfer(BankAccount& from, BankAccount& to, int amount) {
    std::scoped_lock lock(from.mtx_, to.mtx_);
    from.balance_ -= amount;
    to.balance_ += amount;
}
```

### 2.5 std::shared_mutex (C++17) - 读写锁

```cpp
#include <shared_mutex>
#include <map>
#include <string>

class ThreadSafeMap {
private:
    mutable std::shared_mutex mtx_;
    std::map<std::string, int> data_;
    
public:
    int get(const std::string& key) const {
        std::shared_lock<std::shared_mutex> lock(mtx_);
        auto it = data_.find(key);
        return (it != data_.end()) ? it->second : -1;
    }
    
    void set(const std::string& key, int value) {
        std::unique_lock<std::shared_mutex> lock(mtx_);
        data_[key] = value;
    }
    
    bool contains(const std::string& key) const {
        std::shared_lock<std::shared_mutex> lock(mtx_);
        return data_.count(key) > 0;
    }
};
```

## 3. std::atomic 与内存序

### 3.1 基本原子操作

```cpp
#include <atomic>

class AtomicCounter {
private:
    std::atomic<int> count_{0};
    
public:
    void increment() {
        count_.fetch_add(1, std::memory_order_relaxed);
    }
    
    void decrement() {
        count_.fetch_sub(1, std::memory_order_relaxed);
    }
    
    int get() const {
        return count_.load(std::memory_order_relaxed);
    }
};
```

### 3.2 内存序详解

```cpp
#include <atomic>
#include <thread>

std::atomic<bool> ready{false};
std::atomic<int> data{0};

void producer() {
    data.store(42, std::memory_order_release);
    ready.store(true, std::memory_order_release);
}

void consumer() {
    while (!ready.load(std::memory_order_acquire)) {
        std::this_thread::yield();
    }
    int value = data.load(std::memory_order_acquire);
}
```

### 3.3 比较交换 (CAS)

```cpp
#include <atomic>

class LockFreeStack {
private:
    struct Node {
        int value;
        Node* next;
    };
    
    std::atomic<Node*> head_{nullptr};
    
public:
    void push(int value) {
        Node* new_node = new Node{value, nullptr};
        new_node->next = head_.load(std::memory_order_relaxed);
        while (!head_.compare_exchange_weak(
            new_node->next, new_node,
            std::memory_order_release,
            std::memory_order_relaxed)) {
        }
    }
};
```

### 3.4 内存序选择指南

| 内存序 | 用途 | 性能 |
|--------|------|------|
| relaxed | 简单计数器、无依赖操作 | 最高 |
| acquire/release | 生产者-消费者、标志位 | 中等 |
| seq_cst | 默认、多线程复杂同步 | 最低 |

## 4. std::condition_variable

### 4.1 生产者-消费者模式

```cpp
#include <condition_variable>
#include <mutex>
#include <queue>

template<typename T>
class BlockingQueue {
private:
    std::mutex mtx_;
    std::condition_variable cv_;
    std::queue<T> queue_;
    size_t max_size_;
    bool stopped_ = false;
    
public:
    BlockingQueue(size_t max_size) : max_size_(max_size) {}
    
    bool push(T value) {
        std::unique_lock<std::mutex> lock(mtx_);
        cv_.wait(lock, [this] { 
            return queue_.size() < max_size_ || stopped_; 
        });
        if (stopped_) return false;
        queue_.push(std::move(value));
        cv_.notify_one();
        return true;
    }
    
    bool pop(T& value) {
        std::unique_lock<std::mutex> lock(mtx_);
        cv_.wait(lock, [this] { 
            return !queue_.empty() || stopped_; 
        });
        if (queue_.empty()) return false;
        value = std::move(queue_.front());
        queue_.pop();
        cv_.notify_one();
        return true;
    }
    
    void stop() {
        std::lock_guard<std::mutex> lock(mtx_);
        stopped_ = true;
        cv_.notify_all();
    }
};
```

### 4.2 线程池示例

```cpp
#include <condition_variable>
#include <mutex>
#include <queue>
#include <vector>
#include <thread>
#include <functional>

class ThreadPool {
private:
    std::vector<std::thread> workers_;
    std::queue<std::function<void()>> tasks_;
    std::mutex mtx_;
    std::condition_variable cv_;
    bool stop_ = false;
    
public:
    ThreadPool(size_t num_threads) {
        for (size_t i = 0; i < num_threads; ++i) {
            workers_.emplace_back([this] {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(mtx_);
                        cv_.wait(lock, [this] { 
                            return stop_ || !tasks_.empty(); 
                        });
                        if (stop_ && tasks_.empty()) return;
                        task = std::move(tasks_.front());
                        tasks_.pop();
                    }
                    task();
                }
            });
        }
    }
    
    template<typename F>
    void enqueue(F&& task) {
        {
            std::lock_guard<std::mutex> lock(mtx_);
            tasks_.emplace(std::forward<F>(task));
        }
        cv_.notify_one();
    }
    
    ~ThreadPool() {
        {
            std::lock_guard<std::mutex> lock(mtx_);
            stop_ = true;
        }
        cv_.notify_all();
        for (auto& worker : workers_) {
            worker.join();
        }
    }
};
```

## 5. std::future, std::promise, std::async

### 5.1 std::async 异步执行

```cpp
#include <future>
#include <iostream>

int compute(int x) {
    return x * x;
}

void example_async() {
    std::future<int> fut = std::async(std::launch::async, compute, 10);
    std::cout << "Result: " << fut.get() << "\n";
}
```

### 5.2 std::promise 手动设置结果

```cpp
#include <future>
#include <thread>

void worker(std::promise<int>& prom) {
    try {
        int result = 42;
        prom.set_value(result);
    } catch (...) {
        prom.set_exception(std::current_exception());
    }
}

void example_promise() {
    std::promise<int> prom;
    std::future<int> fut = prom.get_future();
    std::thread t(worker, std::ref(prom));
    t.join();
    int value = fut.get();
}
```

### 5.3 std::packaged_task

```cpp
#include <future>
#include <thread>

int task(int x) {
    return x + 1;
}

void example_packaged_task() {
    std::packaged_task<int(int)> pt(task);
    std::future<int> fut = pt.get_future();
    std::thread t(std::move(pt), 10);
    t.join();
    int result = fut.get();
}
```

### 5.4 std::shared_future 多线程共享结果

```cpp
#include <future>
#include <vector>
#include <thread>

void example_shared_future() {
    std::promise<int> prom;
    std::shared_future<int> sf = prom.get_future().share();
    
    std::vector<std::thread> threads;
    for (int i = 0; i < 4; ++i) {
        threads.emplace_back([sf] {
            int value = sf.get();
        });
    }
    prom.set_value(42);
    for (auto& t : threads) t.join();
}
```

## 6. 线程安全容器

### 6.1 线程安全队列

```cpp
#include <mutex>
#include <queue>
#include <memory>

template<typename T>
class ThreadSafeQueue {
private:
    mutable std::mutex mtx_;
    std::queue<std::shared_ptr<T>> queue_;
    
public:
    void push(T value) {
        std::lock_guard<std::mutex> lock(mtx_);
        queue_.push(std::make_shared<T>(std::move(value)));
    }
    
    std::shared_ptr<T> try_pop() {
        std::lock_guard<std::mutex> lock(mtx_);
        if (queue_.empty()) return nullptr;
        auto result = queue_.front();
        queue_.pop();
        return result;
    }
    
    bool empty() const {
        std::lock_guard<std::mutex> lock(mtx_);
        return queue_.empty();
    }
};
```

### 6.2 线程安全哈希表

```cpp
#include <shared_mutex>
#include <vector>
#include <list>
#include <algorithm>

template<typename K, typename V, size_t Buckets = 16>
class ThreadSafeHashMap {
private:
    struct Bucket {
        mutable std::shared_mutex mtx;
        std::list<std::pair<K, V>> data;
        
        V get(const K& key, const V& default_value) const {
            std::shared_lock<std::shared_mutex> lock(mtx);
            auto it = std::find_if(data.begin(), data.end(),
                [&](const auto& p) { return p.first == key; });
            return (it != data.end()) ? it->second : default_value;
        }
        
        void set(const K& key, V value) {
            std::unique_lock<std::shared_mutex> lock(mtx);
            auto it = std::find_if(data.begin(), data.end(),
                [&](const auto& p) { return p.first == key; });
            if (it != data.end()) {
                it->second = std::move(value);
            } else {
                data.emplace_back(key, std::move(value));
            }
        }
        
        void erase(const K& key) {
            std::unique_lock<std::shared_mutex> lock(mtx);
            data.remove_if([&](const auto& p) { return p.first == key; });
        }
    };
    
    std::vector<Bucket> buckets_;
    
    Bucket& get_bucket(const K& key) {
        return buckets_[std::hash<K>{}(key) % Buckets];
    }
    
public:
    ThreadSafeHashMap() : buckets_(Buckets) {}
    
    V get(const K& key, const V& default_value = V{}) const {
        return const_cast<ThreadSafeHashMap*>(this)->get_bucket(key).get(key, default_value);
    }
    
    void set(const K& key, V value) {
        get_bucket(key).set(key, std::move(value));
    }
    
    void erase(const K& key) {
        get_bucket(key).erase(key);
    }
};
```

## 7. 死锁预防

### 7.1 固定加锁顺序

```cpp
#include <mutex>

class Account {
public:
    std::mutex mtx;
    int balance;
};

void transfer_safe(Account& a, Account& b, int amount) {
    std::mutex* first = &a.mtx;
    std::mutex* second = &b.mtx;
    if (first > second) std::swap(first, second);
    
    std::lock_guard<std::mutex> lock1(*first);
    std::lock_guard<std::mutex> lock2(*second);
    
    a.balance -= amount;
    b.balance += amount;
}
```

### 7.2 使用 std::scoped_lock (C++17)

```cpp
#include <mutex>

void transfer_scoped(Account& a, Account& b, int amount) {
    std::scoped_lock lock(a.mtx, b.mtx);
    a.balance -= amount;
    b.balance += amount;
}
```

### 7.3 使用 std::lock + std::lock_guard

```cpp
#include <mutex>

void transfer_lock(Account& a, Account& b, int amount) {
    std::lock(a.mtx, b.mtx);
    std::lock_guard<std::mutex> lock1(a.mtx, std::adopt_lock);
    std::lock_guard<std::mutex> lock2(b.mtx, std::adopt_lock);
    a.balance -= amount;
    b.balance += amount;
}
```

### 7.4 使用 timed_mutex 避免无限等待

```cpp
#include <mutex>
#include <chrono>

bool try_transfer(Account& a, Account& b, int amount) {
    std::unique_lock<std::timed_mutex> lock1(a.mtx, std::defer_lock);
    std::unique_lock<std::timed_mutex> lock2(b.mtx, std::defer_lock);
    
    if (!lock1.try_lock_for(std::chrono::milliseconds(100))) {
        return false;
    }
    if (!lock2.try_lock_for(std::chrono::milliseconds(100))) {
        return false;
    }
    
    a.balance -= amount;
    b.balance += amount;
    return true;
}
```

## 8. std::latch 和 std::barrier (C++20)

### 8.1 std::latch - 一次性同步

```cpp
#include <latch>
#include <vector>
#include <thread>

void example_latch() {
    const int num_tasks = 4;
    std::latch done(num_tasks);
    
    std::vector<std::jthread> threads;
    for (int i = 0; i < num_tasks; ++i) {
        threads.emplace_back([&done, i] {
            std::cout << "Task " << i << " completed\n";
            done.count_down();
        });
    }
    
    done.wait();
    std::cout << "All tasks completed\n";
}
```

### 8.2 std::barrier - 可重用屏障

```cpp
#include <barrier>
#include <vector>
#include <thread>
#include <iostream>

void example_barrier() {
    const int num_threads = 4;
    auto on_completion = []() noexcept {
        std::cout << "--- Phase completed ---\n";
    };
    std::barrier sync(num_threads, on_completion);
    
    std::vector<std::jthread> threads;
    for (int i = 0; i < num_threads; ++i) {
        threads.emplace_back([i, &sync] {
            for (int phase = 0; phase < 3; ++phase) {
                std::cout << "Thread " << i << " phase " << phase << "\n";
                sync.arrive_and_wait();
            }
        });
    }
}
```

## 9. 协程 (C++20)

### 9.1 基本协程结构

```cpp
#include <coroutine>
#include <iostream>

struct Generator {
    struct promise_type {
        int current_value;
        
        Generator get_return_object() {
            return Generator{
                std::coroutine_handle<promise_type>::from_promise(*this)
            };
        }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() {}
        
        std::suspend_always yield_value(int value) {
            current_value = value;
            return {};
        }
    };
    
    std::coroutine_handle<promise_type> h;
    
    ~Generator() { if (h) h.destroy(); }
    
    bool next() {
        h.resume();
        return !h.done();
    }
    
    int value() {
        return h.promise().current_value;
    }
};

Generator fibonacci(int n) {
    int a = 0, b = 1;
    for (int i = 0; i < n; ++i) {
        co_yield a;
        auto tmp = a;
        a = b;
        b = tmp + b;
    }
}

void example_coroutine() {
    auto gen = fibonacci(10);
    while (gen.next()) {
        std::cout << gen.value() << " ";
    }
}
```

### 9.2 异步协程示例

```cpp
#include <coroutine>
#include <future>
#include <iostream>

struct Task {
    struct promise_type {
        std::promise<void> prom;
        
        Task get_return_object() {
            return Task{prom.get_future()};
        }
        std::suspend_never initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() { prom.set_value(); }
        void unhandled_exception() { prom.set_exception(std::current_exception()); }
    };
    
    std::future<void> fut;
};

struct Awaitable {
    std::future<int>& fut;
    
    bool await_ready() { return fut.wait_for(std::chrono::seconds(0)) == std::future_status::ready; }
    void await_suspend(std::coroutine_handle<> h) {
        std::thread([this, h]() mutable {
            fut.wait();
            h.resume();
        }).detach();
    }
    int await_resume() { return fut.get(); }
};

Task async_compute() {
    std::promise<int> prom;
    std::future<int> fut = prom.get_future();
    std::thread([&prom] {
        std::this_thread::sleep_for(std::chrono::seconds(1));
        prom.set_value(42);
    }).detach();
    
    int result = co_await Awaitable{fut};
    std::cout << "Result: " << result << "\n";
}
```

## 10. 并发编程检查清单

### 10.1 线程创建与管理

- [ ] 使用 std::jthread (C++20) 或确保 join/detach
- [ ] 传递引用参数使用 std::ref 或 std::cref
- [ ] 避免 detach，优先使用 join 或 jthread
- [ ] 检查 joinable() 后再 join()

### 10.2 互斥锁使用

- [ ] 始终使用 RAII 锁管理（lock_guard, unique_lock, scoped_lock）
- [ ] 避免手动 lock/unlock
- [ ] 多锁使用 scoped_lock 或 std::lock
- [ ] 读多写少场景使用 shared_mutex
- [ ] 不要在持有锁时调用未知代码

### 10.3 原子操作

- [ ] 简单计数器使用 atomic
- [ ] 复杂数据结构使用 mutex
- [ ] 根据需求选择合适的内存序
- [ ] 避免过度使用 seq_cst

### 10.4 死锁预防

- [ ] 固定多锁获取顺序
- [ ] 使用 scoped_lock 同时获取多锁
- [ ] 避免嵌套锁
- [ ] 考虑使用 try_lock 超时

### 10.5 条件变量

- [ ] 使用谓词避免虚假唤醒
- [ ] 在循环中 wait 或使用 wait 谓词版本
- [ ] notify 前确保状态已改变
- [ ] 使用 notify_one 还是 notify_all 需判断

### 10.6 future/promise

- [ ] 检查 future::get() 只调用一次
- [ ] 使用 shared_future 多线程共享结果
- [ ] 设置 promise 异常处理异常情况
- [ ] 选择合适的 async 启动策略

### 10.7 C++20 新特性

- [ ] 使用 std::jthread 替代 std::thread
- [ ] 使用 std::latch 进行一次性同步
- [ ] 使用 std::barrier 进行可重用屏障
- [ ] 协程需理解其生命周期和内存管理

## 11. 常见陷阱

### 11.1 竞态条件

```cpp
int counter = 0;
void unsafe_increment() {
    ++counter;
}

std::atomic<int> safe_counter{0};
void safe_increment() {
    safe_counter.fetch_add(1, std::memory_order_relaxed);
}
```

### 11.2 虚假唤醒

```cpp
std::condition_variable cv;
std::mutex mtx;
bool ready = false;

void wait_wrong() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock);
}

void wait_correct() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return ready; });
}
```

### 11.3 对象生命周期

```cpp
class BadExample {
    std::thread t_;
    void start() {
        t_ = std::thread([this] {
            this->do_work();
        });
    }
    ~BadExample() {
        if (t_.joinable()) t_.detach();
    }
};

class GoodExample {
    std::jthread t_;
    void start() {
        t_ = std::jthread([this](std::stop_token token) {
            while (!token.stop_requested()) {
                this->do_work();
            }
        });
    }
};
```
