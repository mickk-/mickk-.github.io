---
layout: post
title: Concepts for variables
series: Over-expressive Types
slug: concepts-for-variables
categories: devlog
topics: generic concepts
date: 2016-05-04
---

Let's look at the situation backwards: can we write concepts in such a fashion that the following
use sites are easy to read and review:

```cpp
template<typename Type>
concept bool StrawConcept = …;

SimpleConstraint{Thing}
struct generic_data {
    Thing as_a_data_member;
};

SimpleConstraint{Thing}
auto generic_function_by_val(Thing as_a_value_param);

RefConstraint{Thing}
auto generic_function_by_ref(Thing& as_a_ref_param);

RefConstraint{Thing}
auto generic_forwarding_function(Thing&& as_a_forwarding_param);
```

By asking the question, I've already classified the constraints in two groups. We'll start with
`generic_data` and `generic_function_by_val` and their `SimpleConstraint`.
