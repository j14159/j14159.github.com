---
layout: post
title: "IX and the throughput/latency tradeoff"
date: 2015-01-14 19:22:22 -0800
comments: true
categories: learning
---

Since [Saem](https://twitter.com/saemg) put me onto the Coursera ["Learning How To Learn"](https://class.coursera.org/learning-003) course and [Tavis](https://twitter.com/tavisrudd) (as well as [Tyler](https://twitter.com/tylerweir) iirc) pointed me at [Scott Young](http://www.scotthyoung.com/blog/), I've been trying to figure out how to better approach learning all of the material I try to throw at myself.

This coming Monday for the [Polyglot reading group](https://plus.google.com/u/0/communities/110886264051164890990) we're covering the [IX protected dataplane OS](https://www.usenix.org/conference/osdi14/technical-sessions/presentation/belay) [paper](https://www.usenix.org/system/files/conference/osdi14/osdi14-paper-belay.pdf) which describes a set of tradeoffs that operating systems must make when deciding how to handle IO (network in this case).

While initially skimming the paper to try to figure out how I might chunk it to build a more cohesive understanding, I realized that I had an intuitive grip on the tradeoff between throughput and latency but I'd never tried to properly define it for myself.  Being able to clearly define something I think is a mark of actual understanding and builds the ability to leverage it.  I figured this would be a good exercise in that spirit and both Dr Oakley and Scott Young seem to agree from what I've seen so far.

# My Attempt
Describe throughput, latency, and the tradeoff between them as generally as possible but with a view to how they pertain to the IX paper.

## Throughput
Is a measure of how much *uninterrupted* work can be done.

## Latency
Is a measure of how long it takes from submission of some work until its completion.

I think "submission" is likely interchangeable with any of the following if you prefer:

* definition
* creation
* identification

## The Tradeoff
In a system with limited capacity to do work, latency is proportional to the ability to interrupt existing work.  If interruption of current work is *not* permitted, meaning existing work proceeds with *no breaks*, the latency is equivalent to the time it takes to complete the pre-existing work because the system cannot do the new work until it finishes the _existing_ work.  If interruptions *are* permitted without restriction, throughput is entirely bound by the frequency of interruptions.

In a system that preempts work, the balance is struck by limiting the frequency of interruptions.

In a cooperative system, the balance is struck by yielding control after an alotted amount of work is done (note that this depends on altruistic workers).

# Next
I'm going to try to apply this same sort of approach for simpler base concepts.  I think it's roughly in line with the [Feynman technique](http://www.scotthyoung.com/learnonsteroids/grab/TranscriptFeynman.pdf) albeit simplified for the moment perhaps.

If you disagree with my definition with respect to the IX paper, I'd love to hear about it.  As mentioned I've only skimmed the paper so far and so it's entirely likely my definition and/or understanding will change a bit as I dig in deeper.
