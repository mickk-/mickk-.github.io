---
layout: post
title: Self-contained Ranges Post-mortem
categories: devlog
topics: ranges
date: 2015-01-01
---

In the first approach (of a second tentative), ranges are entirely
self-contained. That is to say, a range stands for whatever
state is necessary to represent one step of iteration. Two issues arise from
that:

1) Random-access ranges are a very bizarre fit, in that an offset that can be used
    to access an element arguably also counts as a part of some iteration state. E.g.:

    {% highlight cpp %}
    auto r = /* ... */;
    for(int offset = 0; valid(offset, r); ++offset) {
        // offset is separate from r
        consume(at(r, offset));
    }
    {% endhighlight %}

1) Saveable double-ended ranges (to reuse D terminology), i.e. double-ended
    ranges that can be copied, are curiously similar to pairs of bidirectional
    iterators. This subtly leads to the following: suppose we operate over text
    split over the two colons it contains (e.g. `"abc:def:123"`):

    {% highlight cpp %}
    // Bidi iterator version
    void iterators(Bidi const start, Bidi const stop)
    {
        // prepare
        auto first_colon  = std::find(start, stop, ':');
        auto second_colon = std::find(first_colon, stop, ':');

        // consume
        process_first_part(start, first_colon);
        process_second_part(first_colon, second_colon);
        process_third_part(second_colon, stop);
    }

    // Saveable double-ended range version
    void range(Bidi range)
    {
        // prepare
        auto not_a_colon = [](auto const& c) { return c != ':'; };
        auto first_split = range::span(not_a_colon, std::move(range));
        auto& first_part = first_split.first;
        auto& rest       = first_split.second;
        auto second_split = range::span(not_a_colon, std::move(rest));
        auto& second_part = second_split.first;
        auto& third_part  = second_split.second;

        // consume
        process_first_part(first_part);
        process_second_part(second_part);
        process_third_part(third_part);
    }
    {% endhighlight %}

    While the two versions look to be equivalent, the iterators are more 'efficient' in that e.g.
    `first_colon` is reused and shared between the two subranges `[start, first_colon)` and
    `[first_colon, second_colon)`, and similarly for `second_colon`. For an N-partition of a
    bidirectional range, we need N+1 iterators with 0 overlap of iterators; or N subranges
    with N-1 overlap between them.

    Moreover, the apparent equivalence breaks down if the truly bidirectional nature
    of the iterators is exploited. E.g. performing `--first_colon` (assuming valid preconditions)
    re-partitions the entire range in three subranges still. But it is not possible to unpop
    `second_part` to 'rewind' back an element---and pushing back an element into it would be even more
    nonsensical, since ranges represent iteration and not necessarily e.g. sequences in
    memory, so there might not even be a place to push back into.

    This problem is not just a matter of taste. Writing a binary search algorithms for those
    not-quite-bidirectional ranges is much more unnatural than it is with iterators. For a
    double-ended range, a possible implementation may perform up to two scans per iteration:
    first to peek the middle element, then a second time to prepare the appropriate subrange.

        searching for 6, length(r) == 11

        iteration 0, length(r) == 11:
            r = [ - - - - - [4] x - - 6 - ]
            read midpoint   {first scan}
            4 < 6
            advance r to x  {second scan}

        iteration 1, length(r) == 5:
            r = [ x - [-] @ - ]
            …

    It feels like it would be wiser to split the range down the middle no matter what in one go,
    but it's actually interesting to contemplate how that could be achieved, what type the
    resulting subranges would have, and how the next iteration would take place (polytypic recursion?).

    An iterator version however could scan down the middle, read it, and adjust itself by one place if
    necessary.

---

So while forward and double-ended ranges are very satisfactory (at least in the non-saveable case, for
double-ended ranges), it is disappointing that random-access ranges don't really fit into our story.
If we go back to the problem of an N-partition of a bidirectional range, we can find a hint of where
our abstraction might be breaking down:

> For an N-partition of a bidirectional range, we need N+1 iterators with 0 overlap of iterators;
> or N subranges with N-1 overlap between them.

Interestingly, if we look deeper into it, that '0 overlap of iterators' statement turns out to be
treacherous. Indeed, one of the arguments as to why we turned to self-contained ranges in the first
place was that a pair of e.g. `filter_iterator`s had the unfortunate property that it duplicates the
filtering functor, one for each end of the range. So for an N-partition, we
now have N+1 functors, even though we really only need the one. And since our N-partition of subranges
has N-1 such functors, we're not really coming out ahead here either with self-contained ranges!

Thus begins the case for position-based ranges, as they are called.
