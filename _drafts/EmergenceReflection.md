---
layout: post
title: 'Emergence: Reflection'
categories: [Emergence, Development Log]
tags: [Emergence, C++]
---

Reflection system is an important part of every engine, but there is no standard solution for that in C++: there are
lots of libraries with their pros and cons. I've decided to write my own reflection for 
[Emergence](https://github.com/KonstantinTomashevich/Emergence) and I will describe it in this post.

### Motivation

It's impossible to underestimate how useful good reflection system could be: reflection provides wide variety of
tools to make code more flexible, robust, elegant and give it some kind of consciousness. But everything comes
with a price: reflection might consume a lot of compile time or be rather slow from performance point of view.
Therefore I've decided to write my own reflection system that is fine-tuned for
[Emergence](https://github.com/KonstantinTomashevich/Emergence) project needs and is as lightweight as possible.

When you're starting to design your own solution it is crucial to define usage scope and examine expected use cases.
Otherwise, it is easy to build complex orbital gun with lots of unusable features. For the 
[Emergence](https://github.com/KonstantinTomashevich/Emergence) main usage scope of reflection is managing data:
observing field values, changing them or iterating through the whole data structure. There are some examples:

- Indices observe specified fields and process their values changes.
- [Celerity](https://github.com/KonstantinTomashevich/Emergence/tree/a275a21/Library/Public/Celerity) events use 
  reflection to extract and copy useful data from objects.
- Serialization library, which is not written yet, uses reflection to iterate over data structure to serialize or 
  deserialize it.
- Reflection is used as a high-level parameter for generic tasks like
  [Celerity::Assembly](https://github.com/KonstantinTomashevich/Emergence/tree/a275a21/Library/Public/Celerity/Extension/Assembly) 
  extension.

After examination of these use cases I've decided that:

- There is no need to reflect methods: reflecting fields, no-argument constructor and destructor is enough.
- We can focus on [standard layout types](https://en.cppreference.com/w/cpp/named_req/StandardLayoutType) 
  and ignore other types as in the most cases data is stored in such types.
- Reflection must provide API for O(1) field access: no search, no nested hierarchy traversing.
- Reflection headers must be lightweight and template-free for optimal compile time.
- Reflection must be aware of unions and inplace vectors: otherwise iteration would show irrelevant fields.

Thats how the idea of 
[StandardLayoutMapping service](https://github.com/KonstantinTomashevich/Emergence/tree/a275a21/Service/StandardLayoutMapping) 
was born.

### Features overview

### Registering types

### Conditional field iteration

### Mapping implementation details

### Patches

Hope you've enjoyed reading! :)
