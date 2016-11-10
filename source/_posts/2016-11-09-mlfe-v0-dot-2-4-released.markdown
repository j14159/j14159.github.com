---
layout: post
title: "MLFE v0.2.4 Released"
date: 2016-11-09 16:07:32 -0800
comments: true
categories: 
---
[MLFE version 0.2.4](https://github.com/j14159/mlfe/tree/v0.2.4) is now available and includes a few significant changes:

- fixes for type aliases and their use in union types and type constructors thanks to [Daniel Abrahamsson](https://github.com/danabr)'s work.
- early support for records with row polymorphism.
- some fixes for how maps are typed.

## Type Aliases
Up until [Daniel](https://github.com/danabr)'s fixes, the following wouldn't type correctly:

    module circles
     
    export new/1, area/1
     
    type radius = float
     
    type circle = Circle radius
     
    new r =
      Circle r
     
    area c =
      let pi = 3.14159 in
      match c with
        Circle r -> pi *. (r *. r)

The issue was that the typer wasn't looking up what `radius`'s members were when it tried to type a `circle` and further, type variables used in nested types weren't being handled correctly either.  Big thanks to [Daniel](https://github.com/danabr) for his work on this.

## Records
Basic records with row polymorphism are now supported in MLFE (not compatible with Erlang's records):

    module records_with_x
     
    export make_xy/2, make_xyz/3, get_x/1, get_x_and_the_record/1
     
    make_xy x y = {x=x, y=y}
     
    make_xyz x y z = {x=x, y=y, z=z}
     
    get_x rec =
      match rec with
        {x=x} -> x
     
    {- No matter what type of record you pass in, the record in the
       returned tuple will always remember the full set of its fields
       in the return type.  E.g. if you pass in {x=1, y=2, z=3}, the
       return type won't forget about y and z, the full type returned
       by this function will be (if you pass that exact record):
     
       (int, {x: int, y: int, z: int})
     
    -}
    get_x_and_the_record rec =
      match rec with
        {x=x} -> (x, rec)
        
Worth noting that the above module of course constructs polymorphic records.  You could call `make_xy "foo" "bar"` and `make_xy 1 2.0` and the two would be distinct types:

    {x="foo", y="bar"}  -- {x: string, y: string}
    {x=1, y=2.0}        -- {x: int, y: float}

My understanding of row polymorphism so far has been driven by [a presentation by Neel Krishnaswami](https://www.cs.cmu.edu/~neelk/rows.pdf) as well as skimming a few papers and some of [Elm](http://elm-lang.org/)'s history.

Currently missing:

- adding fields to a record or updating/replacing them
- accessing individual record fields

I'm not currently planning on adding the removal of record fields as it seems like it could get problematic and/or confusing quickly although it poses no particular technical challenges as far as I can tell.

To add fields to a record I'd like the following to work:

    -- This takes {x: 'a} and returns ('a, {x: 'a, hello: string})
    get_x_and_add_hello rec =
      match rec with
        {x=x} -> (x, {rec | hello="world!"})

Accessing individual members should look pretty simple, e.g.

    -- should type to (int, string):
    let tuple = get_x_and_add_hello {x=1} in
      match tuple with
        (x, rec) -> (rec.x, rec.hello)

### For the Curious
MLFE's typer does use row variables to capture the extra fields in a record being passed around.  The row variable is constructed and added to any record the [first time it's typed](https://github.com/j14159/mlfe/blob/v0.2.4/src/mlfe_typer.erl#L1431) and the function [`unify_records/4` in mlfe_typer.erl](https://github.com/j14159/mlfe/blob/v0.2.4/src/mlfe_typer.erl#L762) is probably also of interest.

Records are currently compiled to maps simply because it makes implementation right now far easier than as tuples.  As such we need a way to distinguish between records and maps in pattern matches so the code generation adds a key `'__struct__'` to both with the key either `'map'` or `'record'` to distinguish between the two.  Compiled pattern matches of course leverage this as well.

## Contributing and Discussing
As always contributions are welcome subject to the licence and code of conduct as specified in the project's repository.  Please join the IRC channel `#mlfe` on [freenode](http://freenode.net/) for help or general discussion (governed by the same code of conduct as the project).  I'm on IRC as `j14159` and Twitter with [the same username](https://twitter.com/j14159).
