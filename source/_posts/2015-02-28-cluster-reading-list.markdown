---
layout: post
title: "Cluster Reading List"
date: 2015-02-28 14:28:38 -0800
comments: true
categories: 
---

At [work](http://askuity.com) we use [Mesos](http://mesos.apache.org/) to manage a dynamic pool of resources for our [Spark](http://spark.apache.org) jobs and while I've also dabbled with [Hadoop](http://hadoop.apache.org/) in the past, I've found my understanding of the mechanics and initial design concerns of all of the above a bit lacking.  Rather than parrotting the standard marketing lines, I wanted to understand the fundamental reasons and motivation behind treating a pool of machines (virtual or otherwise) as an anonymized pile of aggregate CPUs and RAM for many different types of work.  A non-exhaustive list of things I want to understand are:

* How are larger jobs broken down and parallelized?
* How are parts of jobs made to coexist in an environment of constantly changing demands?
* How do we ensure full utilization of extant resources and do we _need_ to in a cloud environment as opposed to  simply scaling the resources?
* While I understand the basics of being concerned with "data locality" (get the computation as close as possible to the data), what are some practical ways to achieve this in a general way, applicable to many applications?

In hopes of answering these (and other) questions, I started to compile this reading list of papers that seemed like they would help answer them.  An earlier form of this list was posted as a gist [here](https://gist.github.com/j14159/19d100a556effacd1475).

At present I have completed the Google File System, MapReduce, and Microsoft Dryad papers, all of which I have thoroughly enjoyed reading and they have already contributed to a better understanding of how I should be approaching Spark.

# The List
This list was built primarily by crawling through the references sections of a few different papers.  Some of these have come by way of suggestions from [Omar](https://twitter.com/omarkj) and [Michael](https://twitter.com/ook) as noted below.  In pursuit of a reasonable reading and study plan for myself, I tried to lay the material out in something of a logical path from early successes to more recent ideas and implementations.

If you feel I've left out anything particularly awesome or am incorrect in any assumptions/descriptions with respect to the later papers, I'd love to hear about it and will update my list accordingly.

## Fundamentals
A brief list of papers and ideas that I thought would give me something of a foundation in what the main problems, concerns, and early solutions were.

1.  [Google File System](http://static.googleusercontent.com/media/research.google.com/en//archive/gfs-sosp2003.pdf) - most of the later material references this paper and it seems like this enabled a lot of other ideas later on.  The distribution of chunks of a file around a cluster enables a lot of interesting data locality stuff.
2.  [Google MapReduce](http://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf) - a pretty lucid and well thought out divide-and-conquer approach to parallelizing a huge job.  The functional programming part of me loved the LISP shout-out/inspiration.
3.  [Dryad](http://research.microsoft.com/en-us/projects/dryad/eurosys07.pdf) for a more fine-grained/raw approach to programming against a cluster.
4.  [Quincy](http://www.sigops.org/sosp/sosp09/papers/isard-sosp09.pdf) is, from what I understand so far, a more detailed look at an alternative scheduler for Dryad.
5.  [Delay Scheduling](http://people.csail.mit.edu/matei/papers/2010/eurosys_delay_scheduling.pdf) for a detailed look at a different Hadoop scheduler.  Note the involvement of [Matei Zaharia](https://twitter.com/matei_zaharia) who later worked on both the Mesos and Spark projects.

## Cluster Schedulers
Just the two big ones here, unfortunately there don't seem to be any papers out there on Google's Borg project.

1.  [Mesos](http://people.csail.mit.edu/matei/papers/2011/nsdi_mesos.pdf) as mentioned above is what I currently use to manage Spark resources.
2.  [Omega](http://eurosys2013.tudos.org/wp-content/uploads/2013/paper/Schwarzkopf.pdf) is (was?) a project whose aim is/was to fix shortcomings in Google's existing cluster scheduling/management system (assuming Borg).

## More Programming Models

1.  [DryadLINQ](https://www.usenix.org/legacy/event/osdi08/tech/full_papers/yu_y/yu_y.pdf) is a higher-level programming model built on top of Dryad.  Composition FTW.
2.  [Pregel](http://kowshik.github.io/JPregel/pregel_paper.pdf) a graph processing paper from Google.  References Dryad as well but I'm not sure how close it is as I haven't read it yet and it seems to be less about dataflow.
3.  [MapReduce Online](http://db.cs.berkeley.edu/papers/nsdi10-hop.pdf) for an iterative MapReduce.
4.  [Distributed GraphLab](http://vldb.org/pvldb/vol5/p716_yuchenglow_vldb2012.pdf) sounds like another graph-like computation model but one that embraces asynchrony and parallelism a bit more.  Possibly belongs in the next category but I'm not sure yet.

## Data Flow

1.  [MillWheel](http://static.googleusercontent.com/media/research.google.com/en//pubs/archive/41378.pdf) as recommended by [Omar](https://twitter.com/omarkj) talks about stream processing but does appear to tackle the problem with a form of computation graph that looks like a dataflow approach.
1.  [Naiad](http://sigops.org/sosp/sosp13/papers/p439-murray.pdf) is relatively recent from Microsoft Research (2013) and intriguingly mentions "cyclic dataflow" (also via [Omar](https://twitter.com/omarkj)).
2.  [Derflow](https://www.info.ucl.ac.be/~pvr/erlang14cameraready.pdf) is about "deterministic dataflow" with Erlang.  Sounds too good to pass up.

## Big Ideas

1.  [Condor](http://research.cs.wisc.edu/htcondor/doc/condor-practice.pdf) is referenced both by MapReduce and Dryad as an inspiration for cluster management approaches so I've added it to get a bit more perspective at the end.
2.  [The Datacenter as a Computer](http://www.morganclaypool.com/doi/pdf/10.2200/S00193ED1V01Y200905CAC006) (thanks to [Michael](https://twitter.com/ook) for the reminder) is a bit over 100 pages but seems to cover a lot more of Google's general approach to clusters and more.
