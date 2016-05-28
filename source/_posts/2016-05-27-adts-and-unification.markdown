---
layout: post
title: "ADTs and Unification"
date: 2016-05-27 18:44:16 -0700
comments: true
categories: compiler types
---
I've been tinkering with a statically-typed language for the Erlang VM for the
past few months, up until recently little more than an ML-shaped front for
Core Erlang with a type inferencer and incomplete support for basic Erlang
stuff.  Before fleshing out some of the required bits (like binaries), I'm
working through implementing simple abstract data types.  I'm aiming for
something roughly analogous to OCaml's variants or Haskell's GADTs.  Essentially
I want something like the following to work:

    type my_list 'x = Nil | Cons ('x, my_list 'x)

but also:

    type json_subset =  int
                      | string
                      | (string, json_subset)
                      | Array (list json_subset)

It's the latter that makes type unification a bit squirelly and this post is
mostly thinking through how I'm going to attempt it at first.  It's entirely
possible I don't fully understand the problem I'm trying to tackle yet but as
this entire project is an attempt to better understand type inference and
checking I'm pretty OK with that.

For clarity, the unification function gets called currently in the following
situations:

- for items being cons'd together into a list
- across all pattern match clauses to type the match (all patterns must unify
  with each other, all result types must unify as well)
- inside each clause to ensure the types forced by guard functions match the
types in the result expression
- inside each guard, sort of a special case of function application that I can
  likely generalize better.
- Erlang FFI returns.  These look like pattern matches but only the result
portion of each clause is checked.
- function application.

A cleaner example to describe the problem is based on the following types:

    type t = int | A int
    type u = int | float
    type v = int | float | (string, float)

Inferencing of basic matches already works, e.g.

    f x = match x with
        0 -> :zero
      | 1 -> :one
      | _ -> :more_than_one

will type to `{t_arrow, [{t_int}], t_atom}` meaning "a function from integers to
atoms".  What I'd like is for

    f x = match x with
        i, is_integer i -> :int
      | A _             -> :got_an_A

to type to `{t_arrow, [{t_adt, "t"}], t_atom}` but this won't be too hard
either.

The following is a little more difficult:

    f x = match x with
        i, is_integer i                 -> :int_but_could_be_one_of_three_types
      | f, is_float f                   -> :could_be_u_or_v
      | (s, f), is_string s, is_float f -> :definitely_v

The first two clauses (when unified) are shaped like a `u` while the third makes
it a `v`.  Unification works by trying to unify only two types at a time at
present so in a naive implementation the first and second clauses will unify to
`{t_arrow, [{t_adt, "u"}], t_atom}` and the third will cause a unification
failure because there is no type encapsulating `u` and `v`.  I think I've got
two options, aside from saying "reorder your clauses":

1.  Rework `unify` to take a list of expressions instead of two.
2.  Require a union type covering `v` and `u` in order to type

I think #1 is better ("just works") but I think I'm going to choose #2 as at
moment the list of problems to solve before opening up the source is a bit nasty,
including but not limited to:

- type errors are nothing more than `{error, {cannot_unify, Type1, Type2}}` and
  don't even include a line number.  "User-hostile" is an understatement.
- documentation and testing facilities are nonexistent.  I've got some
  particular directions I want to go with both (mostly selfish learning stuff)
  before I take the lid off.
- a truly scary number of dialyzer errors.  That my type-checking language
  doesn't type-check is a source of mild amusement for me at the moment.
- pretty much zero support for compiling anything but a list of files.
- some trivial-to-implement stuff like sending and receiving messages.
- haven't even begun to tackle strings-vs-lists, etc.

I'll probably make the repository public once at least the first three items of the above
list are done as right now those are the bits I'm most embarassed about.

Extra bits for the curious:

- implemented using [leex](http://erlang.org/doc/man/leex.html) and
  [yecc](http://erlang.org/doc/man/yecc.html) with the `cerl` and `compile`
  modules to create Core Erlang ASTs and generate `.beam` files respectively.
- has a basic FFI to Erlang, I haven't yet tried calling dializer so it requires
a set of clauses to type the result.
- type checker was based initially on
[How OCaml type checker works](http://okmij.org/ftp/ML/generalization.html).
- not much of a design behind the language at the moment.  Bits of syntax come
from Haskell, OCaml, and Elm at the moment.  It's grown as I've learned.
