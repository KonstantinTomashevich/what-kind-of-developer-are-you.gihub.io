---
layout: post
title: 'Emergence: Reflection'
categories: [Emergence, Development Log]
tags: [Emergence, C++]
---

Reflection system is an important part of every engine, but there is no standard solution for that in C++: there are
lots of libraries with their pros and cons.  I've decided to write my own reflection for 
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

Before diving into details it is good to inform the reader what our reflection can and cannot achieve. Let's start
from the features:

- Lightweight field-only reflection for [standard layout types](https://en.cppreference.com/w/cpp/named_req/StandardLayoutType).
- Field identifiers and handles provide O(1) access to field info.
- Conditional field iteration is aware of unions and inplace vectors in the context of given data object.
- One-bit flags are supported.
- Field identifiers are stable and can be used for serialization.
- Patches provide generic way to apply changes to an object.

But there are some limitations too:

- Only [standard layout types](https://en.cppreference.com/w/cpp/named_req/StandardLayoutType) are supported.
- Method registration is not supported.
- Field meta registration is not supported.

I would like to add my personal opinion about field meta, because it's not so difficult to implement, but nevertheless
I've decided to omit this feature. The thing is that usually meta adds unnecessary coupling between data and its usage.
It looks very convenient and useful at the first glance, but than you might end up with transform implementation that
depends on networking implementation in order to add networking-related meta. Or all meta types might be declared
as one big heap and included everywhere. But that is not the biggest problem: what if you need different networking
or interpolation or some other meta-based settings for shared data type in two different projects? For example,
one project needs to replicate scale and the other one doesn't. So I've decided to avoid using meta at all in order
to prevent such problems from appearing.

### Terms

- `Mapping` -- structure than contains all the info about registered data type.
- `MappingBuilder` -- special object that provides API for `Mapping` creation.
- `Field` -- handle to the information about one concrete field, belongs to `Mapping`.
- `FieldId` -- special id that is unique in context of `Mapping` and allows to get `Field` in O(1).
- `Patch` -- archive of changes, linked to `Mapping`, that can be applied to an arbitrary object.
- `PatchBuilder` -- special object that provides API for `Patch` creation.

### Registering types

Let's explore how type registration works in [Emergence](https://github.com/KonstantinTomashevich/Emergence).

Actually, there is two registration APIs: verbose core API that is represented by `MappingBuilder` class and
more user-friendly macro-based approach from `MappingRegistration` header. The first one is basic API that 
allows user to register types whatever way user wants. The second one is built on top of the first one and
represents the approach [Emergence](https://github.com/KonstantinTomashevich/Emergence) uses to register types.
Choice to have both the verbose basic approach and the simplified macro approach was made in order to achieve
flexibility: if user is satisfied with macro-based approach, he can use this approach, otherwise it is
possible to implement custom registration approach which will still produce compatible results.

`MappingBuilder` is essentially an implementation of classic Builder pattern for `Mapping`: it encapsulates
registration process and ensures that all `Mapping` instances are finished and ready to use. Registration through
`MappingBuilder` API is quite simple, but verbose. It looks like this:

```c++
MappingBuilder builder.
builder.Begin ("MyComponent"_us, sizeof (MyComponent), alignof (MyComponent));
builder.SetConstructor (...);
builder.SetDestructor (...);

// ...
FieldId fieldA = builder.RegisterInt16 ("a"_us, offsetof (MyComponent, a));
FieldId fieldB = builder.RegisterFloat ("b"_us, offsetof (MyComponent, b));
FieldId fieldC = builder.RegisterNestedObject ("c"_us, offsetof (MyComponent, c), typeCMapping);
// ...

Mapping typeMyComponentMapping = builder.End ();
```

As you can see, this approach only cares about putting fields into `Mapping`, but not about storing `Mapping`s and
`FieldId`s. Also, it forces us to duplicate field names and types. Obviously, this API wasn't designed for direct
use -- it was designed to be the simplest base API for building custom better APIs. And 
[Emergence](https://github.com/KonstantinTomashevich/Emergence) offers such API out of the 
box in `MappingRegistration` header.

In `MappingRegistration` approach every type stores information about itself in special `Reflection` structure
and provides instance of this structure through static `Reflect` method, like this:

```c++
struct MyComponent final
{
    // Fields of our component.
    int16_t a = 0;
    float b = 0.0f;
    SomeClass c;

    // Reflection structure that lists all reflected
    // information about this type.
    struct Reflection final
    {
        FieldId a;
        FieldId b;
        FieldId c;
        Mapping mapping;
    };
    
    // Static method for getting static reflection info.
    static const Reflection &Reflect () noexcept;
};
```

Duplicating field names like this might still look a bit clumsy, but this approach is not only defining how we are
storing results of registration process, but also provides universal form of access to them making registration 
very easy:

```c++
const MyComponent::Reflection &MyComponent::Reflect () noexcept
{
    static Reflection reflection = [] ()
    {
        EMERGENCE_MAPPING_REGISTRATION_BEGIN (MyComponent);
        EMERGENCE_MAPPING_REGISTER_REGULAR (a);
        EMERGENCE_MAPPING_REGISTER_REGULAR (b);
        EMERGENCE_MAPPING_REGISTER_REGULAR (c);
        EMERGENCE_MAPPING_REGISTRATION_END ();
    }();

    return reflection;
}
```

As you can see, everything about registered fields is deduces automatically and we're not duplicating anything now!
It's neat, isn't it?

You can also notice that macro is called `EMERGENCE_MAPPING_REGISTER_REGULAR`, instead of just 
`EMERGENCE_MAPPING_REGISTER`. It is called like that because we have two "irregular" field archetypes: inplace
strings and plain blocks of memory. Right now we can not safely deduce whether given field is inplace string or
plain block, therefore we're registering them though separate macros: `EMERGENCE_MAPPING_REGISTER_BLOCK` and
`EMERGENCE_MAPPING_REGISTER_STRING`.

But what about arrays? Thankfully, there is solution for that too!

```c++
// Header.

struct ArrayComponent final
{
    std::array<uint32_t, 32u> ints;

    struct Reflection final
    {
        // We declare array of fields inside our reflection structure.
        std::array<FieldId, 32u> ints;
        Mapping mapping;
    };
    
    static const Reflection &Reflect () noexcept;
};

// Object.

const ArrayComponent::Reflection &ArrayComponent::Reflect () noexcept
{
    static Reflection reflection = [] ()
    {
        EMERGENCE_MAPPING_REGISTRATION_BEGIN (ArrayComponent);
        EMERGENCE_MAPPING_REGISTER_REGULAR_ARRAY (ints);
        EMERGENCE_MAPPING_REGISTRATION_END ();
    }();

    return reflection;
}
```

### Field projection

### Conditional field iteration

### Mapping implementation details

### Patches

Hope you've enjoyed reading! If you have any suggestions, reach me through telegram or email.
