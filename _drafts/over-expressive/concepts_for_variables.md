---
layout: post
title: Concepts for variables
series: Over-expressive Types
slug: concepts-for-variables
categories: devlog
topics: generic concepts
date: 2016-05-05
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
// not provided by the Standard Library
template<typename Type>
using unqualified_t = std::remove_cv_t<std::remove_reference_t<Type>>;

template<typename It>
using iterator_reference_t = typename std::iterator_traits<unqualified_t<It>>::reference;

template<typename It>
concept bool IteratorVar = requires {
    typename iterator_reference_t<It>;
} && requires(It it) {
    { *std::as_const(it) } -> iterator_reference_t<It>;
    ++it;
}

template<typename It>
concept bool IteratorTarget = IteratorVar<It&>;

IteratorVar{It}
struct the_simplest_data {
    It as_a_data_member;
};

IteratorVar{It}
auto the_simplest_function_by_val(It as_a_value_param);

IteratorTarget{It}
auto the_simplest_function_by_ref(It& as_a_ref_param);

IteratorTarget{It}
auto the_simplest_forwarding_function(It&& as_a_forwarding_param);
```

In fact, we could entirely dispense with `IteratorTarget`:

```cpp
IteratorVar{It}
auto the_simplest_function_by_ref(It& as_a_ref_param);

// similarly for the_simplest_forwarding function
```

That's technically a bit imprecise as the actual variable of interest has type `It&` whereas the
constraint is performed against a variable of type `It`. However as things are though both
`IteratorVar<It>` and `IteratorVar<It&>` compute identical answer. Whether that should be or not
(and thus whether `IteratorTarget` is useful or not) leads us to a potentially interesting tangent,
so I'll leave it at that for today[^1].

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
this insight to be revolutionary and I don't consider concepts-for-variables to be a superior
alternative to concepts-for-types either.

As I am currently experimenting with concepts there are select occasions that still call for a
concept-for-types. Just like how it makes sense for `is_constructible_v` to operate on a declared
type for its first parameter, while it makes sense for `is_assignable_v` to operate on an
expression-as-type instead.

Above all, this is all new and experimental considering C++ concepts are barely fresh off the
presses. The time may come where I discard this approach entirely.
