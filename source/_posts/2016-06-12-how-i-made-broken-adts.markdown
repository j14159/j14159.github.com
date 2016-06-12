---
layout: post
title: "How I Made Broken ADTs"
date: 2016-06-12 11:08:52 -0700
comments: true
categories: 
---
I've made some progress on ADTs in my toy BEAM language and have
discovered an issue that initially turned up with recursive types but
would be exhibited by any type that references another user-defined
ADT.  Spoiler:  the answer is to have a set of bindings from type
names to their instantiated types and use these by deep-copying as
with functions (h/t
[Pierce's TAPL](https://mitpress.mit.edu/books/types-and-programming-languages)
for the bit about making copies).
Briefly, the following works: 

    type my_list 'x = Nil | Cons ('x, my_list 'x)

    f x = match x with
        Nil                         -> :nil
      | Cons (i, Nil), is_integer i -> :length_one
      | Cons (i, _)                 -> :more_than_one_item

While this does not:

    type recursive_tuples = int
                          | float
                          | (string, recursive_tuples)

    f x = match x with
        i, is_integer i             -> :int
      | f, is_float f               -> :float
      | (key, value)                -> :pair
      | (key, (another_key, value)) -> :nested_pair

The nested tuple `(another_key, value)` fails to unify with the type
`recursive_tuples`.  This is because the AST node for the type definition
gets instantiated to this record:

    #adt{name="recursive_tuples"
         vars=[],
         members=[t_int,
                  t_float,
                  {t_tuple, [t_string,
                             #adt{name="recursive_tuples",
                                  vars=[],
                                  members=[]}]}]}

The first example works because the typer maintains a proplist of type
constructor to AST type nodes so any reference to a type constructor
will be able to find a type to instantiate.  The second example fails
to unify because once the typer sees the tuple, it uses the _tuple
member's_ enclosed members to try to type the nested tuple.  Note that
the defined ADT specifies its members while the reference to the ADT
later does not - there's nothing to unify with as it appears to be an
empty type.

Referring to a set of bindings for the types available to the module
should solve this, along with deep copying for multiple instances.
I'm tempted to just use generalization and instantiation but it may
ultimately be simpler to copy and make new reference cells.
