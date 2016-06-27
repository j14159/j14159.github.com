---
layout: post
title: "Introducing MLFE"
date: 2016-06-27 15:53:14 -0700
comments: true
categories: 
---
# What
[ML-Flavoured Erlang](https://github.com/j14159/mlfe) is a statically
typed language with basic parametric polymorphism (generics) for the
Erlang VM that I've been hacking away at in my spare time since
roughly the beginning of 2016.  I'm making the repository public now
for version 0.1.0 with the Apache 2.0 license as I think I've hit the
point of it being "not entirely useless".  Currently every type is
inferred as there are no type annotations and the error messages leave much
to be desired but it works.  Here's a simple example module:

    module example_module

    /* A function that adds 2 to an integer.  If you try to call this
       with a float you'll get a type error.
    */
    add2 x = x + 2

    // a basic ADT, just no-argument type constructors:
    type even_odd Even | Odd

    // Convert an integer to the basic ADT:
    even_or_odd x = match x % 2 with
        0 -> Even
      | _ -> Odd

It's written in Erlang using
[leex](http://erlang.org/doc/man/leex.html) and
[yecc](http://erlang.org/doc/man/yecc.html) for parsing and the `cerl`
module to build a Core Erlang AST for compilation.  I had to build the
docs for `cerl` directly as they don't seem to be generally hosted
anywhere.  If you're curious and want to explore, you can find them
in your Erlang distribution's compiler lib.  For OSX, Erlang 18.3's
compiler source installed by [Homebrew](http://brew.sh/) on my machine
lives in the following folder:

    /usr/local/Cellar/erlang/18.3/lib/erlang/lib/compiler-6.0.3

I used [edoc](http://erlang.org/doc/apps/edoc/chapter.html) to generate API
docs and keep them around locally for reference.

The [MLFE repository](https://github.com/j14159/mlfe) has some basic
getting started help in the `README` but the user experience is rough around the
edges to put it mildly.  Those of you familiar already with Erlang
will note the distinct (temporary) absence of binaries and maps as well.

# Why
I started working on this for two reasons:  I wanted something like
the various forms of ML ADTs to be available on the Erlang VM along with static
typing (I like the concision and the help), and I wanted to learn more
about how type systems are implemented.

Much of the ADT requirement can be satisfied by atoms and tuples in
Erlang proper but it always ends up feeling a bit cumbersome and while
dialyzer is great, it feels like I have to work pretty hard to
properly constrain contracts to what I really want.  None of this
stops me for reaching for Erlang when it's a good fit, it's just a
friction point I wanted to try to smooth out.

# How
Critical to this happening at all were the following resources:

- ["How the OCaml type checker works"](http://okmij.org/ftp/ML/generalization.html)
  by Oleg Kiselyov.
- github.com/tomprimozic's ["Grow Your Own Type System"]( https://github.com/tomprimozic/type-systems/tree/master/algorithm_w)
for some clarification on arrows and schema instantiation.
- Benjamin C. Pierce's
  [Types and Programming Languages](https://mitpress.mit.edu/books/types-and-programming-languages)
  which I still need to finish and re-read properly.
- the source to
[Lisp Flavoured Erlang](https://github.com/rvirding/lfe) for clearing
up some leex stuff and use of `cerl`.

# What's Next
For version 0.2.0 it's all mostly low-hanging fruit:

- binaries
- maps
- quoted atoms
- a test form/macro
- do notation/side effects

I'd like to write a basic rebar3 plugin for it as well to simplify
working with Erlang since so much functionality is going to depend on
mixing with it for the time being.  Longer term the type inferencer
needs to be substantially reworked and possibly even rewritten.
Specific goals there:

- logging of typing decisions tied to reference cells and binding
points (function and variable names)
- proper garbage collection of reference cells, probably abandoning
  processes for this and using something like
  [ETS](http://erlang.org/doc/man/ets.html)
- always generalizing type variables

The incomplete short-list of things I'd like to see in 1.0.0 (if I get
that far):

- records with structural pattern matching.  I want to be able to
match on a subset of fields.
- type annotations
- anonymous functions
- pattern matching in function definitions, e.g. `f (x, y) = x + y` to
destructure a tuple without an explicit `match ... with ...`
- an Emacs mode so I don't have to manually indent things
- something like `go fmt`.  My intention is to integrate comments with
  the AST partially to enable this.

# Contributing and Discussing
Contributions are welcome subject to the licence and code of conduct
as specified in the project repository.  I lurk in `#erlounge` on
freenode as `j14159` and I'm on Twitter with [the same username](https://twitter.com/j14159).
