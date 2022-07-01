---
layout: post
title: 'Tutorial: Basics of link-time polymorphism'
categories: [Tutorials, Architecture]
tags: [Tutorials, C++, CMake]
---

Everyone knows what runtime and compile time polymorphisms are, but what about link-time polymorphism? There is much
less materials about this type of polymorphism and it seems almost forgotten. So I decided to write short tutorial 
about it!

### What link-time polimorphism is?

Runtime polymorphism operates on top of virtual methods, lambdas and function pointers. Compile time polymorphism
utilizes special template-based technique like SFINAE and CRTP to get things done. But what could we do during link
time? While assembling compiled object files into monolithic libraries and executables linker also resolves so-called
external references: declarations that are visible to compiled object but are defined outside of its scope. And that's
exactly where the secret of link-time polymorphism is hidden: we're selecting where linker searches for these 
definitions and by switching definitions we could achieve polymorphism. The most classic example is C standard library:
if you're using only standard functions, your program could be compiled on any system that supports C, but each time
different implementation library would be linked to your program. Let's discuss pros and cons of this method.

Pros:
- There is no runtime cost of link-time polymorphism: call cost is equal to regular function call and even inlining
  is possible through link-time optimizations.
- There is no additional compile time cost: we're not creating hordes of templates like in compile-time polymorphism,
  therefore compilation time is not higher that compilation time of regular code without polymorphism.
- It's the only method that can be used to hide platform-specific code.

Cons:
- You cannot change implementation while program is running because linking is already done. However, if dynamic
  linking is used, it is possible to change implementation between program executions by swapping shared library.
- You need buildsystem support: decision what libraries to link is done through build system, therefore you need
  to be ready to modify it.

As you can see, link-time polymorphism is not the silver bullet and it has its own restrictions. But, im my opinion,
it deserves to be used much more that it is used nowadays: in a lot of cases we don't really need to change 
implementation while program is running, but runtime polymorphism is still used and so we're paying excessive
performance price for it.

### Example problem

Let's imagine that our replication needs to serialize data in textual format in development mode for easier debugging
and in binary format in production mode for smaller file size. The most classic solution for such cases is runtime
polymorphism: using pure virtual interface and two implementations for production and development. It is working
solution, but is looks kind of clumpsy for me:

- We don't need to ever switch implementations during runtime, but we're paying runtime polymorphism costs.
- Unneded development logic will be compiled into production executable and vice versa. It's not critical, of course,
  but not elegant either.

Through runtime polymorphism works well enough here, I'm not satisified with this kind of approach, therefore
I've decided to write simple step-by-step tutorial how to do that with link-time polymorphism.

### Initial solution

Lets start from the simplest way of implementing link-time polymorphism: object file selection. It means that we decide
on buildsystem level which files belong to the target and by that we are selecting correct implementation for our 
target.

#### Interface

The best way to start it to draft our serialization interface. For the purpose of simplicity we would just serialize
strings and ints and append comments to development mode files. Of course, it not even near the real serializer case,
but it's better to have simple unrealistic example that difficult to understand but real one.

```c++
#pragma once

#include <cstdint>

// We are using PImpl approach to hide implementation data from user code.
// https://en.cppreference.com/w/cpp/language/pimpl
// It also allows to easily switch DLLs.
//
// The other approach is to use inplace fixed size array, which is better for this case,
// but makes  tutorial much less readable. Therefore, I've decided to sacrifice
// performance, which means nothing here, for readability. :)
struct SerializerImplementation;

class Serializer final
{
public:
    explicit Serializer (const char *_outputFileName) noexcept;

    Serializer (const Serializer &_other) = delete;

    Serializer (Serializer &&_other) noexcept;

    ~Serializer () noexcept;

    void WriteInt32 (int32_t _number) noexcept;

    void WriteAsciiString (const char *_string) noexcept;

    void WriteAsciiComment (const char *_string) noexcept;

private:
    // We cannot use std::unique_ptr here because SerializerImplementation is undefined.
    SerializerImplementation *implementation = nullptr;
};

``` 
{: file='Library/Serialization/Serializer.hpp'}

#### Implementations

Both our implementations will write data to a file through `std::ofstream`. So they will have same 
`SerializerImplementation`, move constructor and destructor. We don't want to duplicate this code, therefore
it makes sense to make this code common for both implementations.

```c++
#pragma once

#include <fstream>

#include <Serialization/Serializer.hpp>

struct SerializerImplementation
{
    std::ofstream output;
};
```
{: file='Library/Serialization/Common/SerializerPrivate.hpp'}

```c++
#include <Serialization/Common/SerializerPrivate.hpp>

Serializer::Serializer (Serializer &&_other) noexcept : implementation (_other.implementation)
{
    _other.implementation = nullptr;
}

Serializer::~Serializer () noexcept
{
    delete implementation;
}
```
{: file='Library/Serialization/Common/SerializerPrivate.cpp'}

Now lets focus on our sample implementations. `Development` implementation will save each element directly as text,
separating them using line ends:

```c++
#include <Serialization/Common/SerializerPrivate.hpp>
#include <Serialization/Serializer.hpp>

Serializer::Serializer (const char *_outputFileName) noexcept
    : implementation (new SerializerImplementation {std::ofstream {_outputFileName, std::ofstream::out}})
{
}

void Serializer::WriteInt32 (int32_t _number) noexcept
{
    implementation->output << _number << std::endl;
}

void Serializer::WriteAsciiString (const char *_string) noexcept
{
    implementation->output << _string << std::endl;
}

void Serializer::WriteAsciiComment (const char *_string) noexcept
{
    implementation->output << "# " << _string << std::endl;
}
```
{: file='Library/Serialization/Development/Serializer.cpp'}

For `Production` implementation we will directly use binary output:

```c++
#define _CRT_SECURE_NO_WARNINGS

#include <Serialization/Common/SerializerPrivate.hpp>
#include <Serialization/Serializer.hpp>

Serializer::Serializer (const char *_outputFileName) noexcept
    : implementation (
          new SerializerImplementation {std::ofstream {_outputFileName, std::ofstream::out | std::ofstream::binary}})
{
}

void Serializer::WriteInt32 (int32_t _number) noexcept
{
    // It is incorrect to save ints like this because it ignores endianness. But it is ok enough for our sample.
    implementation->output.write (reinterpret_cast<const char *> (&_number), sizeof (_number));
}

void Serializer::WriteAsciiString (const char *_string) noexcept
{
    // We are adding 1 to strlen in order to capture zero terminator.
    implementation->output.write (_string, strlen (_string) + 1u);
}

void Serializer::WriteAsciiComment (const char *_string) noexcept
{
    // There is no comments in production files.
}
```
{: file='Library/Serialization/Production/Serializer.cpp'}

#### User application

For this sample we will keep user application as small as possible, because we have no need of complex logic here:

```c++
#include <Serialization/Serializer.hpp>

int main ()
{
    Serializer serializer {"Test.out"};
    serializer.WriteAsciiComment ("This is test file comment line.");
    serializer.WriteAsciiString ("Hello, world!");
    serializer.WriteInt32 (1024);
    return 0;
}
```
{: file='App/Main.cpp'}

#### CMake script

Finally, we have arrived at our final and most important logic: CMake build system script. It is fairly simplistic
because we're using the simpliest approach of link-time polymorphism:

```cmake
cmake_minimum_required (VERSION 3.21)
project (LinkTimePolymorphismBasicsProject)

set (CMAKE_CXX_STANDARD 20)

# Build system option that is used for implementation selection.
option (DEVELOPMENT "Whether to use development serialization library." OFF)

# Declare core set of sources, that are used in any case.
set (SOURCES
        "App/Main.cpp"
        "Library/Serialization/Serializer.hpp"
        "Library/Serialization/Common/SerializerPrivate.cpp"
        "Library/Serialization/Common/SerializerPrivate.hpp")

# Depending on build system option append source file with implementation.
if (DEVELOPMENT)
    list (APPEND SOURCES "Library/Serialization/Development/Serializer.cpp")
else ()
    list (APPEND SOURCES "Library/Serialization/Production/Serializer.cpp")
endif ()

# Finally, create our application target.
add_executable (App ${SOURCES})
target_include_directories (App PRIVATE "Library/")
```
{: file='CMakeLists.txt'}

#### Conclusion

Despite being the most simple and straighforward way to use link-time polymorphism, direct file selection is the most
widely used too: it is the usual go-to solution for platform-independence layers implementations and other
layers, like graphics API independence layer. But it is not "poetic" enough, isn't it? Below we will discover more
scalable and more "poetic" approaches to link-time polymorphism.

### Make implementations separate libraries

...

### Use implementations as switchable dynamic libraries

...
