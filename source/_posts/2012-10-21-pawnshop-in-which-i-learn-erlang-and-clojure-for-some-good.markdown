---
layout: post
title: "Learning Erlang + Clojure For Some Good"
date: 2012-10-21 17:22
comments: true
categories: [Erlang, Clojure, PawnShop]
---

## Background

Since pretty much everything I do at my [day job](http://twitter.com/hootsuite) involves Akka and Scala, Erlang's been on my radar for quite a while.  Using RabbitMQ in current products and considering Riak to solve some nasty storage issues has of course heightened my interest in it as well.  Thanks to [@ujm](http://twitter.com/ujm) and [@tavisrudd](http://twitter.com/tavisrudd) I've also been hearing a lot about Clojure for the past year or so and it has (not so) mysteriously turned up in a few testing tools at the aforementioned day job due to [@ujm](http://twitter.com/ujm)'s unbridled love for it.  As I've been trying to improve my own approach to and understanding of functional programming, it seemed both natural and frankly exciting to dive into something involving both.

## The Pitfall

This brings up a minor pet peeve of mine:  while evaluating something new and interesting, people often come up with trivial example projects and in the past, I've been possibly more guilty than most.  This is something I want very much to avoid now.  The problems I've seen in myself with using something _too_ trivial to evaluate and learn new things are:

* performance assumptions are never grounded in reality when your code doesn't do anything realistic.  A perfect example is testing concurrent websocket connection limits that simply echo input (I did this with Play! 2.0 and I've seen some naive numbers from the node.js crowd mentioned lately as well).
* simple projects rarely expose some of the strange corners of a language or platform.  Anyone who has extensively used Scala's implicits or made extensive use of futures and promises will understand this.
* most importantly overly trivial projects won't expose your __own__ weaknesses and misunderstandings early on.

## Some Goals

Given the above, there are a few specific challenges I'm looking to work on with this project in both Erlang and Clojure:

* unique ID generation without a central locking resource.  I want to be able to establish total ordering without any central coordination.  _Loose_ temporal ordering is a nice to have.
* horizontal scale, add and remove additional resources with no coordination required beyond some configuration.  As long as nodes can get in touch with the database, they should continue to function.

And I have a few goals specific to Clojure and Erlang:

* learn to use their build tools (Leiningen and Rebar)
* loosen my reliance on object orientation
* get more comfortable with dynamic typing (I tend to yell a lot about the supremacy of static typing).
* experiment with a heterogeneous solution.  I'll be building the same thing in both languages so I think some interesting comparisons could arise with regard to things like deployment, configuration and performance.

Naturally I'm going to make a few rules for myself, the first and most important being that I'm not going to hide my mistakes.  When I change my mind or screw up, all the details will be in the commit history.  Second, the whole thing will be open source so that anyone will be able to see my pace, changes and progress in github at any time.
Lastly, I'll welcome feedback and optimization suggestions(with credit given where credit is due) but won't accept pull requests or massive rewrites from anyone - this is mine to learn and mess up.

## The Actual Project

I'm going to build a simple URL shortener and image hosting service called [PawnShop](http://github.com/j14159/PawnShop) with the following basic features:

* exchange a URL for a short hash
* redirect short URLs to originals
* upload an image (png/jpeg) in return for a short hash
* redirect HTTP requests for a URL hash to the original
* display the matching submitted image for a given requested hash
* track click stats for each hash generated.  I'll store these at millisecond resolution.
* display hosted images inside a minimal template
* provide a REST API to retrieve click stats, likely report by one hour intervals at the smallest at first.
* basic authentication via email address and password

The nice-to-have features will be:

* click stat reports/graphs with D3.js
* pluggable authentication

The basic technical bits:

* This will be a single monolithic project, no interdependent services.
* It will be heterogeneous in that I want to be able to run the Erlang version side by side with the Clojure version in the same cluster, off the same database, behind the same load balancer and things should Just Work (TM).
* I'm going to use standard web frameworks, likely [Noir](http://webnoir.org) and either [Chicago Boss](http://chicagoboss.org) or [Nitrogen](http://nitrogenproject.com)
* The only data store I'll be using is Riak.

## Following Along

I'm going to aim for roughly a post-per-week here.  Blog posts related to PawnShop will essentially be a notebook/journal of my reasoning for certain decisions as well as both negative and positive experiences throughout the learning and development process.

[__GitHub__](http://github.com/j14159/PawnShop)

[__Trello__](https://trello.com/board/pawnshop/5084584abfe3e0931a00fdce)


## Riak and the Data Model

A while ago, [@jamesgolick](http://twitter.com/jamesgolick) told me I wasn't allowed to use Riak until I had read the famous Dynamo paper in its entirety.  He's right, and you should [read it too](http://s3.amazonaws.com/AllThingsDistributed/sosp/amazon-dynamo-sosp2007.pdf) if you haven't yet.

A URL shortener model seems a great fit for an eventually consistent Dynamo-style store like Riak.

* To shorten URLs and host pictures, you want high consistency for DB writes meaning that when a user submits new data, you want the write to complete with reasonable certainty.
* To retrieve URLs, low read consistency (small quorum) is OK.  Since we're not **mutating** URL and picture entries, we don't have to worry about an update that might not have hit replicas yet.
* To add and retrieve click stats, low read and write consistency is fine.  Missing one or two clicks out of thousands for a single report isn't the end of the world, we just want reasonable accuracy when the stats are requested and eventually high accuracy.  If it takes a few hours for this to happen, so be it.

## Assumptions/Dependencies

### statsd
Since I'm going to monitor the hell out of everything, I'll assume something statsd-compatible is available wherever this project runs.  These days I don't think this is really an option at all for any sort of sane environment.

## Next Steps

The next few posts will likely cover the following topics:

* decentralized ID/hash generation
* Riak model for shortened links and picture data
* Riak model for click stats

But those could change at any time.

Comments/questions always welcome on [Twitter](http://twitter.com/j14159)

