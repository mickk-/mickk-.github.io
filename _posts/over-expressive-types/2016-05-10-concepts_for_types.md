---
layout: post
title: Concepts for types
series: Over-expressive Types
categories: devlog over-expressive-types
topics: generic concepts
date: 2016-05-10
last-modified-on: 2020-03-05
---

In this post and the next I will be using a deliberately dumbed-down iterator concept for the
purpose of illustration, one that only supports indirection (`*it`) and incrementing (`++it`) with
no pre- or post-condition. Each implementation will be gauged by attempting to write the simplest,
non-trivial function templates that can make use of that concept.

A direct approach is to consider that concepts operate on (concrete) types. That is to say, the
constraint `IteratorType<It>` is satisfied as long as `It` is a type that can be used as an
iterator. In my opinion this is how concepts have historically been informally specified, going back
to the days of [the SGI STL](http://www.martinbroadhurst.com/stl/) (e.g. the [STL Trivial Iterator
concept](http://www.martinbroadhurst.com/stl/trivial.html)). Consequently it appears it is also the
most common way of writing and talking about concepts today.

```cpp
template<typename It>
using iterator_reference_t = typename std::iterator_traits<It>::reference;

template<typename It>
concept IteratorType = requires {
    typename iterator_reference_t<It>;
} && requires(It it, It const cit) {
    { *cit } -> iterator_reference_t<It>;
    ++it;
};

template<IteratorType It>
iterator_reference_t<It> the_simplest_function_by_val(It it)
{
    ++it;
    return *it;
}

template<IteratorType It>
iterator_reference_t<It> the_simplest_function_by_ref(It& it)
{
    ++it;
    return *it;
}

template<IteratorType It>
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
    requires IteratorType<std::remove_reference_t<It>>
{ /* same as before */ }
```

I am however very fond of introduction syntax as it cuts down on a lot of noise and boilerplate, so
I find this very unsatisfactory.
