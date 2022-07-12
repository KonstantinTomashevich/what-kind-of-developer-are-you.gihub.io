---
layout: post
title: 'Tutorial: String interning'
categories: [Tutorials, Memory]
tags: [Tutorials, C++]
---

Is there a lot of instances of the same string in your application that are wasting lots of space? 
Or are you checking equality or hashing strings quite often and wondering how to do it more efficiently?
Fortunately, string interning comes to the rescue!

### Motivation

Let's start from describing two most common cases where string interning helps us a lot.

Strings are much more informative for humans than numbers. It is easy to understand what we're dealing with when
we're seeing `unitType = "Knight"`, but it is much harder if we're seeing `unitType = 42`. Therefore string is
the best variant when we need to refer to a well-known object like config entry. But it also means that such
string will be copied lots of times! What if every unit has `std::string unitType` and there is 100 units? We
would waste `99 * X` bytes where `X` is the average config entry id length. And what if there is more units?
Not looking good, eh?

Checking string equality is much less appealing than checking number equality -- it's `O(string length)` instead
of `O(1)` after all. The same applies to string hashing -- it is also `O(string length)`. So if we are checking
string equality or hashing strings a lot, we might end up with a bottleneck: string iteration would consume 
much more time than actual algorithm. But there should be a way to avoid this problem, right?

Now it is the time to unravel the mystery of string interning: all interned strings are immutable and are stored in a 
special storage that contains not more than 1 instance of a string. So interned strings are never duplicated and 
therefore don't waste memory! But that's not all: if string is never duplicated then we're free to use pointer 
equality check instead of string equality check and by that we can check if strings are equal in `O(1)`. 
The same goes for hashing: why not just use unique string pointer as hash?

### Simple implementation

To get better understanding of string interning concept we will start from simple implementation:
- We will ignore string encoding and just use `char`s.
- We will ignore anti-defragmenetation tecnique.
- We will ignore unused string deallocation.
- We will ignore multithreaded access.

Now let's examine our interned string class:
```c++
// InternedString class instances share immutable string
// that is allocated inside string pool.
class InternedString final
{
public:
    InternedString () = default;

    // Let's accept both c-style strings and modern string views
    // as constructor arguments.
    explicit InternedString (const char *_string) noexcept;

    explicit InternedString (const std::string_view &_string) noexcept;

    // Dereference operator will return pointer to actual string.
    const char *operator* () const noexcept;

    // Interned string hash that is computed in O(1).
    [[nodiscard]] uintptr_t Hash () const noexcept;

    // Interned string equality check that is done in O(1).
    bool operator== (const InternedString &_other) const;

    bool operator!= (const InternedString &_other) const;

private:
    // Pointer to location of actual string in memory that
    // might be shared between many InternedString instances.
    const char *value = nullptr;
};
```
{: file='InternedString.hpp'}

`InternedString` interface is quite simple and straighforward, isn't it? Let's switch to implementation:

```c++
#include <cassert>
#include <cstring>
#include <unordered_set>

#include "InternedString.hpp"

// Registrar function is the most important part in any string
// interning implementation. This function checks whether given
// value is already interned and interns it if it's not.
//
// There are 2 common questions that are answered by registrar function:
// - How value lookup is implemented?
// - How interned values are allocated?
//
// Answers to these questions describe strengths and weaknesses of
// registrar function.
static const char *RegisterValue (const std::string_view &_value)
{
    // There is no sense to allocate space for empty strings.
    if (_value.empty ())
    {
        return nullptr;
    }

    // In this simple implementation we use unordered set to
    // store views of all interned values. It is a trivial
    // solution, but it is quite common nevertheless.
    static std::unordered_set<std::string_view> stringRegister;

    // Lets check whether value is already interned.
    auto iterator = stringRegister.find (_value);

    if (iterator == stringRegister.end ())
    {
        // Value is not interned: allocate memory and insert it.
        char *space = new char[_value.size() + 1u]; // We need to add 1 byte for null-terminator.
        strcpy (space, _value.data ());
        auto [insertionIterator, result] = stringRegister.emplace (space);
        assert (result);
        return insertionIterator->data ();
    }

    // Value is already interned: we can just return pointer to it.
    return iterator->data ();
}

// In constructors, we're getting pointers to interned values (and interning this value if needed).
InternedString::InternedString (const char *_string) noexcept : InternedString (std::string_view {_string})
{
}

InternedString::InternedString (const std::string_view &_string) noexcept : value (RegisterValue (_string))
{
}

const char *InternedString::operator* () const noexcept
{
    return value;
}

uintptr_t InternedString::Hash () const noexcept
{
    // We can use pointer as hash result,
    // because interned strings are never duplicated in memory.
    return reinterpret_cast<uintptr_t> (value);
}

bool InternedString::operator== (const InternedString &_other) const
{
    // We can compare pointers directly,
    // because interned strings are never duplicated in memory.
    return value == _other.value;
}

bool InternedString::operator!= (const InternedString &_other) const
{
    return !(*this == _other);
}
```
{: file='InternedString.cpp'}

And that's everything we need to do in order to implement string interning in a simple manner! 
You can already test whether it works correctly:

```c++
InternedString first {"Hello, world!"};
InternedString second {"Welcome-welcome!"};
InternedString third {"Hello, world!"};

std::cout << "first  = " << *first << std::endl;
std::cout << "second = " << *second << std::endl;
std::cout << "third  = " << *third << std::endl << std::endl;

std::cout << "first  == second: " << (first == second ? "true" : "false") << std::endl;
std::cout << "second == third : " << (second == third ? "true" : "false") << std::endl;
std::cout << "first  == third : " << (first == third ? "true" : "false") << std::endl;
```

Expected output:

```
first  = Hello, world!
second = Welcome-welcome!
third  = Hello, world!

first  == second: false
second == third : false
first  == third : true
```

Of course, this implementation is trivial and ignores some important concepts like unused string 
deallocation and string-related memory defragmentation. We will explore these 2 topics below.

### Destiny of unused interned strings

In our trivial implementation interned strings are never truly deallocated: they persist for whole program execution.
Is it a real problem? Generally speaking, it is, but lots of implementations stick to no-deallocation solution for
several reasons:

- Usually interned strings are well-known values like constants or some ids. In this case they are either always used
  anyway or quickly become used again after short period of being unused. So there is no sense to deallocate them.
- Interned strings are usually quite small, therefore reference counters might significantly increase memory usage.
- Reference counting makes copy constructor, move constructor and assignments non-trivial: we can not just copy
  pointer like we're doing in no-deallocation solution. Performance impact is small, but working with non-trivial 
  types is more difficult from the architectural point of view.
- If strings are ever deallocated, it's not possible to use addresses as stable hashes anymore! If some generic
  structure stores hashes without values, interned string might become unused and will be deallocated, leaving
  its address and therefore its hash value free to grab for new interned string, which would invalidate 
  affected hash structure.
  
Due to these reasons [Emergence](https://github.com/KonstantinTomashevich/Emergence) implementation of this concept
never deallocates unused interned strings. [Press Fire Games internal engine](https://www.pressfire.com/technologies)
also never deallocates unused interned strings. But Java uses garbage collector to deallocate interned strings. 
So it is up to you to decide whether you need to do something about unused interned strings or not.

### Fighting defragmentation

Strings are small objects with wide variety of different sizes. When we're allocating strings through global heap
allocator like `new` or `malloc` at random moments, we're creating perfect ground for memory defragmentation.
Of course, it's possible to solve this problem too!

The core idea is to preallocate big memory blocks where interned strings will be stored and then use custom
allocator to allocate strings inside these blocks. If there is not enough memory for interning new string,
additional memory block will be allocated. If we're not deallocating unused interned strings, stack allocator is
both the simplest and the most efficient custom allocator for this purpose: each block has its own stack allocator
that pushes new strings into itself. In this case we can call the whole structure for managing memory for interned
strings **a pool of stacks**: we're using pool-based allocator to allocate new stack allocators that will allocate
space for interned strings. It might sound monstrous, but it is actually quite easy to implement and use.

But what changes when we're deallocating unused interned strings? Not a lot, actually! Sure, we cannot use stack
allocators anymore, but we can use heap allocators instead. Therefore, the whole structure would become 
**a pool of heaps**. But beware of defragmentation inside interned string heaps: if strings are allocated and
deallocated often we might still encounter it inside these heaps.

Instead of modyfying our simple implementation I will direct reader right into 
[Emergence](https://github.com/KonstantinTomashevich/Emergence) implementation of interned strings called 
`UniqueString` -- there are 
[header](https://github.com/KonstantinTomashevich/Emergence/blob/daf48ca/Service/Memory/Implementation/Original/Memory/Original/UniqueString.hpp) 
and [object](https://github.com/KonstantinTomashevich/Emergence/blob/daf48ca/Service/Memory/Implementation/Original/Memory/Original/UniqueString.cpp) 
files. `UniqueString`s are never deallocated and therefore use **a pool of stacks** approach.

Hope you've enjoyed reading! :)
