---
layout: post
title: 'Emergence: Memory management'
categories: [Emergence, Development Log]
tags: [Emergence, C++]
---

It is very important to have consistent memory model for your project from the start, because it is almost impossible
to change it later. In the post I will talk about [Emergence](https://github.com/KonstantinTomashevich/Emergence)
memory management library and its design decisions.

### Motivation

Having the right approach to managing memory is crucial for performance: it impacts both allocation cost and
cache coherency. Of course, you can just allocate all the memory on heap by using new, but it will lead to slow
allocation, cache misses and memory fragmentation. In my opinion, game engine without consistent memory management
model is like giant with legs made of straw. Therefore, I decided that 
[Emergence](https://github.com/KonstantinTomashevich/Emergence) must be built on top of consistent memory library
from the very beginning. Furthermore, I've decided to write my own memory management library to have full control
on how memory allocation works.

Writing your own memory management library is not a walk in the park and it's quite easy to get lost in details.
For example, if you're committed enough, you can try to ignore the existence of C standard library allocation
functions and write your own platform-specific versions. But it is a tedious and troublesome task. I've decided
to be less thorough about that and primarily focus on the API instead of implementation details: once API is done
right and whole project uses it, it shouldn't be difficult to change implementation details under the hood.

One important part about memory management model is that it should not only be consistent and easy to use,
but also provide tools for memory usage analysis, for example custom profiler. Having such tools is important
because they allow us to visualize how we're using memory and provide us with insight on what can be improved.
When I was searching for such tool to integrate into my memory management library, I've mostly encountered tools
that track memory allocations by file name or by one of the static predefined tags. It looked like it is not
good enough for [Emergence](https://github.com/KonstantinTomashevich/Emergence): I wanted to be able to build
dynamic memory usage flame graph and none of the tools I found was fully capable of doing it. So I've decided
that I need to write my own tool.

To summarize, I've decided to follow these principles:

- The most important part is consistent and easy-to-use API.
- C standard library functions is a good enough backend for the beginning.
- Library must provide API and a tool for memory usage profiling and analysis.

### Features

It is important to have a rich enough feature set. It should be neither too small nor too big: it should provide
enough control on how memory is allocated, but user must not sink in the sea of details. I've decided to go with
this feature set for [Emergence](https://github.com/KonstantinTomashevich/Emergence) memory management library:

- Unordered pool allocator for lighting-fast allocation and deallocation of objects of the same type.
- Ordered pool allocator, that has slow deallocation operation, but is more cache coherent and provides iteration
  over allocated chunks.
- Stack allocator for lighting-fast cache-coherent allocation of temporary trivial objects.
- Heap allocator for general purpose allocation.
- String interning implementation for ASCII strings.

It's better to provide memory usage profiler features as separate list to avoid blending with memory management
features:

- Memory usage is tracked for logical groups that can be created at any moment during runtime.
- Memory usage snapshot for all logical groups can be taken at any moment of time. 
  It makes sampling-based profiling possible.
- Profiler should be able to represent all memory operations as continuous set of events.
  It makes instrumentation-based profiling possible.
- There should be an API for serializing and deserializing memory usage tracks: initial memory snapshot and 
  continuous set of events that happened after this snapshot, 
- There should be a tool for viewing memory tracks.

Initially I also wanted to implement client-server profiler like [Tracy](https://github.com/wolfpld/tracy), but I've
decided to postpone it. It is not so difficult, but it requires time and I've decided that it is better to spend this 
time on other [Emergence](https://github.com/KonstantinTomashevich/Emergence) libraries.

### Allocators

I will start from describing implemented allocators and their implementation details.

#### Pool

Let's start by refreshing some theory about pool allocators. These allocators can be used to acquire chunks of memory
of predefined fixed size, for example 16-byte sized chunks. Every chunk is either used or free: used chunks contain
user data while free chunks are organized into linked list -- every free chunk stores link to the next free chunk.
Chunks are unified into pages that are usually quite big. Allocator manages several pages and allocates new ones if 
needed. Pages are stored as a linked list: each page has a pointer to next one. To sum up, references look like this:

![Pool allocator pages and pointers](/assets/img/EmergenceMemoryManagement/PoolAllocator.png)

Pool object stores references to the first page and to the first free chunk. And that's all! Well, almost all, we also
need to store chunk size, required chunk alignment and page capacity.

[Emergence::Memory service](https://github.com/KonstantinTomashevich/Emergence/tree/e8c37b6/Service/Memory) provides
two types of pools: OrderedPool and UnorderedPool that implement ordered and unordered pool allocation strategies.
We'll start from listing their operations and discus difference between them later.

- Construction: to construct pool you need to specify chunk size, required chunk alignment, preferred page chunk 
  capacity and allocation group that is used for profiling (more about allocation groups later). For example:

```c++
OrderedPool records (Memory::Profiler::AllocationGroup {"Records"_us},
                     _recordMapping.GetObjectSize (),
                     _recordMapping.GetObjectAlignment (),
                     /* page chunk capacity */ 128u);
```

... Pool operations ? ...

Now let's discuss the difference between ordered and unordered pool allocators. Basically, ordered pool guarantees
that page list and free chunk list are sorted by addresses in ascending order. You can see that image above
illustrates ordered pool. This ordering provides several benefits:

- Memory allocation is more cache coherent as free chunk list ordering minimizes amount of holes between used chunks.
- Pool shrinking is much more effective: it can be done in O(pageCount + freeChunkCount) instead of 
  O(pageCount * pageCapacity * freeChunkCount).
- Iteration over used chunks is much more effective: we do not need to iterate over whole free list to check whether
  chunk is free.

...

#### Stack

#### Heap

### String interning

### Profiling
