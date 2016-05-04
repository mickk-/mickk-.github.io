---
layout: post
title: From types to expressions back again
series: Over-expressive Types
slug: over-expressive-back-again
categories: devlog
topics: generic concepts
date: 2016-05-02
---

Interestingly we can reverse the mapping introduced by expression-`decltype`, going instead from
types to expressions:

```cpp
template<typename ExprType> ExprType   declexpr();
// a variant, which can in fact be found in the Standard Library
template<typename ExprType> ExprType&& std::declval();
```

(An explanation of the choice that `std::declval` makes here can be found [on
StackOverflow](http://stackoverflow.com/q/25707441).)

Let's recap. Given an expression `expr` we can map it to a type thanks to expression-`decltype`:

```cpp
using exprtype = decltype( (expr) );
```

In turn we can convert this expression-as-type back again to the expression `declexpr<exprtype>()`.
However it should be noted that since `declexpr` is *declared* but not *defined* (since it is hard
to produce an actual value of arbitrary type out of thin air) we can only use it in limited ways.
For instance:

```cpp
using finalexprtype = decltype( ++declexpr<exprtype>() );
```

As signalled by the tell-tale use of expression-`decltype`, we have computed an expression-as-type
yet again. It is a type equivalent to `declype( ++expr )` where `expr` is our starting expression.
We have demonstrated is a way to carry around expression simulacra in the guise of types.

Why does this matter? Let's take a look at `std::is_constructible_v`:

```cpp
constexpr bool answer = std::is_constructible_v<foo, bar, baz>;
```

What we have here is a function from types to `bool`. Remarkably the types it accept take different
forms: the very first parameter is the *declared* type to an imaginary variable while the following
types are *expressions-as-types* for imaginary initializers. In other words, the previous snippet
asks the question (where `valid_definition` is a made-up pseudo-operator):

```cpp
constexpr bool answer = valid_definition {
    foo imaginary_variable(std::declval<bar>(), std::declval<baz>());
};
```

In general, how a type parameter will be interpreted is not apparent in the signature of a template.
It all comes down to convention, and what the template is intended to do. E.g.
`std::is_assignable_v` is very similar to `std::is_constructible_v` but unlike it even its first
parameter is an expression-as-type, as revealed by its semantics:

```cpp
constexpr bool answer = std::is_assignable_v<foo, bar>;
constexpr bool answer = valid_expression(
    std::declval<foo>() = std::declval<bar>()
);
```

Armed with what we know about expressions, types, and metafunctions we can now tackle an important
question: what can C++ concepts apply to, and what *should* C++ concepts apply to? While it's
obvious that concepts will be ranging over types most of the time, that's not precise enough.
