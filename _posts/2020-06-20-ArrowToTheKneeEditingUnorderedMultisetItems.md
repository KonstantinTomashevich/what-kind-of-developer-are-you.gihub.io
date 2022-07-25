---
layout: post
title: 'Arrow to the Knee: Editing unordered_multiset items'
date: 2022-06-20 12:20:00 GMT+3
categories: [Notes, Arrow to the Knee]
tags: [Arrow to the Knee, C++, Containers]
---

Have you ever thought of editing `std::unordered_multiset` items while iterating over it? Does it even make sense?
I will tell you why and how I tried to do it and what happened to my code afterwards in this post.

### Getting into position

`std::unordered_multiset` iterator returns const references to items, and it works like that for a reason: developers
would've never expected that some psycho tried to edit items during iteration. And even if they did expect it,
there is no practical way to make this work, because there is no cheap way to teach `std::unordered_multiset`
to track changes in its records.

So, why did I need to edit items, after all? This case arose during
[Pegasus](https://github.com/KonstantinTomashevich/Emergence/tree/2896ee3f2e26ace474662dd07e85599750c6dcaa/Library/Private/Pegasus)
development: user requests write access to the record, so we need to provide it as fast as possible, and only after
user has done everything needed to be done do we check whether the key fields are different. Of course, we also have a
separate preallocated record that stores only the key fields to perform this check. So, when we find out that user
has changed the key fields, we need to somehow update `std::unordered_multiset` according to these changes.

There is one more case which is even more complicated at the first glance: user might just delete the record. In this
case we don't want to check whether the key fields have been changed, we want to just erase this record right away.

### Getting shot

The first thought that came to my mind was:

> Oh, we already have iterators, so we can just erase and then emplace and everything will be fine!
{: .prompt-danger }

Sounds reasonable, right? Even if the value is not the same as it was before, we still have a valid iterator,
so any operations on it should work fine. But it isn't the case with unordered containers.

The thing is, just having an iterator is not enough to perform the removal properly: unordered
container needs to calculate the hash once more and that is where things are getting fucked up, because the hash is
different now.

To check that, try running this snippet using any compiler and any STD implementation you like:

```c++
std::unordered_multiset <int> set;
set.emplace (1);
set.emplace (2);
set.emplace (3);

auto iterator = set.find (2);
const_cast <int&> (*iterator) = 4;
set.erase (iterator);

for (const int &number : set)
{
    std::cout << number << std::endl;
}
```

You can also try to execute it with numbers other than `4`: it would result in unexpected
~~(or sometimes normal but not really expected)~~ behavior.

### Healing the wounds

But in
[Pegasus](https://github.com/KonstantinTomashevich/Emergence/tree/2896ee3f2e26ace474662dd07e85599750c6dcaa/Library/Private/Pegasus)
everything works fine, so there is the solution for this problem, right? Yep, there is! A bit clunky and ad hoc though.

We can start from the fact that we need to somehow keep the hash the same after changing key fields. That means, that
we need to roll the key fields back to their initial values, right? This is the only way to think about that. And it is close
to how
[Pegasus](https://github.com/KonstantinTomashevich/Emergence/tree/2896ee3f2e26ace474662dd07e85599750c6dcaa/Library/Private/Pegasus)
actually deals with this problem: it stores a multiset of pointers and already has a preallocated record with initial
values for change checking, therefore we can just edit the iterator like that:

```c++
// Copy link to the record for further usage.
const void *record = *_position;

// Replace iterator content with backup to make a hash result valid.
const_cast<void const *&> (*_position) = storage->GetEditedRecordBackup ();

// Now we can safely erase record from our set.
records.erase (_position);

// And reinsert our changed record!
records.emplace_back (record);
```

Code above is a simplified version of what
[Pegasus](https://github.com/KonstantinTomashevich/Emergence/tree/2896ee3f2e26ace474662dd07e85599750c6dcaa/Library/Private/Pegasus)
does.

But what if you don't have a record backup? Well, if you know the index count during compile time, which is not the case for
[Pegasus](https://github.com/KonstantinTomashevich/Emergence/tree/2896ee3f2e26ace474662dd07e85599750c6dcaa/Library/Private/Pegasus),
there is another way to deal with this problem: storing the hash as an additional field and recalculating it only after
the record is erased from the map. Unfortunately, I've never tried it myself, so I can't tell you more about it.

Hope you've enjoyed reading about this tragic failings with `std::unordered_multiset`! :)
