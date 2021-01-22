## Ch 2 Managing threads

Threads are started by constructing a `std::thread` object that specifies the task to run. The task could be a function that takes no parameters, also a function object that takes additional parameters.

```C++
class background_task; // callable

// it was considered this way
// background_task noNameFunc();
std::thread my_thread(background_task());
```

declares a my_thread **function**, when background_task taks a single parameter,  it is a type of pointer-to-a-function-taking-no-parameters-and-returning-a-background_task-object.

Solution(prevent interpreation as a function declaration):
```C++
std::thread my_thread((background_task()));
std::thread my_thread{background_task()};
```

Once a thread started, you need to explicitly decide to:
1. **wait** for it to finish
2. **run on its own.**

The main program is terminated, the std::thread object is destroyed and dtor called std::terminate() to handle this scenario.

Problem: A thread may access a local variable via pointers or references it holds.

```C++
// function exits while my_thread is still running
void f() {
    //...
    my_thread.detach();
}
```

Make the thread function self-contained and copy the data into the thread rather than sharing the data.

By calling `join()` on the associated std::thread instance, we could wait for a thread to complete. Once `join()` called, the obj is no longer joinable, and `joinable()` return `false`.

It should be careful where the code calls `join()`, which will bolck.

Using RAII to deal with exception or make a function exits with two thread ends.

```C++
class thread_guard {
    // ...
    ~thread_guard() {
        if (t.joinable()) {
            t.join();
        }
    }
};

void f() {
    //.. thread t(obj);
    thread_guard g(t);
    do_something_in_current_thread();
}
```

If don't wait, t will be deal with std::terminate()

```C++
t.detach();
assert(!t.joinable());
```

detach() can be called only the joinable() returns true.

```C++
std::thread t2 = std::move(t1);
```

Ownership and move sematics.

## Ch3 Sharing data between threads

Race condition elaborations was learned from OS course. One thread is reading while another is removing, writing or inserting.

#### 3.2 Protecting shared data with mutexs

Before accessing a shared data structure, you **lock** the mutex associated with that data. When finished, you **unlock** the mutex.

```C++
std::mutext some_mutex;
```

Lock mutex with a call to the `lock()` member function, and unlock it with `unlock()`. Due to consideration of exceptions, it is not recommended practice to call the member functions directly. Instead, the `std::lock_guard<T>` implements RAII idiom for a mutext.

```C++
void f() {
    std::lock_guard<std::mutext> guard(some_mutext);
}
```

In C++ 17 with class template argument deduction feature, it can be written in this way:
```C++
std::lock_guard guard(some_mutext);
```

C++ also introduces an enhanced version of lock guard called `std::scoped_lock`, so in a C++17 environment, this may well be written as:
```C++
std::scoped_lock guard(some_mutex);
```

In designer perspective, it is common to group the mutex and the protected data together in a class. But if one of the member functions returns a ptr or ref to the protected data, you have blown a big hole in the protection. Protecting data with a mutex therefore requires careful interface design.

#### 3.2.3 Spotting race conditions: thread safe stack
Deleting a node in doubly linked list via protecting access to each node individually may not work as we want, because the race condition could still happen. The whole data structure and the whole delete operation should be considered.

Consider a stack data structure, like `std::stack`. If you change `top()`, the top element, so that it returns a copy rather than a ref and protect the internal data with a mutex, the potential race condition is similar as delete operation in doubly linked list node. One of thread is reading `empty()` or `size()`, but another thead calls `push()` or `pop()`.

If the stack is protected by a mutex internally, only one thread can be running a stack member function at any one time.

A solution settles this condition via combining the pop with top, and returns a `std::shared_ptr` of the corresponding object.

#### CppCon Back to Basics:Concurrency -Arthur O'Dwyer
If we are not going to pass the mutex arround, use `std::lock_guard<std::mutex>`. Otherwise, use the `std::unique_lock<std::mutex>`, which is similar to `std::unique_ptr`. And it is not copyable but movable.
```C++
unique_lock<mutex> f(uniqe_lock<mutex> lk) {
    // ...
    if (/* somecase */) {
        lk.unlock(); // release the lock
    }
    return lk;
}

// somewhere
unique_lock<mutex> lk(mtx);
f(std::move(lk));
```

```C++
std::once_flag once_;
auto& getConn() {
    std::call_once(once_, [] () {
        conn_ = NetworkConnection(defaultHost);
    });
    return *conn_;
}
```

`std::call_once(once_, f, args...);` executes the Callable object f exactly once, even if called concurrently, from several threads

#### 3.2.4 Deadlock

Always lock the two mutexes in the same order.

The C++ has a cure for this. The `std::lock` is a function that can lock two or more mutexes at once without risk of deadlock.

The `std::adopt_lock` parameter is supplied in addition to the mutex to indicate to the `std::lock_guard` objects that the mutexes are **already locked**, and they should adopt the ownership of the existing lock on the mutex rather than attempt to lock. [cpp concurrency p.52]

```C++
friend void swap(X& lhs, X& rhs) {
    if (&rhs == &lhs) return ;
    std::lock(lhs.m, rhs.m); // m : std::mutex
    std::lock_guard<std::mutex> lock_a(lhs.m, std::adopt_lock);
    std::lock_guard<std::mutex> lock_b(rhs.m, std::adopt_lock);
    swap(lhs.some_data, rhs.some_data);
}
```

In C++17, `std::scoped_lock` could lock more than one mutex.

```C++
friend void swap(X& lhs, X& rhs) {
    if (&rhs == &lhs) return ;
    // CTAD
    // std::scopedJ_lock<std::mutex, std::mutex>
    std::scoped_lock guard(lhs.m, rhs.m); // m : std::mutex
    swap(lhs.some_data, rhs.some_data);
}
```

#### 3.2.6 std::unique_lock

The `std::unqiue_lock` does not always own the mutex and accepts the second argument as following:
1. `std::adopt_lock` means that ctor have the lock obj manage the lock on a mutex.
2. `std::defer_lock` indicates that the mutex should remain **unlocked** on construction.

The `std::unqiue_lock<std::mutex>` can be passed to the `std::lock()` to lock the underlying locks.

Unless transferring lock ownership is neccessary, `std::scoped_lock` or `std::lock_guard` is better and convinient.

#### 3.2.7 Granularity
The ownership of a mutex can be transferred between instances by **the move semantics**, say, `std::move()`.

In general, a lock should be held for only the minimum possible time needed to perform the required operations. This also means that time-consuming operations such as  acquiring another lock or waiting for I/O to complete shouldn't be done while holding a lock.

If you do not hold the required locks for the **entire** duration of an operation, you are exposing yourself to race conditions.

#### 3.3

In some scenario, the shared data needs protection only from concurrent access while it is **being initialized**, but after that no explicit synchronization is required. That is to say, it need the `std::call_once() and std::once_flag` to guarantee.

The infamous **double-checked locking** pattern has the potential for nasty race conditions.

A scenario that accessing a local variable declared with **static** is proper to use `std::call_once(), std::once_flag`.

#### multiple reader and a single writer
The C++17 provides `std::shared_mutex` and `std::shared_timed_mutex`. `std::lock_guard<std::shared_mutex>` and `std::unique_lock<std::shared_mutex>` can be used for the locking to ensure exclusive access(the writter's side). 

The use of `std::shared_lock<std::shared_mutex>` is to obtain shared access, read-only access; multiple threads can therefor access simultaneously without problems(the reader's side).

The `std::recusive_mutex` works like `std::mutex`. How many times the recursive_mutex is locked, the corresonding number of call to `unlock` is required.

## Ch4 Synchronizing concurrent operations

condition variables, futures, latches and barriers.

#### 4.1
Use the condition variable to wait for a certain event. It involves **waiting** and **notifying**.

#### 4.1.1 waiting for a condition with condition variables
Both of `std::condition_variable` and `std::condition_variable_any` works with a mutex in order to provide appropriate synchronization. The former is preferred with consideration of performance, OS resources and additional costs.

When the resource is prepared, the `std::condition_variable` calls the `notify_one()` member function on the instance to notify the waiting thread (if there is one).

```C++
std::mutex mtx;
std::condition_variable cond;

// produce
void produce_thread() {
    {
        data d prepared;
        std::lock_guard<std::mutex> lk(mtx);
        push d into container;
    }
    cond.notify_one();
}

// consume
void consume_thread() {
    while (true) {
        std::unique_lock lk(mtx);
        cond.wait(lk, [] { return container is not empty; });
        fetch data from container;
        lk.unlock();
        process data;
        if (all data processed) break;
    }
}
```

The `wait()` takes the second parameter to check the condition being waited for. If the condition isn't statisfied, wait() unlocks the mutex and puts the thread in a **blocked** or **waiting** state. Otherwise, returns if it's satisfied.

When the condition_variable is notified by a call to `notify_one()` from other thread, the thread wakes from its slumber (unblocks it), reacquires the lock on the mutex and checks the condition again.

It is not advisable to use a function with side effects for the condition check.

`notify_all()` notify all the condition_variable that are waiting.

#### 4.2 futures
A future may have data associated with it just wait it. Once an event has happend (and the future has become **ready**), the future can't be reset.

An instance of `std::future` refers to its only one associated event, whereas multiple instances of `std::shared_future` may refer to the same event.

The future objects themselves don't provide synchronized accesses.

Use `std::async` to start an asynchronous task, which returns a `std::future` object. Get the value via calling `get()` on the future. And the thread blocks until the future is **ready**.

`std::async` could accept additional arguments. If the first argument is a ptr to a member function, the second argument provides the corresponding object instance on which applies to it (either directly, or via a pointer, or wrapped in std::ref). And the remaining arguments are passed to the member function.

Otherwise, the first argument is a callable object or function, the second and subsequent arguments are passed.

If the arguments are rvalues, both callable and subsequent arguments, the copies are created by moving the originals.

#### 4.2.2

```C++
template<typename R, class ...Args>
class packaged_task<R(Args...)>;

std::packaged_task<int(int,int)> task([](int a, int b) {
    return std::pow(a, b); 
});
std::future<int> result = task.get_future();

task(2, 9);

std::cout << "task_lambda:\t" << result.get() << '\n';
```

`std::packaged_task<>` ties a future to a function or callable object (function, lambda expression, bind expression, or another function object) so that it can be invoked asynchronously.

The `std::packaged_task` object itself is a callable object, which can be passed to a `std::thread`. When invoked, the return value is stored as the asynchronous result in the `std::future` obtained from `get_future()`.

Passing tasks between threads via `std::packaged_task` could send messages to the right thread.

```C++
std::mutex m;
// using local scope to lock the corresponding critical section
void gui_thread() {
    while (!gui_shutdown_msg_recved()) {
        std::packaged_task<void()> task;
        { // local scope with
            std::lock_guard<std::mutex> lk(m);
            retrieve_task_or_loop();
        }
        task();
    }
}

std::thread gui_bg_thread(gui_thread);

// If the thread that posted the msg to the gui thread needs to know that the task has been completed, waits for the future.
// Otherwise, simply discards.
template<typename Func>
std::future<void> post_task(Func f) {
    std::packaged_task<void()> task(f);
    std::future<void> res = task.get_future();
    std::lock_guard<std::mutex> lk(m);
    tasks.push_back(std::move(task));
    return res;
}
```

#### 4.2.3 std::promise

If the results come from more than one place, `std::packaged_task` could not deal with the situation well.

`std::promise<T>` provides a means of setting a value that can later be read through an associated `std::future<T>` object. A std::promise/std::future pair would provide a mechanism; the waiting thread could block on the future (future::get()), while the thread providing the data could use the promise to set the associated value and make the future ready.

std::future is obtained by calling the `get_future()` of the promise 

#### 4.2.4
The exception is stored in the future in place of a stored value. A call to get() rethrows that stored exception. So as to std::packaged_task.

With call for `set_exception()` member function, the `std::promise()` provides the same facility.

```C++
try {
    somePromise.set_value(calc());
} catch (...) {
    somePromise.set_exception(std::current_exception());
}
```

The destructor of std::promise or std::packaged_task will store a `std::future_error` exception with an error code of `std::future_errc::broken_promise` in the associated state if the future is not ready.

#### 4.2.5 Waiting from multiple threads.

Only the first call for `get()` works, the subsequent call would result undefined behavior.

`std::future` is only moveable, so ownership can be transferred, but only one instance refers to a particular async result at a time.

`std::shared_future` isntances are copyable, so you can have multiple objects referring to the same associated state.

`std::future::share()` creates a new std::shared_future and transfers ownership to it directly.

```C++
std::future f = std::async(someFunc);
std::shared_future sf1(f.share());
// std::shared_future sf2(std::move(f));
// std::shared_future sf3(f); implicit transfer of onwnership
```

## Ch6 Designing lock-based concurrent data structures

Multiple threads can access the data structure concurrently. No data will be lost or corrupted, all invariants will be upheld, and there will be no problematic race conditions. This data structre is said to be **thread-safe**.

The smaller the protected region, the fewer operations are serialized, and the greater the potential for concurrency.

#### Guidelines for designing DSs for concurrency

Two aspects to consider when designing data structures for concurrent access:
1. ensureing that the accesses are **safe**
2. **enabling** genuine concurrent access.

1. Avoid race condition in the interface by providing functions for complete operations rather than for operation steps.
2. Ensure the jpresence of exceptions to ensure the invariants.
3. Minimize the opportunities for dedlock, avoid nested locks where possible.

A crucial question to consider, if one thread is accessing the  data structure through a particular function, which functions are safe to call from other threads?

### 6.2 Lock-based concurrent DSs

The design of lock-based concurrent data strucutures is all about ensuring that the right mutex is locked.

Single mutex lock is hard enough to protect a data structure. But multiple mutexes locks bring the possibility of deadlock.

Using RAII, such as `std::lock_guard<>`, ensures the mutex is never left locked, and is a safe way to deal with exception. 

Because the memory might be use up, using `std::shared_ptr` to return rather than a copy to ensure exception-safe.

If a member function calls another member function, while both held lock, it turns the opportunity for deadlock.

The *serialization* of threads can potentially limit the performance. There's thread waiting for the resource without block, there's a potential performance problem. Using condition variables is a way to deal with this case.

A solution to the problem of waiting for a queue entry is to use conditional wait, rather than continuously calling member function.

If the data is held by `std::shared_ptr`, the allocation of the new instance can be done outside the lock.

```C++
void push(T val) {
    std::shared_ptr<T> data(
        std::make_shared<T>(std::move(val)));
    std::lock_guard<std::mutex> hold(mut);
    // put into the internal
    data_queue.push(data);
    data_cond.notify_one();
}
```

This fine grained locks reduces the time the mutex is held, allowing other threads to perform. By taking control of the detailed implementation of the data structure, you can provide more fine-grained locking and allow a higher level of concurrency.

#### 6.2.3 

In concurrency, any access or modification in shared variable must be considered to lock. If is clearer to wrap a complex part in function.

Hold the locks for the shortest possible length of time, lock the concurrency read and modification part.

Exceptions is considerable when returning from a function with no memory for constructing a new object.

A lock should ensure the safety of a shared variable during the whole time it operates (reads and writes).

With the condition_variable, it is considerable to care the time where to wait and notify. The predicates is important.

### 6.3 Designing

The data strucuture should not return a reference, pointer or iterator to user directly under concurrently accessing, for those case would broken some invariants and cause undefined behaviours.

## <<Concurrency with Modern C++>>

### Basics and The Contract

The memory-ordering defines the details which causes the happens-before relations and are, therefore, key fundamental part of the memory model. The C++ memory model defines a contract.

Each participants (the system, the processor, the compiler, the cache, etc.) of the code involves the contract, each of them wants to optimise its part. The weaker the rules are that the programmer has to follow, the more potential there is for the system to generate a highly optimised executable.

There are contract levels in C++11:
1. Single threading (One control flow)
2. Multithreading (Tasks, threads, condition_variables)
3. Atomic (Sequential consistency, acquire-release semantic, relaxed semantic)

Strong -> weak: 1 > 2 > 3

Before C++11, there was only one contract, neither multithreading nor atomics. The observed behaviour of the program **corresponded** to the sequence of the instructions in the source code. There was no memory model but **sequence point**.

The **sequential consistency** is called the **strong** memory model, and the **relaxed semantic** being called the **weak** memory model.

The more we weaken the memory model, the more we change our focus towards other things, such as:
1. The program has more optimisation potential.
2. The possible number of control flows of the program increases exponentially.
3. We are in the domain for the expoerts.
4. The program breaks our intuition.
5. We apply micro-optimisation.

### Atomic

Atomics are the base of the C++ memory model. By default, the strong version of the memory model is applied to the atomics.

#### Strong Memory Model

Sequential consistency provides two guarantees:
1. The instructions of a program are executed in source code **order**.
2. There is a global order of all operations on all threads.

Suppose there are two threads:
```C++
// Both x and y are atomic type.
// Thread 1
x.store(1);;
res1 = y.load();

// Thread 2
y.store(1);
res2 = x.load();
```

By default, sequential consistency applies (Both api have the default argument `std::memory_order order = std::memory_order_seq_cst`). Which order can the statements be executed?

The sequential consistency guarantees that
1. the instructions are executed in the order defined in the source code.
2. all instructions of all threads have to follow a global order. It means that another thread sees the operations of the thread in the same executed order (the order as the source code defines). That is to say, instructons of each thread would not be reordered.

#### Weak Memory Model

The system guarantees *well-defined* program behaviour without data races (for the atomic operation goes). If the relaxed semantics put into practice, the pillars of the contract dramatically change. On the one hand, it is **difficult to understand possible interleavins** of the two threads. On the other hand, the system has a lot more **optimisation possibilities**.

With the relaxed semantic, the *counter-intuitive* behaviour is that the thread can see the operations of the another in a different order, which executes without the order the code lists, so there is no view of a global clock.

Between the sequential consistency and the relaxed-semantic are a few more models. With the **acquire-release semantic*, the programmer has to obey weaker rules than with sequential consistency.

#### The Atmomic Flag

`std::atomic_flag` is a atomic boolean. Unlike all specializations of `std::atomic`, it is guaranteed to be lock-free. Unlike `std::atomic<bool>`, `std::atomic_flag` does not provide load or store operations.

Member function: `clear()` sets its value to `false`, while `test_and_set` sets the value back to `true` and return the previous value. There is no method to ask for the current value, but in C++20, `test()` supplied.

To use `std::atomic_flag`, it must be initialised to `false` with the constant `ATOMIC_FLAG_INIT`.

```C++
std::atomic_flag lock = ATOMIC_FLAG_INIT;
// std::atomic flag(ATOMIC_FLAG_INIT); unspecified
```

`std::atomic_flag` has two outstanding properties, it is:
1. the only lock-free atomic, and it is non-blocking.
2. the building block for higher level thread abstractions.

It is powerful enough to build a spinlock. Because of the expensive context switch in the wait state from user space to the kernel space, sometimes it is better to use spinlock in the case that the time for block is short.
```C++
std::atomic_flag lock = ATOMIC_FLAG_INIT;

void lock() {
    while ( lock.test_and_set() );
}

void unlock() {
    lock.clear();
}
```

#### std::atomic

The keyword `volatile` in Java and C# has the meaning of `std::atomic` in C++, is for special objects, on which optimised read or write operations are not allowed.

Memory order may cause the condition variables in two way:
1. spurious wakeup while no notification happened.
2. lost wakeup: the sender sends its notification before the receiver gets to a wait state.

`compare_exchange_strong`, this operation often called compare and swap (CAS), it is the foundation of non-blocking algorithms.

### The Synchronisation and Ordering Constraints

To classify these six memory ordering, it helps to answer two questions:
1. Which kind of atomic operations should use which memory model?
2. Which synchronisation and ordering constraints are defined by the six variants?

#### Kind of Atomic Operation

1. Read operation: `memory_order_acquire` and `memory_order_consume`
2. Write operation:`memory_order_release`
3. Read-modify-write operation:`memory_order_acq_rel` and `memory_order_seq_cst`

`memory_order_relaxed` defines no synchronisation and ordering constraints.

1. Sequential consistencay:`memory_order_seq_cst`
2. Acquire-release:`memory_order_consume, memory_order_acquire, memory_order_release, memory_order_acq_rel`
3. Relaxed:`memory_order_relaxed`

The sequential consistency establishes a global order between threads.

The acquire-release semantic establishes an ordering between **reading and writing operations** on the **same** atomic variable with different threads.

The relaxed semantic only guarantees the modification order of some atomic m. Modification order means that all modifications on a particular atomic `m` occur in some particular total order.

#### Sequential Consistency

All operations on all threads obey a universal clock. The program execution is deterministic.

*sequenced-before* => *happens-before*.

#### Acquire-Release Semantic

There is no global synchronisation between threads in the acquire-release semantic; there is only synchronisation between atomic operations **on the same atomic variable**.

All read and write operations cannot be moved after a release operation, and all read and write operation cannot be moved before an acquire operation.

The reading of an atomic var with `load` or `test_and_set` is an acquire operation.

The releasing of a lock or mutex *synchronizes-with* the acquiring of a lock or a mutex.

Acquire and release operations come in pairs.

The acquire-release semantic is transitive. That means if it exists between two threads (a, b) and between (b, c), you get an acquire-release semantic between (a, c).

The sequential consisitency is too expensive, we want the light-weight acquire-release semantic.

The relation *sequenced-before* and *synchronizes-with* should be noticed.

If `dataProduced.store(true, std::memory_order_release)` *happens-before* `dataProduceed.load(std::memory_order_acquire)`, all operaetions before `dataProduced.store(true, std::memory_oreder_release)` *happens-before* all operations after `dataProduced.load(std::memory_order_acquire)`.
