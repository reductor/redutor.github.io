---
layout: post
title:  "The downsides of C++ Coroutines"
date:   2023-08-10 19:00:00 +1000
categories: cpp
---

This blog post aims to shed light on the potential risks associated with transitioning a codebase to incorporate coroutines. **Continual misuse of coroutines may result in software vulnerabilities and performance degradation.**

Coroutines, even in the absence of multi-threading, demand a level of caution comparable to writing multi-threaded code due to their asynchronous nature. Understanding and properly managing coroutines is vital as they introduce complexities that, if mishandled, can lead to security vulnerabilities and suboptimal performance in software systems.

## Contents

* [How normal functions and the stack work](#how-normal-functions-and-the-stack-work)
* [How c++ coroutine functions work](#how-c-coroutine-functions-work)
* [Overlooking the Complexity of Asynchronous Code](#overlooking-the-complexity-of-asynchronous-code)
* [Safeguarding Argument Lifetime in Coroutines](#safeguarding-argument-lifetime-in-coroutines)
* [Managing Iterator and Pointer Invalidation in Coroutines](#managing-iterator-and-pointer-invalidation-in-coroutines)
* [Understanding Eager vs. Lazy Starting Coroutines for Improved Coroutine Integrity](#understanding-eager-vs-lazy-starting-coroutines-for-improved-coroutine-integrity)
* [Ensuring Robustness: Testing Coroutine Suspend Points](#ensuring-robustness-testing-coroutine-suspend-points)
* [Memory Allocation Impact on Coroutine Performance](#memory-allocation-impact-on-coroutine-performance)
* [Managing Stack Bloat with HALO in Coroutines](#managing-stack-bloat-with-halo-in-coroutines)
* [Impact of Debug Builds on Coroutine Performance](#impact-of-debug-builds-on-coroutine-performance)
* [Managing the Cascading Effect of Stackless Coroutines](#managing-the-cascading-effect-of-stackless-coroutines)
* [Comparing Stackful Coroutines (Fibers) to Stackless Coroutines](#comparing-stackful-coroutines-fibers-to-stackless-coroutines)


## How normal functions and the stack work 

When a regular function is invoked, its arguments are typically passed using registers. If there's an excess of arguments or if specified by the Application Binary Interface (ABI), they may be passed using the stack. The exact allocation of registers and stack usage is governed by the ABI specifications.

Internally, upon invocation, the function allocates sufficient stack space to accommodate local variables and temporary values. It's important to note that not all of these variables end up on the stack; some might reside in registers or could even be optimized out entirely by the compiler.

Upon completion of the function's execution, the initially reserved stack space is reset to its original state when the function was called, and control is returned to the caller.

ABIs can vary significantly across platforms. Some ABIs may define both volatile and non-volatile registers, necessitating saving and restoration processes as required during function execution.

The stack space responsible for holding arguments, local variables, and temporary values is essentially a standard memory block, often allocated at the thread's initiation, with each thread possessing its dedicated stack.

This stack memory sees extensive use with every function call within a thread, resulting in frequent access. Consequently, it tends to reside in the L1/L2 cache, making it notably faster to access.

## How c++ coroutine functions work

Much like conventional functions where arguments are passed through registers or the stack, coroutines adhere to the same Application Binary Interface (ABI) specifications. However, the behavior and internal mechanisms differ significantly.

A key distinction is that a coroutine function can pause its execution (suspend) while allowing the calling function to continue and invoke other functions. Later, it can resume the paused coroutine, either within the same context or even on a different thread.

Due to this ability to pause and resume, storing local variables, temporaries, and arguments on the stack, as with regular functions, is unfeasible. Instead, these elements need to be stored elsewhere, typically in the heap.

An illustrative example of how a compiler might transform coroutine code can be observed:

Consider the original coroutine code:
```cpp
task<void> func()
{
  int mylocal = 10;
  co_return;
}
```

The compiler-generated code might appear as follows:
```cpp
// Struct representing the coroutine state
struct func_frame
{
  task<void>::promise_type __promise;
  int __step = 0;

  decltype(__promise.initial_suspend()) __initial_suspend_awaiter;
  decltype(__promise.final_suspend()) __final_suspend_awaiter;

  // Structure to hold local and temporary variables
  struct 
  {
    // Local and temporary variables reside here
    int mylocal;
  } local_and_temps;

  // Function to resume coroutine execution
  void resume()
  {
    switch(__step)
    {
      case 0:
        // co_await __promise.initial_suspend();
        __initial_suspend_awaiter = __promise.initial_suspend();
        if (!__initial_suspend_awaiter.await_ready())
        {
          __step = 1;
          __initial_suspend_awaiter.await_suspend();
          return;
        }
      case 1:
        __initial_suspend_awaiter.await_resume();
        // .. func body
        mylocal = 10;
        // co_return
        __promise.return_void();
        // co_await __promise.final_suspend();
        __final_suspend_awaiter = __promise.final_suspend();
        if (!__final_suspend_awaiter.await_ready())
        {
          __final_suspend_awaiter.await_suspend();
          return;
        }
        delete this;
    }
  }
};

// Coroutine function transformed into a coroutine frame
task<void> func()
{
  func_frame * frame = task<void>::promise_type::operator new(func_frame);
  task<void> ret = frame->__promise.get_return_object()
  frame->resume();
  return ret;
}
```

This representation is a conceptual demonstration of the underlying mechanism, showcasing a hidden structure that holds the coroutine state. In practice, certain optimizations might occur; for instance, simple local variables like mylocal could be optimized out. However, in scenarios where variables must persist across suspension points, they may be stored in `local_and_temps`. Note that in real scenarios, these variables would be initialized in place, but this is simplified example code.

Coroutines optimized using the HALO (coroutine Heap Allocation eLision Optimization) technique can, if the compiler accurately determines the coroutine's lifespan and frame layout, embed a copy of `func_frame` within the caller's local variables, which might be either a normal function or another coroutine.

A misconception arises where some believe allocation could happen on-demand at the first suspension point. However, this isn't feasible as it would require relocating all local variables and temporaries along with their references, which the compiler cannot trace across ABI boundaries.

While there exist more detailed introductions to coroutines, a simplified perspective views them as transforming a function into a structure containing locals and temporaries, coupled with a resume function that executes the function incrementally.


## Overlooking the Complexity of Asynchronous Code

The widely propagated phrase "Write async code like sync code" carries an inherent danger as it oversimplifies the intricate nature of asynchronous programming. This oversimplification can mislead developers into believing that writing asynchronous code is as straightforward as writing synchronous code. In reality, handling asynchronous operations introduces an entirely new spectrum of potential bugs and complexities that are often overlooked.

When transitioning to asynchronous programming, developers might encounter a false sense of security, assuming that the inclusion of `co_await` alone is sufficient to manage asynchronous behavior. However, this belief disregards the subtleties and intricacies of dealing with suspension points. These suspension points, facilitated by `co_await`, are deceptively easy to incorporate, leading developers to overlook potential pitfalls and challenges inherent in asynchronous code.

Experienced developers well-versed in asynchronous programming understand that it's far from straightforward. It demands a deeper understanding of handling concurrency, state management, and error handling, among other considerations. By emphasizing a simplistic "Write async code like sync code" mantra, the broader complexities and potential issues are often disregarded, leaving developers ill-prepared to handle the nuanced challenges posed by asynchronous operations.

Therefore, it's crucial to approach asynchronous programming with a comprehensive understanding of its intricacies and the potential pitfalls associated with suspension points, rather than relying solely on superficial catchphrases that oversimplify the process. Understanding these complexities is fundamental to writing robust, reliable, and bug-free asynchronous code.

## Safeguarding Argument Lifetime in Coroutines

In the realm of traditional functions, developers often abide by well-established guidelines: if an argument is inexpensive to pass, use pass-by-value; if it's costly, utilize a const reference. When ownership is transferred, pass by value or rvalue reference is employed.

These ingrained rules are practically second nature to many developers, often applied without conscious thought. However, when these principles are carried over to coroutine functions, unforeseen complications arise.

Consider the following scenarios:

```cpp
task<void> async_insert(T && val);
```

```cpp
task<void> async_find(const T &);
```

```cpp
task<void> async_write(span<byte>);
```

Spotting these issues during code reviews might be challenging. There's a chance that even a vigilant reviewer could overlook these nuances.

Thankfully, with the promise type in coroutines, there's a mechanism to capture argument types, enabling the creation of static asserts to detect such cases. This approach empowers developers to implement a safeguard—assertions that statically check the argument types—providing a safety net against potential argument lifetime issues.

By leveraging these mechanisms, developers can proactively identify and rectify problematic argument lifetimes, bolstering the robustness and reliability of coroutine-based code.

## Managing Iterator and Pointer Invalidation in Coroutines

Consider a scenario where a function employing coroutines suspends during execution, leaving open the possibility of altered global state due to concurrent operations. Let's delve into an example:

```cpp
task<void> send_all(string s)
{
  for (auto & source : m_sources)
  {
    co_await source.send(s);
  }
}
```

At first glance, most developers focus on the potential invalidation of `m_sources` by the `send` operation. However, due to coroutine suspension within the loop, other concurrent activities might disrupt the data validity mid-operation.

One way to mitigate this is by queuing all sends and awaiting them collectively:

```cpp
task<void> send_all(string s)
{
  std::vector<task<void>> sends;
  sends.reserve(m_sources.size());
  for (auto & source : m_sources)
  {
    sends.push_back(source->send(s));
  }

  co_await wait_all( sends.begin(), sends.end() );
}
```

Yet, a crucial concern arises—ensuring the referenced sources remain valid throughout `send_all` execution, especially if any sources are removed concurrently.

Possible solutions to tackle this issue include:

* Enabling sources to keep themselves alive:

```cpp
struct Source : std::enable_shared_from_this<Source>
{};

task<void> Source::send(string s)
{
  auto pSelf = this->shared_from_this();
  // ...
}
```

* Employing a cloning approach within the caller to retain source validity:

```cpp
task<void> send_all(string s)
{
  std::vector<task<void>> sends;
  std::vector<shared_ptr<Source>> sources = m_sources;
  sends.reserve(sources.size());
  for (auto & source : sources)
  {
    sends.push_back(source->send(s));
  }

  co_await wait_all( sends.begin(), sends.end() );
}
```

* Tracking outstanding promises to ensure source integrity:

```cpp
struct Source
{
  std::vector<std::coroutine_handle> dependent_coroutines;
  ~Source()
  {
    for (auto & coroutine : dependent_coroutines)
      coro.destroy();
  }

  task<void> send(string s)
  {
    auto corohandle = co_await get_current_coroutine{};
    dependent_coroutines.push_back( corohandle );
    // ...
    dependent_coroutines.erase( dependent_coroutines.find( corohandle ) );
  }
}
```

* Implementing a locking mechanism to manage data access:

```cpp
task<void> send_all(string s)
{
  co_await m_sourcesLock.read_lock();
  std::vector<task<void>> sends;
  sends.reserve(m_sources.size());
  for (auto & source : m_sources)
  {
    sends.push_back(source->send(s));
  }

  co_await wait_all( sends.begin(), sends.end() );
  co_await m_sourcesLock.read_unlock();
}
```

Each approach comes with distinct trade-offs in terms of efficiency, complexity, and the potential for handling data integrity. By choosing the right strategy, developers can safeguard against iterator and pointer invalidation issues within coroutine-based systems.

## Understanding Eager vs. Lazy Starting Coroutines for Improved Coroutine Integrity

In the realm of stackless coroutines, two prevalent approaches govern the initiation of coroutine execution: "eager" and "lazy" starting coroutines. Eager coroutines commence execution immediately and persist until the first suspension point, while lazy coroutines start in a suspended state, initiating execution only when specifically resumed.

The choice between these two methods often hinges on scheduling decisions. Lazy coroutines streamline the process of scheduling the entire coroutine from inception to completion on a distinct thread.

While most major coroutine libraries adopt lazy starting coroutines, a proposed concept like Boost.Async champions eager starting coroutines.

Consider a scenario where a function consistently adheres to the pattern:

```cpp
co_await fetch_data("key");
```

In such cases, the immediate `co_await` suggests that the referenced `key` is maintained by the caller throughout its use. This insight allows for a specific implementation like this:

```cpp
eager_task<string> fetch_data(string_view key)
{
  auto it = cache.find(key);
  if (it != cache.end())
  {
    return it->second;
  }

  auto data = co_await fetch_remote(key);
  cache.emplace(key, data);
}
```

However, a potential pitfall surfaces when a new function is introduced later:

```cpp
eager_task<void> fetch_mydata(string_view key)
{
  return fetch_data(std::format("mysystem/{}", key));
}
```

Initial testing might validate its functionality, but with widespread adoption, unexpected crashes could surface in `fetch_data` due to missing `key` data.

Spotting such issues in a vast codebase becomes challenging. Detecting scenarios where `co_return co_await` is not performed, leading to data loss, requires meticulous code inspection.

Had the coroutines been lazy, such issues could have manifested more consistently. Tools like address sanitizers or other debugging utilities could potentially detect use-after-free scenarios, offering clearer indications of issues.

## Ensuring Robustness: Testing Coroutine Suspend Points

As highlighted in the earlier security-related instances, the act of functions suspending and resuming within coroutines can potentially introduce numerous unforeseen bugs.

To mitigate these risks, it becomes crucial to regularly test the suspend points within your coroutines. However, achieving this task poses a challenge as the decision to suspend or resume is not under the control of the caller of an awaitable.

Consequently, incorporating support for testing suspend points necessitates integrating it into every awaitable or adopting techniques like `await_transform`. Both approaches, especially when considering potential incorporation through dependency injection, present intricate challenges.

Building support for this testing paradigm into each awaitable involves significant effort and meticulous implementation to ensure comprehensive coverage. Alternatively, leveraging `await_transform` introduces complexities, as its integration across a codebase demands careful consideration of potential implications and necessitates a systematic approach to maintain code readability and modularity.

The intricacies involved in establishing a robust testing mechanism for suspend points within coroutines underscore the importance of strategic planning and meticulous execution. Adoption of such practices aligns with the quest for code resilience and reliability, albeit at the cost of navigating complexities inherent in such implementations.

## Memory Allocation Impact on Coroutine Performance

Upon invocation of a coroutine function that hasn't undergone allocation elision, memory allocation occurs for its coroutine frame/state. Furthermore, in attempts to mitigate challenges associated with argument lifetime issues, iterator/pointer invalidation, additional memory allocations might arise.

Hopes for the optimization potential of components like the thread-local tcache in glibc, which stores 7 equally sized allocations across 64 bins ranging from 24-1024 bytes, might arise, anticipating faster memory management.

However, consider the scenario outlined in the `send_all` function using `string`. For each invocation of `send` on every source within the loop, there arise two additional allocations: one for the coroutine frame and another for the string argument. If these allocations are of similar size, having 7 sources could potentially deplete the tcache. When the size of the string and the coroutine frame falls within the same tcache bucket, merely 4 such allocations could exhaust a tcache bucket.

This scenario highlights the potential strain on memory resources due to coroutine-related allocations, emphasizing the impact on memory management mechanisms like the tcache. Such insights are crucial for optimizing memory usage in scenarios where coroutine invocations and associated memory allocations are frequent and can influence the overall performance of the system.

## Managing Stack Bloat with HALO in Coroutines

Heap Allocation eLision Optimization (HALO) in coroutines presents an appealing approach, enabling coroutine frames to remain local without necessitating placement on the heap. However, while this optimization might seem straightforward initially, it can inadvertently lead to potentially large stack frames under specific scenarios.

Consider the following example:

```cpp
task<void> post_metric(string name, int value)
{
  char sendbuffer[16 * 1024];
  // ...
}

task<void> post_metrics()
{
  std::array metrics { 
    post_metric("a", ...);
    post_metric("b", ...);
    post_metric("c", ...);
    // ...
  };
  co_await when_all(begin(metrics), end(metrics));
}
```

Each invocation of `post_metric`` within `post_metrics`` necessitates a stack frame allocation of at least 16KB. As multiple parallel calls stemming from `post_metrics` may be simultaneously active, the cumulative stack size could grow considerably larger.

Furthermore, consider a conditional scenario in another coroutine:

```cpp
task<void> func()
{
  if (cond)
  {
    co_await func1();
  }
  else
  {
    co_await func2();
    co_await post_metrics();
  }
}
```

In this instance, the stack size of `func` potentially includes the size of `post_metrics`. Even if the condition (`cond`) evaluates to `true`, recursion within `func1` and `func2` might further contribute to the stack size of `func`.

Moreover, compiler optimizations could determine HALO decisions based on basic block probability. This means different compilers might employ HALO on diverging branches, leading to inconsistent performance and varying stack sizes based on the compiler's optimization strategy.

Managing stack bloat resulting from HALO optimization involves careful consideration of potential stack frame sizes and recursive coroutine calls to ensure optimal memory utilization and consistent performance across different execution paths.

## Impact of Debug Builds on Coroutine Performance

Even the simplest coroutine, such as the following:

```cpp
task<void> func()
{
  co_return;
}
```
internally involves multiple function calls:

```
func()
promise_type::operator new()
promise_type::promise_type()
promise_type::get_return_value()
promise_type::initial_suspend()
initial_suspend::initial_suspend()
initial_suspend::await_ready()
initial_suspend::await_suspend() [opt]
initial_suspend::await_resume()
promise_type::return_void()
final_suspend::final_suspend()
final_suspend::await_ready()
final_suspend::await_suspend() [opt]
promise_type::operator delete()
```
Under optimized conditions, many of these functions are often inlined, eliminating the associated function call overheads.

However, when executing in a debug build, these functions might exist in an unoptimized state. This lack of optimization could lead to increased overhead due to multiple function calls, impacting the performance of coroutine-based code significantly.

Developers need to be cautious when evaluating coroutine performance in debug builds, as the absence of optimizations may introduce additional overhead from function calls that would otherwise be streamlined in release builds. Profiling and performance analysis tools become invaluable in assessing and addressing these performance disparities between debug and release configurations.

## Managing the Cascading Effect of Stackless Coroutines

In C++, stackless coroutines operate by enabling suspension solely within individual functions rather than affecting the entire call stack.

The nature of these coroutines necessitates that suspension propagates up the call stack, enabling concurrent execution across multiple functions or expecting these functions to be passed into a dispatcher. Consequently, any callers up the call stack must also adopt coroutine behavior.

This propagation has a potentially expansive impact, quickly encompassing a broader scope within the codebase. A few deeply nested function calls that require coroutine behavior can prompt the widespread adoption of coroutine functionality throughout the majority of the code.

To counteract this viral nature, one strategy is to prioritize waiting over suspending further up the call tree. However, this approach might inadvertently limit the potential for parallelism.

Managing the spread of stackless coroutines necessitates a careful balance between leveraging their concurrent capabilities and judiciously containing their propagation throughout the codebase.

## Comparing Stackful Coroutines (Fibers) to Stackless Coroutines

Stackful coroutines, often referred to as fibers, provide a user-mode threading mechanism where a new thread is spawned upfront, allowing normal function calls and switching between fibers.

While stackful coroutines share some security concerns with stackless coroutines, their distinct advantage lies in having an actual stack, narrowing the range of lifetime issues that could occur.

In terms of performance, stackful coroutines differ significantly. They lack the overhead of potentially additional memory allocations or a series of extra functions per call, unlike stackless coroutines. However, they also miss out on the optimizer's ability to fully inline functions, limiting the optimization possibilities compared to stackless counterparts.

A notable distinction is that stackful coroutines do not exhibit a viral nature. Unlike stackless coroutines, where the entire call stack must adopt coroutine behavior for suspension and switching, stackful coroutines enable switching between coroutines without requiring the entire call stack to accept such changes. However, this flexibility also means that a function might be suspended unexpectedly without explicit acknowledgment.

Understanding these differences between stackful and stackless coroutines is essential for making informed decisions regarding concurrency models and their suitability for specific use cases.
