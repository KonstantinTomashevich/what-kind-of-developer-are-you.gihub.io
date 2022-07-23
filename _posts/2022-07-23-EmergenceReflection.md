---
layout: post
title: 'Emergence: Reflection'
date: 2022-07-23 13:50:00 GMT+3
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
- Serialization library, which is not finished yet, uses reflection to iterate over data structure to serialize or 
  deserialize it.
- Reflection is used as a high-level parameter for generic pipeline tasks like
  [Celerity::Assembly](https://github.com/KonstantinTomashevich/Emergence/tree/a275a21/Library/Public/Celerity/Extension/Assembly) 
  extension.

After examination of these use cases I've decided that:

- There is no need to reflect methods: reflecting fields, no-argument constructor and destructor is enough.
- We can focus on [standard layout types](https://en.cppreference.com/w/cpp/named_req/StandardLayoutType) 
  and ignore other types because in the most cases data is stored in such types.
- Reflection must provide API for O(1) field access: no search, no nested hierarchy traversing.
- Reflection headers must be lightweight and template-free for optimal compile time.
- Reflection must be aware of unions and inplace vectors: otherwise iteration would show irrelevant fields.

Thats how the idea of 
[StandardLayoutMapping service](https://github.com/KonstantinTomashevich/Emergence/tree/a275a21/Service/StandardLayoutMapping) 
was born.

### Features overview

Before diving into details it is good to inform the reader what my reflection system can and cannot achieve. 
Let's start from the features:

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
more user-friendly macro-based approach from `MappingRegistration` header. The first one is a basic API that 
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
    
    // Static method for getting reflection info.
    static const Reflection &Reflect () noexcept;
};
```

`Reflection` structure defines how we store reflection data. We still need to duplicate field names: it's far from 
ideal, but I wasn't able to come up with any solution that avoids it entirely. The best thing about this technique of 
storing reflection data is that it provides universal form of access to this data making registration very easy:

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

As you can see, everything about registered fields is deduced automatically and we're not duplicating anything now!
It's neat, isn't it?

You can also notice that macro is named `EMERGENCE_MAPPING_REGISTER_REGULAR` instead of just 
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

Imagine that we have structure that contains other structures as fields:

```c++
struct Inner final
{
    float x = 0.0f;
    float y = 0.0f;

    // ... Reflection ...
};

struct Complex final
{
    Inner a;
    Inner b;
    Inner c;

    // ... Reflection ...
};
```

Accessing field `x` of `Inner` by `FieldId` in O(1) looks trivial: we can just store fields in vector. The same goes
for field `b` of `Complex`. But what about field `x` of field `c` of `Complex`? We still want to access it in O(1),
but there is no trivial way to do it.

Technique to solve this issue is called field projection: if `x` is field of `a` and `a` is field of `n`, then
`a.x` is a field of `n` too. Therefore, our `Complex` structure has whole bunch of fields after projection:
`a`, `a.x`, `a.y`, `b`, `b.x`, `b.y`, `c`, `c.x` and `c.y`. In order to make this technique mathematically complete
we also need to define projection function: `FieldId Project (FieldId rootObjectField, FieldId nestedObjectField)`.
For example, `FieldId` of `a.x` is equal to `Project (Complex::Reflect ().a, Inner::Reflect ().x)`. As these projected
ids are stable we can safely cache them and pass as parameters whenever we need to. In 
[Emergence](https://github.com/KonstantinTomashevich/Emergence) this function is
implemented as `Emergence::StandardLayout::ProjectNestedField` and is actually just a sum of `rootObjectField` and 
`nestedObjectField`! We are able to make it that simple because we are doing projecting right after structure
field registration.

Like any other solution, field projection technique has its pros and cons.

Pros:
- It provides O(1) access to any field of 
  [standard layout type](https://en.cppreference.com/w/cpp/named_req/StandardLayoutType).
- It makes algorithms that make use of linearized structure data, for example binary serialization, much easier to 
  implement, because projection automatically makes reflection data linear.

Cons:
- It uses lots of memory because of field info duplication: only offsets and ids are changed during projection process,
  but we need to duplicate whole infos.
- It makes algorithms that make use of tree-like structure data, for example YAML serialization, a bit more difficult
  to implement, because they need to skip projected fields everywhere.

For [Emergence](https://github.com/KonstantinTomashevich/Emergence), O(1) access to any field was the main reason
to stick to this technique: reflection data is used extensively by storage management logic, for example record 
indexing, therefore reflection access speed defines how effective storage management is.

### Conditional field iteration

It is important to provide user with API that allows to iterate over contextually relevant fields. For example,
let's take a look at that structure:

```c++
struct CollisionGeometry final
{
    CollisionGeometryType type;
    union
    {
        Math::Vector3f boxHalfExtents;
        float sphereRadius;
        struct
        {
            float capsuleRadius;
            float capsuleHalfHeight;
        };
    };

    // ... Reflection ...
};
```

Technically it has 5 fields, but not more than 3 fields are contextually relevant at the same time because
all fields except one are inside union. For example, for spheres only `type` and `sphereRadius` fields are 
relevant. If we're using reflection to serialize object or log it somewhere, we need to skip these irrelevant fields.
The same thing is true for inplace vectors: if vector can hold up to 6 elements, but holds only 2 right now, we should
not iterate over garbage memory, stored in last 4 elements.

At first, I though that unions and inplace vectors are different cases and should be handled in a different way, but 
then I came up with the idea of conditional field iteration: union switch value or count of elements is just an
argument to conditional expression that decides whether field is visible or not in the current context. And there is
no need to waste memory by attaching condition to every field: we only need to specify intervals where conditions
are active. Also, it makes sense to organize conditions as a stack: we are generally adding and removing them while
registering fields.

It all might sound too abstract and high level, so let's switch to actual examples. This is registration function
for our `CollisionGeometry`:

```c++
const CollisionGeometry::Reflection &CollisionGeometry::Reflect () noexcept
{
    static Reflection reflection = [] ()
    {
        EMERGENCE_MAPPING_REGISTRATION_BEGIN (CollisionGeometry);
        EMERGENCE_MAPPING_REGISTER_REGULAR (type);

        EMERGENCE_MAPPING_UNION_VARIANT_BEGIN (type, 0u);
        EMERGENCE_MAPPING_REGISTER_REGULAR (boxHalfExtents);
        EMERGENCE_MAPPING_UNION_VARIANT_END ();

        EMERGENCE_MAPPING_UNION_VARIANT_BEGIN (type, 1u);
        EMERGENCE_MAPPING_REGISTER_REGULAR (sphereRadius);
        EMERGENCE_MAPPING_UNION_VARIANT_END ();

        EMERGENCE_MAPPING_UNION_VARIANT_BEGIN (type, 2u);
        EMERGENCE_MAPPING_REGISTER_REGULAR (capsuleRadius);
        EMERGENCE_MAPPING_REGISTER_REGULAR (capsuleHalfHeight);
        EMERGENCE_MAPPING_UNION_VARIANT_END ();

        EMERGENCE_MAPPING_REGISTRATION_END ();
    }();

    return reflection;
}
```

As you can see, there is nothing about conditions yet: `MappingRegistration` hides them under macros for readability.
But what is hidden under `EMERGENCE_MAPPING_UNION_VARIANT_BEGIN` and `EMERGENCE_MAPPING_UNION_VARIANT_END` macros?

```c++
#define EMERGENCE_MAPPING_UNION_VARIANT_BEGIN(_selectorField, _switchValue)                                            \
    builder.PushVisibilityCondition (reflectionData._selectorField,                                                    \
                                     Emergence::StandardLayout::ConditionalOperation::EQUAL, _switchValue)

#define EMERGENCE_MAPPING_UNION_VARIANT_END() builder.PopVisibilityCondition ()
```

These macros are just operating with conditions through `MappingBuilder` interface. When union begins, we're pushing
condition that says: fields below are visible only when `_selectorField` is equal to `_switchValue`. And when union
ends we're just popping this condition out. There are also other conditional operations, for example inplace 
vector registration makes use of `>` (see `EMERGENCE_MAPPING_REGISTER_REGULAR_VECTOR` macro):

```c++
builder.PushVisibilityCondition (_sizeField, ConditionalOperation::GREATER, index);
```

This condition says that fields below should be visible only when vector size aka `_sizeField` is greater than element
index. Just like that: nothing less, nothing more. Simplicity of this technique makes it very flexible: it is not
just for unions and inplace vectors, it can be used anywhere if user needs it.

Of course, conditional iteration is less performance-friendly that plain iteration, therefore `Mapping` has
two iteration options: `Begin`/`End` for non-conditional iteration and `BeginConditional`/`EndConditional` for
conditional iteration. You can use conditional iteration just like the usual one:

```c++
for (auto iterator = _mapping.BeginConditional (_object), end = _mapping.EndConditional (); 
     iterator != end; ++iterator)
{
    StandardLayout::Field field = *iterator;
    /// ...
}
```

There is one important thing about conditional iteration performance: you might think that it is very slow due to
condition stack operations -- stack push/pops, memory allocation for that and so on. But it is actually not true!
Because push/pop order is always the same, we can get rid of stack operations during conditional iteration by 
baking the operations during type registration. I will not dive into details of this algorithm here, but keep in
mind: conditional iteration is not as slow as you might think.

### Patches

Sometimes it is useful to store difference between two possible states of an object. For example, it can be used
for prefab system to apply prefab values to freshly constructed objects. 
[Emergence](https://github.com/KonstantinTomashevich/Emergence) supports this through patches feature: at any moment
user can create `Patch` using `PatchBuilder` and then apply this `Patch` whenever it is needed.

There is two ways to create a patch. First one is to manually list all the differences like that:

```c++
PatchBuilder builder;
builder.Begin (Player::Reflect ().mapping);
builder.SetBit (Player::Reflect ().alive, false);
builder.SetBit (Player::Reflect ().poisoned, true);
Patch patch = builder.End ();
```

The second one is much simpler: it automatically creates patch from difference between two objects. For example:

```c++
Player initial;
Player other = initial;
other.flags = (1u << Player::ALIVE_FLAG_OFFSET) | (1u << Player::POISONED_FLAG_OFFSET);

Patch patch = PatchBuilder::FromDifference (Player::Reflect ().mapping, &other, &initial);
```

For performance reasons `Patch` system has one limitation: it works only with fields that occupy 8 or less bytes of
memory, so inplace strings and inplace blocks of memory are ignored. But not unique strings, inplace vectors and 
nested objects! Thanks to field projection technique inplace vectors and nested objects are analyzed as sets of
disconnected fields and therefore are freely processed by `Patch`es.

After creation `Patch`es can be easily applied to any object of target type:

```c++
patch.Apply (&object);
```

Just like `Mapping`s, `Patch`es can be moved, copied and stored anywhere you need.

### Implementation details

[Emergence](https://github.com/KonstantinTomashevich/Emergence) reflection system was implemented with memory usage
and cache coherency in mind. All internal data is stored as close to each other as possible:

- `Mapping` with fields is one continuous block of memory: it is reallocated during creation process to ensure that
  it uses just enough memory to store all the fields and not more. This allows field iteration to be as cache coherent
  as possible and also makes simultaneous access to multiple fields cache coherent too.
- Field visibility conditions are stored in special pool in order not to interfere with the field data. In the meantime,
  pool makes sure that conditions are laid down continuously unless page ends. Therefore condition access during 
  conditional allocation is also cache coherent.
- Like `Mapping`, `Patch` is also represented by one continuous block of memory, that is reallocated during creation 
  process. It makes patch application process as cache coherent as possible.

Also, both `Mapping` and `Patch` are actually managed as resource handles, therefore copying a `Mapping` or a `Patch`
does not result in actual data duplication: it only increases resource reference count.

### Reflection usage

[StandardLayoutMapping service](https://github.com/KonstantinTomashevich/Emergence/tree/a275a21/Service/StandardLayoutMapping) 
is a backbone of [Emergence](https://github.com/KonstantinTomashevich/Emergence) and used almost everywhere. There is
a quick summary of what it is used for:

- It powers the [RecordCollection service](https://github.com/KonstantinTomashevich/Emergence/tree/a275a21/Service/RecordCollection)
  by allowing it to use any record field for any index that needs it. Reflection also provides constructors and 
  destructors for the records.
- It powers the [Warehouse service](https://github.com/KonstantinTomashevich/Emergence/tree/a275a21/Service/Warehouse)
  by providing enough information for prepared query and object storages creation.
- It provides [Celerity library](https://github.com/KonstantinTomashevich/Emergence/tree/a275a21/Library/Public/Celerity)
  with information for query preparation, event management and pipeline validation.
- Patches are the backbone of [Celerity::Assembly library](https://github.com/KonstantinTomashevich/Emergence/tree/a275a21/Library/Public/Celerity/Extension/Assembly): 
  they are used to initialize freshly created objects with required data.

To summarize, [StandardLayoutMapping service](https://github.com/KonstantinTomashevich/Emergence/tree/a275a21/Service/StandardLayoutMapping)
is a very important part of [Emergence](https://github.com/KonstantinTomashevich/Emergence) project that powers lots
of other high level libraries. It was specially designed and optimized for optimal usage inside these libraries.

Hope you've enjoyed reading! If you have any suggestions, feel free to contact me through telegram or email.
