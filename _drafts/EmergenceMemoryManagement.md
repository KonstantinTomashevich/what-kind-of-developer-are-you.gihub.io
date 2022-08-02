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

### Allocators

### String interning

### Profiling
