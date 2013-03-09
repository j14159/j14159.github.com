---
layout: post
title: "eSpacewar Open Sourced"
date: 2013-03-09 9:13
comments: true
categories: [erlang, spacewar]
---
[GitHub link for the lazy](https://github.com/j14159/eSpacewar)

[Gameplay link for the lazy](http://esw.noisycode.com:8080/sw.html), not guaranteed work _after_ March 31, 2013

##What Is It?
As [mentioned previously](http://noisycode.com/blog/2013/03/02/testing-erlang-spacewar/), I built an online multiplayer-only version of [Spacewar!](http://en.wikipedia.org/wiki/Spacewar!) using [Erlang](http://erlang.org) for all of the game logic and [Cowboy](https://github.com/extend/cowboy) for websocket communication with clients.  The only client at present is a very simple HTML canvas one, [here is a direct link](https://github.com/j14159/eSpacewar/blob/master/priv/sw.html).

{% img http://static.ow.ly/photos/original/1E7sr.png %}

##Why?

###The Basics
I wanted to validate a couple of choices for building simple things requiring:

* the communication of some universal state to clients on a rapid basis
* changing that state based on input from a variety of clients

Looks like a game, sounds like a game but applicable to a lot of other stuff.

###Learning Erlang
Aside from my (so-far) abortive attempt to build a [URL-shortener](https://github.com/j14159/PawnShop), I haven't found an excuse to really dig into Erlang yet.  At the same time, I have been thinking for a while about the possibility of simple games via websockets.  The concurrency and responsiveness requirements for something like this make Erlang a natural choice.

###More With Websockets
They're still sort of stuck in standards-hell and probably not as good as they could be but I wanted to try them out in more depth.  I didn't consider comet for this because the nature of Spacewar requires a lot of small, rapid updates and I _suspected_ (but cannot yet confirm) that the repeated connection setup overhead _might_ negatively impact this.  

**In actual fact**, my assumption/suspicion is extremely likely to be **totally wrong** for the scope of this project.  It probably only makes a difference at an extremely high rate of messages (eSpacewar does a minimum of 20/second to each client) which I'm obviously not dealing with here.

###Derp Cloud Derp
This one's easy.  I just wanted to see if a tiny EC2 node could actually handle it.  It mostly can but the t1.micro it's hosted on now tends to stutter occassionally.

##What I Liked

###Erlang
I could probably go on forever about this (but won't here).  A few specific points that I ended up really loving:

* Pattern matching permeates _everything_ to the point that if/then/else just seems medieval.
* So easy to avoid shared state.  You just don't have any option (slight lie, go with it) so it makes things easier to keep clean.
* Clean, simple and minimal.  The core is so small and the emphasis on recursion so high that you're naturally pushed to composability (although you'll likely find gross contradictions in my code).

I didn't actually make much use of supervision so can't comment on that yet (but love it in the Akka/Scala world).

###Turns Out People Like It
Who knew?  Simple gameplay, easy to drop in and low requirements.  I've seen some great 10 player moments of mayhem.  Would have liked to see more but I think it would make playing almost impossible due to the number of torpedoes flying around.

##What I Screwed Up
This is a non-exhaustive list, see code and comments for more but these are the big things I _think_ I could have done better.  Please feel free to hit me up with agreements/disagreements and suggestions on [Twitter](http://twitter.com/j14159) or in the comments.

###Unit Tests
There aren't any.  I pretty much just slammed through this project to get things working and learn what I could.

###Collision Detection
Collision detection occurs _after_ an entity (torpedo, player) has been moved and is based on points in space.  This means that if players and/or torpedoes are moving quickly, they can appear to pass through each other.  This could be solved by using line segments to see if paths cross.

A further issue is that the server side has no concept of the ship's shape.  It considers a ship to be a point with a certain bounding area and only registers a collision if two entities overlap.

###handle_info
My gen_server and gen_fsm implementations (see [space.erl](https://github.com/j14159/eSpacewar/blob/master/src/space.erl) and [player.erl](https://github.com/j14159/eSpacewar/blob/master/src/player.erl) respectively for examples) could possibly be simplified a bit by using handle_info over handle_cast or specific states in some cases.  More importantly, this may have made the move from basic processes to gen_server and gen_fsm a little easier.

I would most definitely welcome opinions for and/or against the above, I clearly still have a lot to learn.

###Records
I ended up not using records (just haven't looked closely at them yet) and used tuples everywhere.  I suspect I may have been able to simplify some state by using these but am unsure of how this would impact some of my pattern matching use.  I'm aware they're an extension of tuples but don't know if this enforces a certain structure.  An area for futher exploration.

###Client
I'm clearly not an experienced JavaScript developer and need improvement (the playing space size is hard-coded, bad).  One of the areas I _wanted_ to get into but didn't is WebGL but I'll leave that for later.

##Conclusions
I put this into a repo at a fairly early state so you will be able to dig back and see the progression (it was much uglier before).

As a result of this project, I'm confident enough with Erlang (and Cowboy) to start applying it to some more advanced/involved things so I'm going to push on with some new projects.  I'll undoubtably be posting progress on those at some time soon.

The problems above, unfortunately, likely won't get fixed.  I've essentially learned what I set out to and eSpacewar has served well for that process.  Feel free to fork/host/modify, I'd appreciate a link to this blog if you decide to host it (or some derivative) somewhere else.

Comments/criticism most welcome on [Twitter](http://twitter.com/j14159) and in the comments below.  I'll consider pull requests on GitHub but don't really want to mess with this too much further (feel free to fork and fix/extend).
