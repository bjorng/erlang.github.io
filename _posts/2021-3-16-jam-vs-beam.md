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
are (they are usually on the stack).

JAM code is compact. The opcode (instruction number) for each
instruction fits in one byte. Thus, the instruction sequence above is
just 8 bytes (one byte for each instruction, and one additional byte
for the `42` following the `pushInt1` instruction).

[rpn]: https://en.wikipedia.org/wiki/Reverse_Polish_notation

#### A very brief introduction to BEAM

BEAM don't use the stack for temporary values. Instead, BEAM uses **X
registers** for passing arguments to function and for temporary
values. There are 1024 X registers `x0` through `x1023`.

Here is how the code for expression `A * 3 + B * 42` would look like for BEAM (using a
somewhat simplified notation compared to real BEAM code):

    gc_bif '*', x0, 3 => x0   % Multiply x0 (A) with 3, storing into x0
    gc_bif '*', x1, 42 => x1  % Multiply x1 (B) with 42, storing into x1
    gc_bif '+', x0, x1 => x0  % Add x0 and x1, storing into x0

Note that each instruction has one or more explicit source operands and a destination
operand indicating where the result should be stored.

So how is the code size of this instruction sequence compared to JAM? There are only
three instructions compared to the seven JAM instructions, but the instructions have
explicit operands so we should expect them to be less compact.

How less compact? BEAM instructions are word-based. The first word of
each instruction contains a pointer to the code to execute that
instruction. That means that the mininum size of an instruction is 8
bytes on a 64-bit CPU. Operands follows in the following words. Some
type of of operands can be packed (for example, X registers), but
others (such as an integer literal or an atom) need a whole word.

The size of the instruction sequence above is 11 words or 88 bytes.

To learn more about BEAM, see [A brief introduction to
BEAM][BEAM-primer], and to learn more about how BEAM loads and
executes instructions, see [Interpreter optimization][int-opt].

[BEAM-primer]: http://blog.erlang.org/a-brief-BEAM-primer
[int-opt]: http://blog.erlang.org/Interpreter-Optimizations

### Taking stock: instruction execution differences

Already after this brief introduction to JAM and BEAM we can see that BEAM
probably has an edge over JAM in terms of instruction execution because BEAM
usually executes fewer but more complex instructions in order to achieve the
same task. Indirect branches are usually very expensive on modern CPUs because
branch prediction is usually poor. Therefore, it is better to execute fewer
instruction.

Instruction dispatching is also slightly faster for BEAM, which uses a
[threaded code interpreter][threaded code] as opposed to [byte code interpreter][bytecode]
for JAM. While JAM fetches the opcode byte for an instruction and use the opcode as index
into array to get the pointer to the executable code, BEAM already contains a direct
pointer to th executable code.

To further reduce the number of instructions that will need to be executed, the BEAM loader
will combine common sequencs of instructions into a single instruction.

[threaded code]: https://en.wikipedia.org/wiki/Threaded_code
[bytecode]: https://en.wikipedia.org/wiki/Bytecode

### Matching and building

    tuple_match({a, _, _, X}) ->
        {ok,[X]}.

    try_me_else_fail
    alloc_1
    arg_0
    unpkTuple_4
    getAtom a
    pop
    pop
    storeVar_0    % X
    commit
    pushAtom ok
    pushNil
    pushVar_0     % X
    mkList
    mkTuple_2
    ret
