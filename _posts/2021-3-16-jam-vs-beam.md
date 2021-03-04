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
to speed up JAM?

[road_to_the_jit]: http://blog.erlang.org/the-road-to-the-jit

### The Prolog interpreter

The first version of Erlang was implemented in Prolog in 1986. That
version of Erlang was too slow for creating real applications, but it
was useful for finding out which features of the language were useful
and which were not. New language features could be added or deleted
in a matter of hours or days.

It soon became clear that Erlang needed to be at least 40 times faster
to be useful in real projects.

### JAM (Joe's Abstract Machine)

In 1989 JAM (Joe's Abstract Machine) was first
implemented. [Mike Williams][mike] wrote the runtime system
in C, [Joe Armstrong][joe] wrote the compiler, and
[Robert Virding][robert] wrote the libraries.

[mike]: http://www.erlang-factory.com/conference/ErlangUserConference2013/speakers/MikeWilliams
[joe]: https://github.com/joearms
[robert]: https://github.com/rvirding

JAM was 70 times faster than the Prolog interpreter, but it turned out
that this still wasn't fast enough.

### TEAM (Turbo Erlang Abstract Machine)

Bogumil ("Bogdan") Hausman created TEAM (Turbo Erlang Abstract
Machine). It compiled the Erlang code to C code, which was then
compiled to native code using GCC.

It was significantly faster than JAM for small projects.
Unfortunately, compilation was very slow, and the code size of the
compiled code was too big to make it useful for large projects.

### BEAM (Bogdan's Erlang Abstract Machine)

Bogumil Hausman's next machine was called BEAM (Bogdan's Erlang Abstract
Machine). It was a hybrid machine that could execute both native code
(translated via C) and [threaded code] with an [interpreter]. That
allowed customers to compile their time-critical modules to native code
and all other modules to threaded BEAM code. The threaded BEAM in
itself was faster than JAM code.

[threaded code]: https://en.wikipedia.org/wiki/Threaded_code
[interpreter]: https://en.wikipedia.org/wiki/Interpreter_(computing)

### Lessons Learned from BEAM/C

The modern BEAM only has the interpreter. The ability of BEAM to generate
C code was dropped in OTP R4. Why?

C is not a suitable target language for an Erlang compiler. The main
reason is that an Erlang function can't simply be translated to a C
function because of Erlang's process model. Each Erlang process must
have its own stack and that stack cannot be automatically managed by
the C compiler.

BEAM/C generated a single C function for each Erlang module. Local
calls within the module were made by explicitly pushing the return
address to the Erlang stack followed by a `goto` to the label of the
called function.  (Strictly speaking, the calling function stores the
return address to BEAM register and the called function pushes that
register to the stack.)

Calls to other modules were done similarly by using the GCC extension
that makes it possible to [take the address of a label][gcc_labels]
and later jumping to it.  Thus an external call was made by pushing
the return address to the stack followed by a `goto` to the address of
a label in another C function.

[gcc_labels]: https://gcc.gnu.org/onlinedocs/gcc/Labels-as-Values.html

Isn't that undefined behavior?

Yes, it is undefined behavior even in GCC. It happened to work with
GCC on Sparc, but not on GCC for X86. A further complication was the
embedded systems having ANSI-C compilers without any GCC extensions.

Because of that, we had to maintain three distinct flavors of BEAM/C
to handle different C compilers and platforms. I don't remember any
benchmarks from that time, but it is unlikely that BEAM/C was faster
than interpreted BEAM on any other platform than Solaris on Sparc.

In the end, we removed BEAM/C and optimized the interpreted BEAM so that
it could beat BEAM/C in speed.
