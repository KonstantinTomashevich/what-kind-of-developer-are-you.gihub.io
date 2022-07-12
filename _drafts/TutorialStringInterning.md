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
we're seeing `unitType = "Knight"`, but it is much harder if we're seeing `unitType = 42`. Therefore strings are
the best variant when we need to refer to a well-known object like config entry. But it also means that such
string will be copied lots of times! What if every unit has `std::string unitType` and there is 100 units? We
would waste `99 * X` bytes where `X` is average config entry id length. And what if there is more units?
Not looking good, eh?

Checking string equality is much less appealing than checking number equality -- it's `O(string length)` instead
of `O(1)` after all. The same applies to string hashing -- it is also `O(string length)`. So if we are checking
string equality or hashing strings a lot, we might end up with a bottleneck: string iteration would consume 
much more time than actual algorithm. But there should be a way to avoid this problem, right?

Now it is time to unravel the mystery of string interning: all interned strings are immutable and are stored in a 
special storage that contains not more than 1 instance of string. So interned strings are never duplicated and 
therefore don't waste memory! But that's not all: if string is never duplicated we're free to use pointer comparison 
instead of string comparison and by that comparing strings in `O(1)`. The same goes for hashing: why not just use 
unique string pointer as hash?

### Simplistic implementation

### About unused interned strings

### Improoving implementation
