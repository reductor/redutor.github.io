---
layout: post
title:  "Pass-by-value vs Pass-by-reference"
date:   2022-06-27 19:00:00 +1000
categories: cpp
---

Let's dig into the age old question, should you pass-by-value or pass-by-reference in C++? (or by pointer in C)

This blog post is mostly a re-post of a [reddit comment](https://www.reddit.com/r/cpp/comments/odd2kz/is_passbyvalue_slower_than_passbyrvaluereference/h40hao4/) that I made on r/cpp about pass-by-value and pass-by-reference, with some minor improvements, to make it easier to reference and save.

The answer isn't as easy as it might seem, it depends on the Application Binary Interface (ABI) and your use-cases, there isn't a one size fits all answer, this is even more the case for anything which is built to be cross platform.

First it's probably good to break the problem down into two parts (focusing solely on performance, ignoring readability and maintainability which should often be more important)

 * The language construct costs (copying, moving, etc)
 * Compiler implications (aliasing, pointer provenance, etc)
 * The ABI (the stack, registers, etc)

## Language constructs

tl;dr Pass-by-value can end up with unnecessary copies

To help understand the language constructs, here is a bit of a refresher on value types

<table>
<tbody>
<tr><th>Calling with lvalue</th>
<td>
```cpp
T data;
func( data );
```
</td></tr>
<tr><th>
Calling with xvalue (type of rvalue)<br/>
NOTE: `data` can be used after `func`
</th>
<td>
```cpp
T data;
func( std::move( data ) );
```
</td>
</tr><tr>
<th>
Calling with prvalue (type of rvalue)<br/>
NOTE: The input arg can't be used after `func`
</th>
<td>
```cpp
func( T{} );
```
</td></tr>
</tbody>
</table>

Now let's see how these value types impact the what happens before and after the call

:**Behavior before the call**:||||
&nbsp; | `void func(T)` | `void func(const T &)` | `void func(T &&)`
:--|:--|:--|:--
**Calling with lvalue** | Copies | Reference | N/A
**Calling with xvalue** | Moves | Reference | Reference
**Calling with prvalue** | In-place construct | Reference | Reference

:**Behavior after if the caller does not move or copy**:||||
^^```arg.something()```:||||
&nbsp; | `void func(T)` | `void func(const T &)` | `void func(T &&)`
:--|:--|:--|:--
**Calling with lvalue** | Unnecessary copy | No overhead | N/A
**Calling with xvalue** | Unnecessary move | No overhead | No overhead
**Calling with prvalue** | No overhead | No overhead | No overhead

:**Behavior after if the caller moves**:||||
^^```other = std::move( arg );```:||||
&nbsp; | `void func(T)` | `void func(const T &)` | `void func(T &&)`
:--|:--|:--|:--
**Calling with lvalue** | Copy and move | N/A | Compile error
**Calling with xvalue** | Two moves | N/A | One move
**Calling with prvalue** | One move | N/A | One move

:**Behavior after if the caller copies**:||||
^^```other = arg;```:||||
&nbsp; | `void func(T)` | `void func(const T &)` | `void func(T &&)`
:--|:--|:--|:--
**Calling with lvalue** | Two copies | One copy | N/A
**Calling with xvalue** | Move and Copy | One copy | One copy
**Calling with prvalue** | One copy | One copy | One copy


## Compiler implications

tl;dr Pass by reference can create additional aliasing situations

In order for the compiler to do optimizations it must make assumptions based on the code it is provided, one of the important ones is understanding who owns what pointer (or reference) and if this pointer/reference will be modified over a calling boundary.

In order to understand some of the implications of this aliasing, let's look at a simple example ([compiler explorer link](https://godbolt.org/z/dEbdEosb1)).

```cpp
struct MyObject
{
    int val;
};

void other_func();
int by_ref( const MyObject & v )
{
    int x = v.val;
    other_func();
    int y = v.val;
    return x + y;
}

int by_value( MyObject v )
{
    int x = v.val;
    other_func();
    int y = v.val;
    return x + y;
}
```

With the `by_ref` function the compiler is unable to determine if `v.val` could have changed during the call to `other_func` (e.g. it might have changed as a referenced to a global variable inside `other_func`), because of this it is unable to optimize this code to do a single load of `v.val`, with the `by_value` implementation it knows the value can not change, as nothing else references it, it is the sole owner.

This aliasing can also happen when you have multiple arguments, even when the compiler is fully aware of the entire function here is another example ([compiler explorer](https://godbolt.org/z/34Wq5W4d8))

```cpp
void two_arg_by_ref( const MyObject & input, MyObject & output )
{
    output.val += input.val * 2;
    output.val += input.val * 4;
}

void two_arg_by_value( MyObject input, MyObject & output )
{
    output.val += input.val * 2;
    output.val += input.val * 4;
}
```

With this example the compiler is unable to determine if `input` and `output` point to the same object so the value could be changing for each increment, meaning the compiler can not remove the extra loads and stores. 

With this next one it might be fairly clear that they could reference the same object, how about when different argument types are used for outputting the value. ([compiler explorer](https://godbolt.org/z/nbzKvh1Kh))

```cpp
void two_arg_by_char_ref( const MyObject & input, char & output )
{
    output += input.val * 2;
    output += input.val * 4;
}

void two_arg_by_short_ref( const MyObject & input, short & output )
{
    output += input.val * 2;
    output += input.val * 4;
}

void two_arg_by_int_ref( const MyObject & input, int & output )
{
    output += input.val * 2;
    output += input.val * 4;
}
```

What might come as as surprise is that the same aliasing can exist for `two_arg_by_int_ref` because `output` might be pointing to the same value as `input.val`, however the probably more unsuspecting one here is `two_arg_by_char_ref` it could possibly also point to `input.val` because it's well-defined behavior to use `reinterpret_cast` on any object to treat it as a `char` (or ideally `std::byte`).

Thankfully with the `two_arg_by_short_ref` version this would be undefined behavior for the `short` to point to the same object as it's a different type that isn't `char` so it's free to remove the unnecessary load/stores.

There is also the caller side of things which is impacted, let's take a look at another example ([compiler explorer](https://godbolt.org/z/oT56M5To8))

```cpp
void by_ref(const MyObject&);
void by_value(MyObject);

int use_by_ref()
{
    MyObject obj{10};
    by_ref(obj);
    return obj.val;
}

int use_by_value()
{
    MyObject obj{10};
    by_value(obj);
    return obj.val;
}
```

With this the `use_by_ref` example even though it is calling a function which is `const MyObject&` that function can still change the actual value (`const` isn't as meaningful as you would hope) so it must load the value of `obj.val` after the return while the `use_by_value` can safely assume that it will not change so does not need to load the value again.

## The ABI

tl;dr Each platform is different, System V (*nix) does better with aggregate handling, Windows frequently leads to Invisible Reference.

This is heavily dependent on the platform, most of my experience is around x86 and most information here is specific to x86-64 and related to how MSVC and Unix System V (Linux, BSD, Mac, etc), to start with I'll break things down into a few categories of what can happen

 * Argument is passed in register(s)
 * Argument is passed as a reference; reuse existing memory location (Traditional reference)
 * Argument is passed as a reference; new memory location just for argument (Invisible reference)

These are in order from what is typically faster to slower, however assumptions about speed can have many exceptions and change over time, so if your making performance decisions use benchmarks not assumptions.

In order to show how things work its good to have some types to work with

```cpp
struct int5
{
    int v1, v2, v3, v4, v5;
};

struct int4
{
    int v1, v2, v3, v4;
};

struct int3
{
    int v1, v2, v3;
};

struct int2
{
    int v1, v2;
};

struct int1
{
    int v1;
};

struct char3
{
    char x, y, z;
}
```

Now with these types let's see how different ABIs handle them (reg is short for register here)

&nbsp; | MSVC x64 ABI | Sys V ABI
:--|:--|:--
`void func(int5)`| Invisible reference in 1-reg | Invisible reference in 1-reg
`void func(int4)`| Invisible reference in 1-reg | Packed in 2-regs
`void func(int3)`| Invisible reference in 1-reg | Packed in 2-regs
`void func(int2)`| Packed in 1-reg | Packed in 1-reg
`void func(int1)`| Packed in 1-reg | Packed in 1-reg
`void func(char3)`| Invisible reference in 1-reg | Packed in 1-reg
`void func(const int3 &)`| Reference in 1-reg | Reference in 1-reg
`void func(const int2 &)`| Reference in 1-reg | Reference in 1-reg
`void func(const int1 &)`| Reference in 1-reg | Reference in 1-reg
`void func(const char3 &)`| Reference in 1-reg | Reference in 1-reg
`void func(int3 &&)`| Reference in 1-reg | Reference in 1-reg
`void func(int2 &&)`| Reference in 1-reg | Reference in 1-reg
`void func(int1 &&)`| Reference in 1-reg | Reference in 1-reg
`void func(char3 &&)`| Reference in 1-reg | Reference in 1-reg

As you can see it gets pretty complex, references are all references however when you pass-by-value things vary based on different platforms, different sizes, there is many more complex situations, especially when it comes to System V ABI's unpacking of structs, which can make it fairly effective passing by value. Windows also has many attributes which can be used to improve and change it's ABI for example `__vectorcall` which allow it to unpack some structures similar behavior to System V. 

Now the "Invisible Reference" thing here is important to understand a bit better, in many cases this will be a copy  of the original data so it has a memory address to reference as the initial value is not allowed to change so referencing it is dangerous.

This also goes for the standard "Reference" if you are using an l-value reference from something which would typically exist in registers (in x86-64 this is where most your code and calculations hopefully live).

Even with all of this information it's nearly impossible to define a one size fits all approach for how functions should accept arguments when it comes to performance, you should just try to consider the basics of avoiding copying heavy to copy objects and moving heavy to move objects (which hopefully you never have heavy to move objects), and focusing on readability and maintainability unless you really need that last drop of performance and this is your bottleneck then and only then bother improving it, and do it with benchmarks and on all your supported platforms.

As a follow-up to this I plan to dig further into Vector math libraries commonly seen in 3D graphics and games and the implications an ABI can have.