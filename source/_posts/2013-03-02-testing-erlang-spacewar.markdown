---
layout: post
title: "Testing Erlang Spacewar"
date: 2013-03-02 15:31
comments: true
categories: 
---
##What's Erlang Spacewar?

A multiplayer version of [Spacewar!](http://en.wikipedia.org/wiki/Spacewar!) built in [Erlang](http://erlang.org) with [Cowboy](https://github.com/extend/cowboy) using websockets.  For the moment, it's quite basic and really intended to be a proof-of-concept.  The idea was to see if it is even _possible_ to use websockets for this sort of thing.

Right now I'm just trying to get a sense of how it performs on a tiny little EC2 VM so I'd be most grateful if you could grab a couple friends and jump in when you have a few minutes to spare.  Fair warning:  I fully expect it to fall over and die if there are more than 50 or so people in there but I guess we'll find out.

I'll give you the link to play, don't worry, just a couple quick things...

##How Long Will It Work For?
I'm going to keep it live (aside from crashes, etc) for at least a couple of weeks I think.

##What You Need:
A browser that supports websockets and HTML canvas.  Think recent versions of Chrome and Firefox, Safari _should_ work.

##Problems You Can Expect
###It's Ugly
And I know.  No explosions yet, very minimal.

###It's Choppy Sometimes
It's running on a t1.micro in Amazon EC2's us-east-1 region.  

**Translation**:  it's got less than a full CPU core (really just uses spare cycles from other bigger VMs) and it's in Virginia so if you're on the West coast like me, expect some lag.

###Crashes/Termination
I'll be messing with the code for the next week so do expect unavailability due to upgrades and crashes.

##Please Do
###Play it and tell me what you think
I know about choppiness and ugliness but if you find any other frustrating bits (or even stuff you love), please do tell me in the comments or on [Twitter](http://twitter.com/j14159).

###Share the link
But please share the link to this blog post - I'd like to give all test players the same warnings/notes I'm giving you.

##Are You Going To Release The Source?
Yes.  It'll be reasonably open (probably BSD or MIT license) on my [github account](https://github.com/j14159) within a couple of weeks.  I'm still cleaning things up and moving to gen_server/proper OTP instead of bare Erlang processes.

For the record, the repository will contain the grim history of almost all my mistakes.  It's been a learning experience.

#Shut Up And Let Me Play!
Of course.  Here's the game, don't forget to enter a username and click "Start Playing":  [Erlang Websocket Spacewar!](http://esw.noisycode.com:8080/sw.html)
