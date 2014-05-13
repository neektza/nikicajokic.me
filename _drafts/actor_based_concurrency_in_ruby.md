---
layout: post
title: "Actor based concurrency in Ruby"
tags: concurrency ruby celluloid
comments: true
---

<blockquote>
In computer science, concurrency is a property of systems in which several computations are executing simultaneously, and potentially interacting with each other.
</blockquote>

# Primitives

For a while now, I've been building a system that has to connect to and keep open many simultaneous long-lived connections towards third party data sources along with polling various REST APIs. In this particular use case, some form of concurrency is a must, although even that wasn't obvious at first.

While not a specifically hard problem to solve, it still took some time to get the architecture right.

Ruby is somewhat poor in language primitives dealing with concurrency. It has support for plain old threads as do many other programing languages. Nothing new there, apart from the GIL. But many words and tears have been shed about the GIL so we'll skip it for now. 

Besides threads, it has a [coroutine](http://link.com) implementation idiosyncronosly named "fibers" - main difference being that threads depend on a scheduler to manage execution windows, and fibers are cooperative, meaning that the programmer must explicitly yield execution to another fiber.

Although it is possible to build a large concurrent system using only these constructs, I think it's ill advised to do so. There are many arguments against it, but these few should be enough to make a point:

* synchronization (locks)
* ?

# Abstractions

Apart from previously listed primitives, there are a few higher level concurrency constructs in Ruby. The first is the venerable EventMachine - Ruby implementation of the [Reactor pattern](http://link.com). So basically node.js, built in Ruby, before node.js was cool.

First iteration of the fore-mentioned system was implemented using EventMachine as the main concurrency crutch. It kept many non-blocking connections open, and reacted to data as it came through. Each endpoint had an associated handler that got called when new data came in. It worked fine for a while, but as requirements changed and domain grew bigger, it became hard to model and implement the domain code using callback based techniques. A lot can be said about callbacks and their effect on code architecture ... ??? ...

Another problem with EventMachine is that the library ecosystem is considerably smaller, since every library that has any form of IO must be non-blocking and specifically tailored to work propertly with EM's reactor loop.

Since Ruby is mainly an OO language, model conceptualization is easier when thinking in concurrent objects than thinking deferable objects and callbacks.

We finally come to the second concurrency abstraction ...

# Celluloid


# Problems with Celluloid:

Testing; but much better than in EM.
Knowing what to model as an actor and what not to.
