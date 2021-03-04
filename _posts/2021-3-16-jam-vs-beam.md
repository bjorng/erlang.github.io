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
a stack machine, the expression is rewritten to [reverse Polish notation][rpn]
like this:

    A 3 *  B 42 *  +

That is, the operands come before the operator. Each operator pops its operand
off the stack, computes the result, and pushes the result back to the stack.

Here is how the code for expression would look like for JAM:

    arg_0          % Push the value of function argument 0 (A) to the stack
    pushInt_3      % Push the integer 3 to the stack
    arith_times    % Pop two elements, multiply them, and push the result

    arg_1          % Push value of function argument 1 (B) to the stack
    pushInt1 42    % Push the integer 42 to the stack
    arith_times    % Pop two elements, multiply them, and push the result

    arith_plus     % Pop two elements and push their sum

The `pushInt1` instruction pushes an unsigned integer that will fit in
one byte. As an optimization, there are also the instructions `pushInt_0`
through `pushInt_15` that will push the integers 0 through 15, respectively.
Similarly, there `arg_0` through `arg_15` to push the first 16 function arguments,
and `argN` to push any function agument.

The advantage of a stack machine is that code generated for it is
fairly compact, since there is no need to indicate where the operands
are (they are usually on the stack). The instruction sequence above is
just 8 bytes (one byte for each instruction, and one additional byte
for the `42` following the `pushInt1` instruction).

[rpn]: https://en.wikipedia.org/wiki/Reverse_Polish_notation

#### A very brief introduction to BEAM



Here is how the code for expression would look like for BEAM:

    gc_bif '*', x0, 3 => x0   % Multiply x0 with 3, storing into x0
    gc_bif '*', x1, 42 => x1  % Multiply x1 with 42, storing into x1
    gc_bif '+', x0, x1 => x0  % Add x0 and x1, storing into x0


[A brief introduction to BEAM][BEAM-primer]

[BEAM-primer]: http://blog.erlang.org/a-brief-BEAM-primer
