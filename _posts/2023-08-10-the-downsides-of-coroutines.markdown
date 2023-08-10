---
layout: post
title:  "The downsides of C++ Coroutines"
date:   2023-08-10 19:00:00 +1000
categories: cpp
---

This blog post is designed to highlight some of the risks which are posed with moving a code base towards coroutines, I believe that **ongoing poor coroutine usage could lead to more insecure and slower programs**.

Coroutines even without multi-threading should be treated with the same suspicion as writing multi-threaded code, its still asynchronous.

## Contents

* [How normal functions and the stack work](#how-normal-functions-and-the-stack-work)
* [How c++ coroutine functions work](#how-c-coroutine-functions-work)
* [You don't "Write async code like sync code"](#you-dont-write-async-code-like-sync-code)
* [Security: Argument lifetime](#security-argument-lifetime)
* [Security: Iterator and pointer invalidation](#security-iterator-and-pointer-invalidation)
* [Security: Eager vs lazy starting coroutines](#security-eager-vs-lazy-starting-coroutines)
* [Security: Testing suspend points](#security-testing-suspend-points)
* [Performance: Memory allocations](#performance-memory-allocations)
* [Performance: HALO stack bloat](#performance-halo-stack-bloat)
* [Performance: Debug builds](#performance-debug-builds)
* [Design: The viral nature of stackless coroutines](#design-the-viral-nature-of-stackless-coroutines)
* [Alternatives: How do stackful coroutines stack up](#alternatives-how-do-stackful-coroutines-stack-up)

## How normal functions and the stack work 

When a normal function is called arguments normally get passed into a function using registers and if there is enough arguments it might pass arguments using the stack the exact registers and stack usage is defined using in the Application Binary Interface (ABI).

Inside the function the first thing that happens is the function reserves enough stack space to store local variables and temporaries, not all of these will be placed on the stack some might live in registers or be optimized out completely.

Finally at the end of the function the stack space initially reserved get's reset to where it was initially when the function first call happens then returns to the caller.

Many details vary based on different ABIs, for example some might have volatile and non-volatile registers which might need to be saved and restored, if needed.

The stack space which stores these arguments, local variables and temporaries is just a normal block of memory which is often reserved at the start of the thread, with each thread having it's own unique stack.

Due to it's heavy usage on every function call within a thread it's often fairly hot within the L1/L2 cache making it very fast to use.

## How c++ coroutine functions work

Just like a normal function arguments are passed using registers and the stack, coroutines are using the same ABI as previously specified, however the code different is vastly different.

Unlike a normal function a coroutine function can suspend and the calling function can continue and call other functions resuming the coroutine later on, or even resume it on a different thread.

Because of this local variables, temporaries and arguments cannot be stored on the stack and must tbe stored else where, this is done by placing them on the heap.

An example of how the compiler might translate code

Original coroutine code:
```cpp
task<void> func()
{
  int mylocal = 10;
  co_return;
}
```

Compiler generated code:
```cpp
struct func_frame
{
  task<void>::promise_type __promise;
  int __step = 0;

  decltype(__promise.initial_suspend()) __initial_suspend_awaiter;
  decltype(__promise.final_suspend()) __final_suspend_awaiter;

  struct 
  {
    // local and temp variables here
    int mylocal;
  } local_and_temps;

  void resume()
  {
    switch(__step)
    {
      case 0:
        // co_await __promise.initial_suspend();
        __initial_suspend_awaiter = _promise.initial_suspend();
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
        __final_suspend_awaiter = _promise.final_suspend();
        if (!__final_suspend_awaiter.await_ready())
        {
          __final_suspend_awaiter.await_suspend();
          return;
        }
        delete this;
    }
  }
};

task<void> func()
{
  func_frame * frame = task<void>::promise_type::operator new(func_frame);
  task<void> ret = frame->__promise.get_return_object()
  frame->resume();
  return ret;
}
```

While this is not exactly how it works, it hopefully helps you imagine what goes on behind the scenes with regards to there being a hidden struct/block of memory which stores the coroutine state.

Inside that coroutine state local variables and temporaries are stored, in this small example `mylocal` probably would be optimized out but in a real coroutine where the local or temporary needs to cross a resume boundary then it would need to be stored in `local_and_temps`, also all of these would be in-place initialized but this is just example code.

With HALO (coroutine Heap Allocation eLision Optimization) when the compiler can determine the exact lifetime of the coroutine and knows its frame layout it might substitute the body of `func` with a copy of `func_frame` as a local variable of the caller (which itself might be a normal function or coroutine).

Some people have incorrectly believed that the allocation can be done on-demand at the first suspension point, this is not possible as it would require relocating all local and temporaries along with references to them which the compiler cannot track across ABI boundaries.

There are many other better introductions to coroutines, the way I like to view them is they are simply turning your function into a struct which contain the locals and temporaries then a resume function that will execute the function in steps.

## You don't "Write async code like sync code"

The phrase "Write async code like sync code" is absolutely terrible, it gives people a false sense of safety, anyone that has written a decent amount of asynchronous code knows that it's not that simple.

You have an entirely new set of possible bugs which you can run into, that you don't tend to think about because you overlook suspension points as they are easy to write you just put `co_await` and it's handled.

## Security: Argument lifetime

When it comes to normal functions most people are used to the rules of if it's cheap to pass then pass by value, if it's expensive to pass then use a const reference, and if passes ownership pass by value or rvalue reference.

These rules are built into many developers heads, you probably do it without even thinking about it, but if you follow those same rules when it comes to coroutine functions things start to go wrong.

Let's consider the following:

```cpp
task<void> async_insert(T && val);
```

```cpp
task<void> async_find(const T &);
```

```cpp
task<void> async_write(span<byte>);
```

Would you discover these in code reviews? I know there is a chance I would miss them.

Thankfully with the promise type you can capture the argument types which would allow you to create a list of static asserts and detect some of these cases.

## Security: Iterator and pointer invalidation

Because a function can suspend any global state could have changed since, imagine the following

```cpp
task<void> send_all(string s)
{
  for (auto & source : m_sources)
  {
    co_await source.send(s);
  }
}
```

Most people probably write this code only thinking about `send` invalidating `m_sources` and unless you have some bad coupling that probably won't happen, however because coroutine suspension can happen in the middle of the loop something else can run and invalidate the data.

Now try to think about how you might solve this, would you queue all the sends then wait after?

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

Seems good at first right? Well who is keeping all the sources alive if they get removed while `send_all` is in progress.

To do this you could source's send to keep themselves alive

```cpp
struct Source : std::enable_shared_from_this<Source>
{};

task<void> Source::send(string s)
{
  auto pSelf = this->shared_from_this();
  ...
}
```

or do it in the caller

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

The other approach is to have outstanding promises tracked.

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
    ...
    dependent_coroutines.erase( dependent_coroutines.find( corohandle ) );
  }
}
```

Another approach is to instead lock and unlock the data.

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

There are many ways you could approach this problem, each having their own strengths and weaknesses.

## Security: Eager vs lazy starting coroutines

With stackless coroutines there are typically two approaches taken for when the body first starts executing eager which starts executing the function immediately and doesn't stop until the first suspend point and lazy which start suspended and won't start until they are first resumed.

These two approaches normally get determined based on scheduling decisions where a lazy coroutine makes it easier to schedule the entire coroutine from start to finish on a different thread.

Most of the major coroutine libraries use lazy starting coroutines, there is a proposed Boost.Async which has eager starting coroutines.

If you you know that a function is always going to only be used with the pattern
```cpp
co_await fetch_data("key");
```

You might consider that `key` is kept alive by the caller because it immediately `co_await`s you could get away with this implementation.

```cpp
eager_task<string> fetch_data(string_view key)
{
  auto it = cache.find(key)
  if (it != cache.end())
  {
    return it->second;
  }

  auto data = co_await fetch_remote(key);
  cache.emplace(key, data);
}
```

Sometime a year or two later someone adds
```cpp
eager_task<void> fetch_mydata(string_view key)
{
  return fetch_data(std::format("mysystem/{}", key));
}
```

You run your tests it works fine, it starts getting wide spread usage suddenly you start seeing random crashes in `fetch_data` (There is no `key` because it's data is gone)

If your lucky you manage to spot this `fetch_mydata` but it could be somewhere completely independent in a different system, then you need to spot that it's not doing `co_return co_await` which is something again most people don't think of.

If you had a lazy coroutine then this would fail regularly and hopefully show up with tools like address sanitizer or anything else which might detect use-after-free.

## Security: Testing suspend points

As you've already seen with all of the security related examples that I have mentioned that functions suspending and resuming can introduce lots of random bugs.

This is why it becomes important to be regularly testing suspend points in your coroutines, which isn't an easy thing to do because the caller of an awaitable doesn't decide if it suspends or not.

Which means you need to build support into every awaitable or start making use of `await_transform`, neither of which are easy especially when it comes to wanting to add this behavior potentially via dependency injection.

## Performance: Memory allocations

When ever a coroutine function is called which hasn't had it's allocation elided away will allocate memory for it's coroutine frame/state.

Additionally due to attempting to overcome the issues with argument lifetime issues and iterator/pointer invalidation you can get even more memory allocations.

You might hope that things like the thread local tcache in glibc which has 7 equalsized allocations stored over 64 bins ranging from 24-1024 bytes, would be making these fast, .

However if you look back to the `send_all` function which took `string` then for every source that it called `send` on there was another 2 allocations (the coroutine frame and the string argument), each of those would be the same size so if you have 7 sources then your guaranteed to exhaust the tcache, if the string size coroutine frame both fit in the same bucket then you would just need 4 to exhaust a tcache bucket.

## Performance: HALO stack bloat

With HALO (coroutine Heap Allocation eLision Optimization) coroutine frames do not need to be placed on the heap and can be local (on the stack or part of the calling coroutine)

This might seem easy enough you don't have too many giant stack frames, however if you manage to get lots of things combined together then you might end up with some very large frames.

For example imagine the following
```cpp
task<void> post_metric(string name, int value)
{
  char sendbuffer[16*1024];
  ...
}

task<void> post_metrics()
{
  std::array metrics { 
    post_metric("a", ...);
    post_metric("b", ...);
    post_metric("c", ...);
    ...
  };
  co_await when_all(begin(metrics), end(metrics));
}
```

This would require atleast 16kb for each `post_metric` and there might be multiple parallel calls coming from `post_metrics` all simultanously in-flight so you might get something much larger.

Now let's also consider the following

```cpp
task<void> func()
{
  if (cond)
  {
    co_await func1()
  }
  else
  {
    co_await func2();
    co_await post_metrics();
  }
}
```

In this case `func` could also include the size of `post_metrics` and even in the case where `cond` is `true`, this is recursive so `func1` and `func2` all could count towards the size of `func`.

You might also end up with optimizations which decide what to perform HALO on based on basic block probability, which might mean that potentially get one compiler which will HALO one branch and another compiler which HALOs the other leading to very inconsitent performance.

## Performance: Debug builds

Even the most basic coroutine has a lot of internal function calls.

```
task<void> func()
{
  co_return;
}
```

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

So that one function is now actually 13 functions behind the scenes optimizations many of them will be inlined, which you won't have those function call overheads.

However what about when you run in a debug build? Well all of those functions could end up existing in an unoptimized state.

## Design: The viral nature of stackless coroutines

C++ coroutines are stackless, this means that suspension only happens for a single function and not the entire callstack.

Because you typically want to allow suspension to propagate up a callstack to allow more functions to do things in parallel or because it expects to be passed into a dispatcher you must make all callers up the callstack also into coroutines.

This can happen in a wider and wider scope, all it takes is a few deeply nested function calls which want to be coroutines for it to quickly spread through the majority of the code base.

One way to avoid this is to be more willing to actually wait instead of suspending further up the call tree, but this could limit parallelism.

## Alternatives: How do stackful coroutines stack up

Stackful coroutines also called fibers, which offer a user-mode threading mechanism these require upfront spawning of a new thread then proceed to call other functions like normal, switching between fibers is also possible.

With stackful coroutines you can still have some of the similar security issues, however the range of lifetime issues shrinks down because you have an actual stack.

The performance issues are much different, you no longer have the overhead of each function call potentially being another memory allocation or a bunch of extra functions, however you miss the possibility of the optimizer pushing things all the way to nano coroutines when everything can be fully inlined.

Stackful coroutines are not viral in nature so if something wants to switch between coroutines it can be done without requiring the entire callstack to accept that switching. Which on the flip side means someone might suspend your function without you knowing.
