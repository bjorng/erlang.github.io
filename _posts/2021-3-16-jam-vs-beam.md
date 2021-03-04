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

The most fundamental difference is that JAM is a [stack machine] and
BEAM is a [register machine]. Let's examine each machine in turn to
see what that means.

[stack machine]: https://en.wikipedia.org/wiki/Stack_machine
[register machine]: https://en.wikipedia.org/wiki/Register_machine

#### A very brief introduction to JAM

Consider this function:

    foo(A, B) ->
        A * 3 + B * 42.

To evaluate the expression in the function body of `foo/2` using
a stack machine, the expression is rewritten to [rpn][reverse Polish notation]
like this:

    A 3 *  B 42 *  +

That is, the operands come before the operator. Each operator pops its operand
off the stack, computes the result, and pushes the result back to the stack.

    arg_0          % Push the value of function argument 0 (A) to the stack
    pushInt_3      % Push the integer 3 to the stack
    arith_times    % Pop two elements, multiply them, and push the result

    arg_1          % Push value of function argument 1 (B) to the stack
    pushInt1 42    % Push the integer 42 to the stack
    arith_times    % Pop two elements, multiply them, and push the result

    arith_plus     % Pop two elements and push their sum



[rpn]: https://en.wikipedia.org/wiki/Reverse_Polish_notation


    gc_bif '*', x0, 3 => x0
    gc_bif '*', x1, 42 => x1
    gc_bif '+', x0, x1 => x0


[A brief introduction to BEAM][BEAM-primer]

[BEAM-primer]: http://blog.erlang.org/a-brief-BEAM-primer
