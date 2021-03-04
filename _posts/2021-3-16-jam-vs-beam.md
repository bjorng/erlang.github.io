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

### Matching and construction

To see how matching and construction of terms work in JAM and BEAM,
let's look at this function:

    tuple_match({a, _, _, X}) ->
        {ok,[X]}.

The compiler will generate the following code for JAM:

    %% Set up a default fail handler that will raise a `function_clause`
    %% exception if matching fails.
    try_me_else_fail

    %% Allocate a stack frame for this clause with room for a
    %% single variable.
    alloc_1

    %% Push the first function argument to the stack.
    arg_0

    %% Pop the top element from the stack and test whether it is
    %% a tuple of size 4. If its, push its elements to the stack.
    %% Otherwise, jump to the fail handler.
    unpkTuple_4

    %% Pop the top element from the stack (the first element of
    %% the tuple). Verify that it is the atom 'a'; if not, the
    %% match fails.
    getAtom 'a'

    %% Pop and discard the second and third elements of the tuple.
    pop
    pop

    %% Pop the stack and store the variable in variable slot 0.
    storeVar_0    % X

    %% Finish matching. The fail handler for failed matching will now
    %% be set to raise a `badmatch` exception if matching fails.
    commit

    %% Push the atom 'ok' (first element of tuple to be built).
    pushAtom 'ok'

    %% Push the empty list ([]) and the value of X.
    pushNil
    pushVar_0     % X

    %% Pop two elements and construct a list cell. Push the reference
    %% to the list cell to the stack. This instruction will do a
    %% garbage collection if there is no sufficient room on the heap
    %% for a list cell.
    mkList

    %% Construct a tuple of size 2: {ok, [X]}. This instruction
    %% will do a garbage collection if there is not sufficient room
    %% on the heap for a tuple of size 2.
    mkTuple_2

    %% Return with created tuple as the top element of the stack.
    ret

Here is the BEAM code:

    %% Ensure that x0 is a tuple of size 4. Jump to the label
    %% FailLabel if not.
    is_tagged_tuple FailLabel, x0, 4, 'a'

    %% Ensure that there 5 words available on the the heap for
    %% term building. Otherwise do a garbage collection.
    test_heap 5, 1

    %% Retrieve the fourth element from the tuple in x0.
    get_tuple_element x0, 3 => x0

    %% Create the list [X]. The previous test_heap instruction
    %% has ensured that there is sufficient room left on the heap.
    put_list x0 nil => x0

    %% Build the tuple {ok, [X]}.
    put_tuple2 'ok', x0 => x0

    %% Return from the function (the return value is in x0).
    ret
