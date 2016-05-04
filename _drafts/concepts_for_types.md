---
layout: post
title: Concepts for types
series: Over-expressive Types
slug: concepts-for-types
categories: devlog
topics: generic concepts
date: 2016-05-03
---

I will be using a deliberately dumbed-down iterator concept to illustrate, in various incarnations.
It will be used to write the simplest, non-trivial function templates that can make use of that
concept.

A direct approach is to consider that concepts operate on (concrete) types. That is to say,
the constraint `IteratorType<It>` is satisfied as long as `It` is a type that can be used as an
iterator. In my opinion this is how concepts have historically been informally specified, going back
to the days of [the SGI STL](https://www.sgi.com/tech/stl/). Consequently it appears it is also the
most common way of writing and talking about concepts today.

```cpp
template<typename It>
using iterator_reference_t = typename std::iterator_traits<It>::reference;

template<typename It>
concept bool IteratorType = requires {
    typename iterator_reference_t<It>;
} && requires(It it, It const cit) {
    { *cit } -> iterator_reference_t<It>;
    ++it;
};

IteratorType{It}
iterator_reference_t<It> the_simplest_function_by_val(It it)
{
    ++it;
    return *it;
}

IteratorType{It}
iterator_reference_t<It> the_simplest_function_by_ref(It& it)
{
    ++it;
    return *it;
}

IteratorType{It}
iterator_reference_t<It> the_simplest_forwarding_function(It&& it)
{
    ++it;
    return *it;
}
```

A potential pitfall of this way of doing things (and the main motivation behind this series of
posts) is that while the concepts themselves are easy to write and review, they are in my opinion
harder to *use* than they could be. There is in fact a bug in `the_simplest_forwarding_function`
above, as demonstrated by the following:

```cpp
// an actual iterator from somewhere
auto it = ...;
the_simplest_forwarding_function(it);
```

The bug here is that while the declared type of `it` is an actual iterator (let's call it
`concrete_iterator`), in the function call the constraints are instantiated to `concrete_iterator&`.
The way we wrote the concept, such a reference type fails to be a model because
`std::iterator_traits<concrete_iterator&>` is not a valid type.

Since the body of the function `{ ++it; return *it }` would otherwise be valid when instantiated
with `concrete_iterator&`, this means that `the_simplest_forwarding_function` is over-constrained. A
possible fix is to ditch the introduction syntax in favour of the more explicit:

```cpp
template<typename It>
iterator_reference_t<std::remove_reference_t<It>> the_simplest_forwarding_function(It&& it)
    requires Iterator<std::remove_reference_t<It>>
{ /* same as before */ }
```

This is what I meant when I said concepts for types are harder to use: because the concept itself
focuses on concrete types, the relationship between the actual types in use in the program (i.e. the
function template parameter `It`) and the concrete types of interest 'leaks' into the usage sites. I
consider it problematic that it is the writer and users of `the_simplest_forwarding_function` that
have to pay attention to this relationship since in a given program a concept is typically written
once but used in multiple places.
