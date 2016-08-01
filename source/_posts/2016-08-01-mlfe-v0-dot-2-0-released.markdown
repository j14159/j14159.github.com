---
layout: post
title: "MLFE v0.2.0 Released"
date: 2016-08-01 12:59:02 -0700
comments: true
categories: 
---
[ML-flavoured Erlang](https://github.com/j14159/mlfe) v0.2.0 is now ready for use.  This version includes a couple of major missing pieces from the first v0.1.0 release including:

- support for maps, both constructing literal maps and pattern matching
- binary support
- UTF-8 binary strings

This release also includes some very early support for writing unit tests inside the modules they're concerned with and running them via [EUnit](http://erlang.org/doc/apps/eunit/chapter.html) and [rebar3](http://www.rebar3.org/) with the help of [Tristan Sloughter's plugin](https://github.com/tsloughter/rebar_prv_mlfe).  If you're using the plugin, you'll be able to run tests defined in `.mlfe` files with `rebar3 eunit`.

Here's the example module from the new [tour of the language](https://github.com/j14159/mlfe/blob/master/Tour.md):

    module my_module
     
    // one function that takes a single argument will be publicly accessible:
    export double/1
     
    /* Our double function defines an "add" function inside of itself.
       Comments for now are C-style.
     */
    double x =
      let add a b = a + b in
      add x x
     
    test "doubling 2 is 4" = some_test_checker (double 2) 4
     
    // Basic ADT with type constructors:
    type even_or_odd = Even int | Odd int
     
    // A function from integers to our ADT:
    even_or_odd x =
      let rem = x % 2 in
      match rem with
          0 -> Even x
        | _ -> Odd x 

The full changelog is available [here](https://github.com/j14159/mlfe/blob/master/ChangeLog.org) and there's a new [language tour document](https://github.com/j14159/mlfe/blob/master/Tour.md) available that explains the use of the language in a little more detail for anyone new to it.

# What's Next
For version 0.3.0 my main goals are:

- probable project rename
- type annotations/ascriptions
- records.  I'd like some basic row polymorphism here (I believe this fits with records in OCaml and Elm) but need to do some more reading and research.  In all likelihood records will get syntax similar to OCaml's and be compiled as Erlang maps to start.

Nice-to-haves include:

- quoted atoms
- the `|>` "pipe" operator
- do notation/side effects

# Contributing and Discussing
Contributions are welcome subject to the licence and code of conduct as specified in the project repository.  Please join the IRC channel `#mlfe` on [freenode](http://freenode.net/) for help or general discussion (governed by the same code of conduct as the project).  I'm on IRC as `j14159` and Twitter with [the same username](https://twitter.com/j14159).
