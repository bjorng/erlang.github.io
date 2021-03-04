---
layout: post
title: JAM versus BEAM
tags: BEAM JAM
author: Björn Gustavsson
---

In my [previous blog post][road_to_the_jit], I mentioned in passing
that JAM (Joe's Abstract Machine) was replaced with BEAM. A reader
asked for more details about what makes BEAM faster than JAM, and
couldn't the same tricks and optimizations used in BEAM have been used
to speed up JAM? This blog post will try to answer that question.

[road_to_the_jit]: http://blog.erlang.org/the-road-to-the-jit

### The fundamental difference between JAM and BEAM

    foo(A, B) ->
        A * 3 + B * 42.

    gc_bif '*', x0, 3 => x0
    gc_bif '*', x1, 42 => x1
    gc_bif '+', x0, x1 => x0

    arg_0
    pushInt_3
    arith_times
    arg_1
    pushInt1 42
    arith_times
    arith_plus


[A brief introduction to BEAM][BEAM-primer]

[BEAM-primer]: http://blog.erlang.org/a-brief-BEAM-primer
