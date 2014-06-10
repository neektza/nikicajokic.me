---
layout: post
title: "Let it crash (Ruby style)"
tags: concurrency ruby eventmachine celluloid
comments: true
compsci: true
---

Zast serete? 
<blockquote>
Let it crash
</blockquote>

## Setting the stage

For a while now, I've been part of a team that is building a system that has to connect to and keep open many simultaneous long-lived connections towards third party data sources along with polling various REST APIs. Not your regular web app with a few background tasks.

The big requirement that makes this interesting is the long-running process that must deal with many things at the same time. While not a particularly unusual problem, it still took some time to recognize it's inherent concurrent nature and get the architecture right.

## Concurrency primitives (threads and fibers)

Ruby is not poor in language primitives dealing with concurrency. It has support for plain old threads as do many other programing languages. Nothing new there, apart from the GIL. But many words and tears have been shed about the GIL so we'll skip it here and only give you [this](http://www.jstorimer.com/blogs/workingwithcode/8085491-nobody-understands-the-gil) to get you started. 

Besides threads, it has a [coroutine](http://www.ruby-doc.org/core-2.1.1/Fiber.html) implementation idiosyncronosly named "fibers". The most important difference between threads and fibers is that threads depend on the scheduler to manage execution windows, and fibers are cooperative, meaning that being in a fiber, the programmer must explicitly yield execution to another one.

Although it is possible to build a concurrent system using only these constructs, I think it's ill advised to do so. There are many arguments against it, but synchronization of shared state should be enough to make you thik twice about "going raw".

## Concurrency abstractions

Apart from previously described primitives, there are a few higher level concurrency constructs in Ruby in form of libraries/gems.

#### EventMachine

The first is the venerable *EventMachine* - Ruby implementation of the [Reactor pattern](http://en.wikipedia.org/wiki/Reactor_pattern). So basically node.js, built in Ruby, before node.js was cool.

First iteration of the fore-mentioned system was implemented using *EventMachine* as the main concurrency crutch. It kept many non-blocking connections open, and reacted to data as it came through. Each endpoint had an associated handler that got called when new data came in.

This worked fine for a while, but as requirements changed and domain grew bigger, it became hard to model and implement the requirements using handlers and other callback-based techniques. These techniques, while not inherently bad, often result in code that is very nested and execution that is non-sequential (if not using deferables/promises to mask the asynchronicity).

Another problem with the *EventMachine* is that the library ecosystem is considerably poorer, since every library that does any form of IO must do non-blocking operations and has to be specifically tailored to work propertly with EM's reactor loop.

That particular problem hugely affects it's usability, since one of Ruby's most important selling points is exactly the richness of the library ecosystem. This was the primary reason to start looking at other options to deal with the concurrent domain of the problem.

#### Celluloid

Since Ruby is a "pure" OO language (everything is an object, even functions), model conceptualization is considerably easier when thinking in objects than in deferables and callbacks. Celluloid is "just" that - a network of concurrently running objects - ie. the Actor system.

Inspired by the Erlang language and Scala's Akka library, it brings a lot of nice features to Ruby. Most importantly a very OOP-like concurrency model and fault-tolerance subsystem by bringing supervisors and supervision trees to the table. Along with these, it has support for futures (analogous to EM's deferrables and node's promises) so you get best of both worlds.

There's a nice intro on Celluloid's capabilities over at its github [page](https://github.com/celluloid/celluloid). Besides that, there are many other examples and  Coelluloid tutorials on the interwebz, but I recommend [this](https://practicingruby.com/articles/gentle-intro-to-actor-based-concurrency) one in particular because it even implements a minimal actor system at the end.


## Recovering without rescuing

Since Celluloid's concurrency features have described many times over, we'll focus on its (IMO) other very important feature - fault tolerance. 

So what are supervisors? They're basically actors that manage other actors that actually do usefull stuff - workers. If so happens that the worker crashes, the supervisor is responsible for restarting it and notifying other responsible entities about the crash.

Supervisors can supervise more than one worker. All workers under a supervisor are what's called a supervision group. If any of the workers in a certain supervision group crash, the supervior decides what to do with the supervision group as a whole, since a worker crash can impact other workers in the group.

[graf]

Originaly, Erlang supervisors have a couple built-in of strategies to deal with crashes in a supervision group. These strategies are described in the [OTP design priciples](http://www.erlang.org/doc/design_principles/sup_princ.html). There's other useful stuff there about building fault tolerant systems, so you should definitely check it out.

Celluloid uses the one-for-one restart strategy. What that means is that if one of the workers crashes, a supervison only restarts that particular worker.

Here's an example of a crash in a supervision group.
[kod]
