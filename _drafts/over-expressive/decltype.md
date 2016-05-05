---
layout: post
title: A decltype reminder
series: Over-expressive Types
slug: over-expressive-decltype
categories: devlog
topics: generic concepts
date: 2016-05-02
---

Did you know there are two features behind the C++11 `decltype` keyword[^1]? On the one hand you can
use `decltype` to query the type of an *entity*, and on the other hand you can use it to query the
type of an *expression*. [This Stack Overflow
question](http://stackoverflow.com/questions/3097779/decltype-and-parentheses) showcases how those
two features can report seemingly different things. Throughout this series I'll make sure to always
explicitly specify which `decltype` variant I'm referring to.

  [^1]:
    C++14 also adds `decltype(auto)`, bringing the count to three--but that is something altogether
    different.

Let's now dig deeper. As it turns out, expression-`decltype` does a very important thing: it
'adjusts' the type it reports. Without going into too much technicalities, you can make the case
that all the following C++ expressions have type `int` (where `int i;` is in scope):

  - `0`
  - `i`
  - `*&i`
  - `std::move(i)`

And yet expression-`decltype` reports these varied answers:

| expression | `decltype( (expression) )`[^2]
| `0` | `int`
| `i` | `int&`
| `*&i` | `int&`
| `std::move(i)` | `int&&`

How expression-`decltype` adjusts its result depends on an additional property of the argument
expression, namely its value category. Hence the prvalue expression `0` merits no adjustment,
whereas the lvalue expressions `i` and `*&i` are translated to an `int&` result, and finally the
xvalue expression `std::move(i)` yields us `int&&`.

This adjustment that expression-`decltype` performs, and the situation we now find ourselves in,
should give pause for reflection. Consider this program fragment:

```cpp
int var1 = 0;
int& var2 = var1;

using type1a = decltype( var1 );
using type1b = decltype( (var1) );
using type2a = decltype( var2 );
using type2b = decltype( (var2) );

std::vector<int> var3;
```

Here, the types `int` and `int&` make several appearances each, in different guises. Some of those
repeated occurrences do not always mean the same thing, so we have to ask ourselves what could make
e.g. an `int&` here different from an `int&` there. As it turns out, a lot of it comes down to
convention and it's some of those conventions that I examine in this series.

Those wanting more after this brief overview may find comprehensive information on all of the above
[on Stack
Overflow](http://stackoverflow.com/questions/17241614/in-c-what-expressions-yield-a-reference-type-when-decltype-is-applied-to-them).

  [^2]:
    We're putting the expression within parentheses to make extra sure we're not accidentally
    using entity-`decltype`.
