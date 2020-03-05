---
layout: post
title: Concepts for variables
series: Over-expressive Types
categories: devlog over-expressive-types
topics: generic concepts
date: 2016-05-09
last-modified-on: 2020-03-05
---

Let's approach the situation from the opposite angle. Rather than starting at concepts, let's start
at concept *use sites*. For instance using our fanciful, pared-down version of an iterator could
look like this:

```cpp
// initialized somehow
auto it = …;

++it; ++it; ++it;
*it;
++it; ++it;
*it;
++it;
*it;
```

Crucially we can immediately observe that we could have defined our initial variable as `auto& it =
…;` (initializer permitting) or `auto&& it = …;` and our program would not be substantially
different. This is in essence the difference between `the_simplest_function_by_val`,
`the_simplest_function_by_ref`, and `the_simplest_forwarding_function`.

In other words, when it comes to specifying close-fitting constraints to our program it turns out
that the *declared type* of `it` is not as important as how it is used as a *variable*. E.g. when
performing `++it` the subexpression `it` is an lvalue of type `concrete_iterator` regardless of
whether the declared type of the variable is `concrete_iterator`, `concrete_iterator&`, or
`concrete_iterator&&`. Let's then write a concept not for types, but for variables, and make use of
it:

```cpp
template<typename It>
using iterator_reference_t = typename std::iterator_traits<
    std::remove_cvref_t<It> // N.b.
>::reference;

template<typename It>
concept IteratorVar = requires {
    typename iterator_reference_t<It>;
} && requires(It it) {
    { *std::as_const(it) } -> iterator_reference_t<It>;
    ++it;
}

template<typename It>
concept IteratorTarget = IteratorVar<It&>;

template<IteratorVar It>
struct the_simplest_data {
    It as_a_data_member;
};

template<IteratorVar It>
iterator_reference_t<It> the_simplest_function_by_val(It as_a_value_param);

template<IteratorTarget It>
iterator_reference_t<It> the_simplest_function_by_ref(It& as_a_ref_param);

template<IteratorTarget It>
iterator_reference_t<It> the_simplest_forwarding_function(It&& as_a_forwarding_param);
```

We introduce `IteratorTarget` to avoid repeating the same mistake as before, making sure the
reference-taking functions check for `IteratorVar<It&>` and not `IteratorVar<It>` (the forwarding
function takes `It&&`, but once a reference variable is bound it does not matter any more whether
they are an lvalue or rvalue reference variable). But on second thought, we could dispense with it:

```cpp
template<IteratorVar It>
iterator_reference_t<It> the_simplest_function_by_ref(It& as_a_ref_param);

// similarly for the_simplest_forwarding function
```

As mentionted that's technically a bit imprecise. However as things are both `IteratorVar<It>` and
`IteratorVar<It&>` compute identical answer. Whether that should be or not (and thus whether
`IteratorTarget` is useful or not) leads us to a potentially interesting tangent, so I'll leave it
at that for today[^1].

  [^1]:
    The difference in constraints between:

    ```cpp
    void by_val(concrete val_param);
    void by_ref(concrete& ref_param);
    ```

    is that `by_val` requires `val_param` to be destructible, whereas `ref_param` already is. Hence
    the crux of the issue is that as currently written neither `IteratorType` nor `IteratorVar`
    checks for destructibility. And of course any answer as to whether a concept should check for
    destructibility or not would apply equally well regardless of whether that concept is written
    against types or against variables.

(There is an however an easy parallel with *cv*-qualifiers, one that is perhaps more practical.
Given our fanciful concept though it would require conceiving of immutable variables that are
iterators. In the interest of sparing the reader, this is left as an exercise.)

A final word and disclaimer
===========================

There is, all told, not much of a difference between the two alternatives. I really don't intend
this insight to be revolutionary and I don't consider concepts-for-variables to be an altogether
superior alternative to concepts-for-types either. It's mostly a game of moving and hiding some
complexity regarding reference- and cv-qualifiers around, but arguably the net sum of complexity is
the same.

The benefit, if any, is who's in charge of that complexity. I claim that in the case of
`IteratorType`, the complexity is pushed on the use sites whereas for `IteratorVar` it's pushed on
the concept writer. That matters to me since I consider that a useful concept is one that is defined
once, and used in multiple places.

As I am currently experimenting with concepts there are still select occasions that call for a
concept-for-types. Just like how it makes sense for `is_constructible_v` to operate on a declared
type for its first parameter, while it makes sense for `is_assignable_v` to operate on an
expression-as-type instead.

Above all, this is all new and experimental considering C++ concepts are barely fresh off the
presses. The time may come where I discard this approach entirely.
