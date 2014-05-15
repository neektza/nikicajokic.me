---
layout: post
title: "Concurrency primitives and abstractions in Ruby"
tags: concurrency ruby celluloid
comments: true
---

<blockquote>
In computer science, concurrency is a property of systems in which several computations are executing simultaneously, and potentially interacting with each other.
</blockquote>

# Context 

For a while now, I've been building a system that has to connect to and keep open many simultaneous long-lived connections towards third party data sources along with polling various REST APIs. While not a particularly unusual problem domain, it still took some time to recognize it's inherent concurrent nature and model it corectly.

# Primitives

Ruby is not poor in language primitives dealing with concurrency. It has support for plain old threads as do many other programing languages. Nothing new there, apart from the GIL. But many words and tears have been shed about the GIL so we'll skip it here. 

Besides threads, it has a [coroutine](http://www.ruby-doc.org/core-2.1.1/Fiber.html) implementation idiosyncronosly named "fibers" - main difference being that threads depend on the scheduler to manage execution windows, and fibers are cooperative, meaning that being in a fiber, the programmer must explicitly yield execution to another one.

Although it is possible to build a concurrent system using only these constructs, I think it's ill advised to do so. There are many arguments against it, but "synchronization of shared state" problem should be enough to make you thik twice about "going raw".

# Abstractions

Apart from previously described primitives, there are a few higher level concurrency constructs in Ruby. The first is the venerable *EventMachine* - Ruby implementation of the [Reactor pattern](http://en.wikipedia.org/wiki/Reactor_pattern). So basically node.js, built in Ruby, before node.js was cool.

First iteration of the fore-mentioned system was implemented using *EventMachine* as the main concurrency tool. It kept many non-blocking connections open, and reacted to data as it came through. Each endpoint had an associated handler that got called when new data came in.

This worked fine for a while, but as requirements changed and domain grew bigger, it became hard to model and implement the requirements using callback-based techniques. These techniques, while not inherently bad, often result in code that is very nested and execution that is non-sequential.

Another problem with the Reactor pattern and *EventMachine* is that the library ecosystem is considerably smaller, since every library that does any form of IO must support non-blocking sockets and has to be specifically tailored to work propertly with EM's reactor loop.

That particular problem hugely affects Ruby's usability, as one of Ruby's  most important strenghts is exactly the richness of the library ecosystem. This was the primary reason to start looking at other options to deal with the concurrent domain of the problem.

So after a while I decided to try the shiny new(er) concurrency tool in Ruby...

# Celluloid

Since Ruby is a "pure" OO language (everything is an object, even functions), model conceptualization is considerably easier when thinking in objects than in deferables and callbacks. Celluloid is "just" that - a concurrent objects implementation for Ruby - ie. the Actor system.

Inspired by Erlang and Akka, it brings a lot of nice features to Ruby, most important being message synchronization (to object inboxes) and fault-tolerance (workers and supervisors). Along with these, it has support for futures (very similar to EM's deferrables and node's promises) so you get best of both worlds. There's a nice intro on it's capabilities on the Celluloid github [page](https://github.com/celluloid/celluloid).

# Pros

Simpler modeling.
Very natural view of concurrency (for anyone coming from OO world).
Fault tolerance.

# Cons

Still young.
Testing; but much better than in EM.
Knowing what to model as an actor and what not to.
