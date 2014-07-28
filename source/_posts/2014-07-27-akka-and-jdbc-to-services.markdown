---
layout: post
title: "Akka and JDBC to Services"
date: 2014-07-27 14:05
comments: true
categories: 
---
I started mapping out this post over a month ago after I saw another question about how to handle blocking database calls on the Akka mailing list.  I've been working towards most of what follows over the last year and after discussing the basics with a few others in the [Vancouver Polyglot](http://www.meetup.com/PolyglotVancouver/) community with some additional prompting from [Saem](https://twitter.com/saemg), I finally put this together ([project link](https://github.com/j14159/akka-jdbc-post) for the lazy).

This represents my current thinking (as of roughly July 2014) about a reasonable approach to the problems of state, blocking, and failure for data store interaction within the confines of Scala and Akka.  This post specifically focuses on JDBC connections but is roughly applicable to any system that provides a blocking API.  I define "reasonable approach" to be one that has significant enough upsides to combat its explicitly acknowledged weaknesses and negatives (I'll get to those towards the end).  The particular approach I'm outlining here is a way to structure an application such that you have some reasonable flexibility later on and some clear benefits up front, yielding a future-viable design.

# Actors And Databases

The initial question often seems to be "how do I handle JDBC/blocking calls with Akka", the answer to
which almost always involves "bulkheading" which we'll get to a bit later in this post.  The issue I want to tackle first is one of state.

Database connections:

* _Are_ state
* Have a lifecycle (connected, in-transaction, disconnected, etc)
* Are prone to errors (timeouts, network partitions, etc)

Database connection lifecycles should be tied directly to an actor's lifecycle with errors on the connection handled by supervision.  A 1:1 Connection:[DAO](http://en.wikipedia.org/wiki/Data_access_object) actor is specifically what I am advocating.  I don't think the DAO approach suffers from lack of acceptance or understanding so I'm not going to get into specifics about it here.  The basics around actor-connection coupling are as follows:

* Actors get a connection factory method with the most basic type being ````() => java.sql.Connection````.

  This method can be constructed elsewhere, closing over the necessary _immutable_ state required to instantiate a connection, e.g. JDBC URL.
  
* On any crash, the connection simply gets closed and the replacement actor instance will lazily recreate its required connection.  Supervision strategies and preRestart() behaviour can be easily tailored to not always kill the connection as this default behaviour is obviously not always desirable.  Tearing down the connection and starting a new actor instance on a column index error is clearly extreme.

{% gist 00b097487976b6486513 %}

Anyone who has deployed Erlang/OTP or Akka applications into a production will have noticed the problem that I have just introduced:  Akka actors (like processes and gen_servers in Erlang) by default have an **unbounded** mailbox.  Since we're making blocking calls inside the database actor, when a burst of requests occurs they will all queue inside the actor's mailbox without any sort of back pressure.  Parallelism *partially* helps to solve this problem.

## Parallelism and Pooling

[Release It!](http://www.amazon.com/Release-It-Production-Ready-Pragmatic-Programmers/dp/0978739213) advocates against writing one's own connection pool and in general I think Mr Nygard puts it fairly well:

> ...I oppose anyone rolling their own connection pool class.  It is always more difficult than you
> think to make a reliable, safe, high-performance connection pool.

Having said that, I will now happily violate his basic rule as well as [Chris Stucchio's position](http://www.chrisstucchio.com/blog/2013/actors_vs_futures.html) by saying the use of a [router](http://doc.akka.io/docs/akka/2.3.4/scala/routing.html) and the previously mentioned supervision strategy applied to a pool of actors is reasonably sound, safer due to less sharing of mutable state (a traditional connection pool), and performant enough (with caveats to follow).

**UPDATE:**  [James Roper](https://twitter.com/jroper) was kind enough to correct some mistaken assumptions on part with respect to connection pooling below.  I'll be giving some thought over the next while as to a good way to leverage connection pooling within this approach.

Using supervision strategies to enable both configurable and *adaptable* responses to errors is better than an opaque connection pool.  Seth Falcon's [pooler](https://github.com/seth/pooler) project (hat tip to [James](https://twitter.com/jamesgolick) for suggesting it to me) is where I first started to understand how useful and fundamentally sane this approach can be.  While I think it is safe to assume that JVM connection pool implementations these days are fast enough and avoid things like ````synchronized```` and heavy locks, passing around a reference to such an object containing so much mutable state is at odds with much of what makes both Scala and Akka effective.

It is important to note that a realistic supervision strategy would not cause the actor in question to restart on every exception, discarding the existing connection and creating another.  If the connection itself has no problems, "Resume" is a perfectly legitimate response from the strategy in many cases.  Your strategy must be
tailored to your particular problem and requirements.

{% gist 49606a5468c734fc6a28 %}

Those in the audience paying attention will note that even though we now have parallel handling of requests, we have still not solved the actual queue problem.  Two of the simplest solutions are:

* Limit the number of requests in flight with a different actor mailbox type:  ````akka.dispatch.BoundedMailbox````.  The downside here is that message sends are now blocking which may add to throughput issues.
* Use proper timeouts with the "ask" pattern and a [circuit breaker](http://doc.akka.io/docs/akka/2.3.4/common/circuitbreaker.html) to fail faster and back off.  More on this approach later.

# Bulkheading

"Bulkheading" is a [pattern](http://letitcrash.com/post/40755146949/tuning-dispatchers-in-akka-applications) used to protect one set of components from negative influence by one or more _different_ sets of components in your system.  Here we're specifically concerned with preventing the set of actors making JDBC calls from using up all of the compute resources (threads) used for *other operations*, e.g. responding to web requests, recording metrics, reporting on JVM stats, etc.

As an example, let's suppose you have a simple application supporting a couple of basic GET requests:

* ````/user/:id```` to get a specific user object from the database by ID
* ````/ping```` to check if the system is alive

Further, let's assume you're using something like the Spray library and thus have actors servicing HTTP requests.  If your single Akka dispatcher has a maximum of N threads and you have M or more _parallel_ user requests occurring at the same time, when M >= N any call to the ````/ping```` endpoint will block without a response until one or more of the user requests complete.  Lest you think that success or failure of these calls is guaranteed to be fast (or even to occur at all), I encourage you to read [Release It!](http://www.amazon.com/Release-It-Production-Ready-Pragmatic-Programmers/dp/0978739213) at a minimum.

In order to **bulkhead** everything else from the JDBC actors, we create a different dispatcher for the database actors and spawn them only on this separate dispatcher.  Now the non-database components will not be affected when the database actors have blocked all of their threads.  Note that this is not a panacea - should all of the database actor threads be blocked, any component that relies on this pool of actors will be prevented from making progress until earlier calls complete.

Dispatcher configuration:
{% gist ab55d058c118f57075ae %}

And a minor change to our pool with a router:
{% gist 8219db85e98e2dc713d5 %}

This is just the beginning of isolating and protecting different pools of actors and tasks (e.g. futures).  While I don't want to get too deep into tuning dispatchers and prioritizing access for different components, you should be aware that there are a variety of ways to affect quality-of-service for clients of your actors.  Here are some of the simplest places to start:

* The number of threads per dispatcher (with high and low thresholds) can be adjusted as needed.  If you need more throughput, make more actors and give them correspondingly more threads.  Don't forget that running more connections has its own penalties and limitations.
* A "pinned" dispatcher can be used wherein each actor gets its own thread all to itself.
* For higher-priority clients, give them their own larger pool of actors on yet another dispatcher if necessary (almost last resort I think).

# DAOs

Those of you with some experience will note that I'm actually conflating [DAO](http://en.wikipedia.org/wiki/Data_access_object) and [Aggregate Roots](http://en.wikipedia.org/wiki/Domain-driven_design#Building_blocks_of_DDD) for reasons that will become more clear shortly.

Each aggregate root gets an actor implementation that provides all of the necessary CRUD functionality or whatever subset is needed (a DAO with a message-passing interface).  In addition I favour creating unique case classes for each type of request that the DAO actor services rather than more ambiguous strings or tuples.  All interaction with the actor is performed via the [ask pattern](http://doc.akka.io/docs/akka/2.3.4/scala/actors.html#Ask__Send-And-Receive-Future).

Hiding CRUD mechanics is important because SQL or some DSL analogue appearing throughout your code and packages is a leak.  Data store details are details about serialization and with very few exceptions should *not* be leaking into your business logic.  "Save this thing", "find things that match these criteria", etc are not specific mechanics, they're closer to _goals_ that need to be satisfied.  "INSERT x INTO TABLE y" are particulars that are not germane to your business logic.

{% gist 6aee8831ffb10747f226 %}

# Failure as a First Class Citizen

Rather than the raw ask pattern, informed somewhat by common Erlang approaches to gen_server interfaces I always create a client trait  in front of the DAO actors returning [Futures](http://www.scala-lang.org/api/2.10.3/?_ga=1.34556641.1524243934.1391722094#scala.concurrent.Future) that can easily be turned into a standalone class or object where necessary (e.g. testing).

{% gist ca191b61a73382316f9c %}

Why Futures?

* They keep failure handling front and centre via recover() and recoverWith() (I believe [Finagle](https://twitter.github.io/finagle/) may be responsible for this).
* Explicitly asynchronous.
* Easy to combine and parallelize via map(), flatMap(), zip(), for-comprehensions, etc.

The first point bears emphasis:  futures in Scala and Finagle expose failure and *doing something with that failure* as a primary concern.  For the simplified starting point described here, timeouts are the primary failure we're initially concerned with.  To that end, we can combine sensible timeouts with [circuit breakers](http://doc.akka.io/docs/akka/2.3.4/common/circuitbreaker.html) to help solve the queue growth problem and keep our systems responsive.

On sensible timeout values, I will defer to [Pat Helland](http://queue.acm.org/detail.cfm?id=2187821):

> Some application developers may push for no timeout and argue it is OK to wait indefinitely. I typically propose they set the timeout to 30 years. That, in turn, generates a response that I need to be reasonable and not silly. Why is 30 years silly but infinity is reasonable? I have yet to see a messaging application that really wants to wait for an unbounded period of time...

By embracing failures as legitimate responses, we can begin to build systems that degrade gracefully and continue to function in the face of most problems that occur (see most of what has been written by the Netflix team).  This point cannot be overstated:  when talking to a data store, you are explicitly making calls to an external and _very probably remote_ system of some sort and thus failure *must* be accounted for and *must* be handled.

# Smells Like A Service

When constructed with the above approach, an application has effectively been built as a set of internal services.  Since all of the clients talk to the internal services via a mechanism that expects failure and encourages its proper handling, you can simply swap out the implementation behind the scenes for remote services whenever such growth is necessary, e.g.

* [Dispatch](http://dispatch.databinder.net/Dispatch.html) HTTP calls already return futures.
* Netty or Akka TCP clients writing to a [Promise](http://www.scala-lang.org/api/2.10.3/?_ga=1.34556641.1524243934.1391722094#scala.concurrent.Promise).
* Actors talking to external services via RabbitMQ, 0MQ, etc (though not what I would _generally_ recommend).

It is far simpler to move functionality to an external application (separate service) in this manner than with a more traditional DAO or ORM approach.

# Explicit Negatives

* Using the ask pattern means that responses from the actor are typed as ````Any````, which effectively bypasses type checking.  One solution with its own issues is to send an already constructed promise with your message to the DAO actor using ````tell````/````!````.  The DAO actor could then write a success or failure to this promise, yielding proper type safety but **complicating** timeouts.

  If you rely entirely on the Akka circuit breaker implementation, the problem can be mostly solved.  Since the circuit breaker will return a failed future after a timeout you specify, you can send an already constructed Promise to the actor and give the circuit breaker the result of Promise.future.  Since the circuit breaker does not have access to the promise you constructed, you should have no problems with a race to write a result to it on low timeouts and/or slow actors (see the [example project](https://github.com/j14159/akka-jdbc-post) for this method).
* It should be fairly obvious that with complex problem domains and a lot of models the actor and thus connection count can get high.  When this occurs, you already have an interface that looks like a service and provides reasonable support for failure handling.  Move larger and/or high-traffic aggregate roots out of the application into actual separate services, swapping out the functionality behind the Future-based API without changing the interface.  The only difference to callers should be at most some different exception classes on failure and completion times.

  While I think externalizing services like this is generally a great idea, exercise caution.  Move [bounded contexts](http://martinfowler.com/bliki/BoundedContext.html) out first and don't forget that when you move to an external service, you add a set of latencies in serialization, deserialization, and additional network traffic.  Ensure that you both _measure_ and account for this.
* Connections are hidden so the functionality provided by your DAO actors must be complete (no arbitrary SQL).  You can get around this by building an actor that takes closures or something along those lines as transaction blocks.  See [Querolous](https://github.com/nkallen/querulous) for this sort of idea but I think this is missing the point in the context of this particular discussion.
* Transactions that span multiple aggregate roots are essentially impossible in any _traditional_ sense (those that are handled entirely inside the database), as they also are in most honest distributed system designs.  People like [Pat Helland](http://www.ics.uci.edu/~cs223/papers/cidr07p15.pdf) and [Peter Bailis](http://www.bailis.org/) amongst many others are doing very interesting work in this area (apologies to the many deserving people I've left out here, too many to name).
* Timeouts are not guaranteed to trigger failure exactly when you want them to (it may take longer).
* The router + actor approach introduces aspects of queue growth as described above so there is an obvious need for back pressure and capacity planning.  Circuit breakers help mitigate this but they're not a silver bullet.
* Boilerplate.

# The Immediate Benefits

There are three particular benefits for any system, especially if done in the early stages of its life:

* APIs that are asynchronous, composable, and failure-centric.
* Easily configurable parallelism and limits.
* Comprehensible subcomponents (the most important benefit of service oriented architectures whether micro or otherwise).

At present I think the above strengths combined with the relative ease of partitioning a system into other more granular services later on (not to mention CQRS and event sourcing) make this a reasonable approach to designing a system.  While I'm not 100% happy with the boilerplate feeling I get at times, the concrete isolation of the models into their initially internal services has definitely helped me already to keep things clean and understandable.  Having said that, it is far from lost on me how much less code it takes to do the same thing in Erlang with the caveat that I have yet to see something comparable to futures in for-comprehensions there (please correct me if I'm missing something like this!).  If you missed the two links above, my example project using the send-in-promises approach is [available here](https://github.com/j14159/akka-jdbc-post).

Thanks to [Saem](https://twitter.com/saemg) and [Ed](https://twitter.com/eddsteel) for proof reading and pointing out where more clarity was necessary.  Comments, criticism and general dicussion welcome in comments or on [Twitter](https://twitter.com/j14159).

