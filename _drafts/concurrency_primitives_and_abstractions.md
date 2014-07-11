---
layout: post
title: "Concurrency primitives and abstractions"
tags: concurrency ruby eventmachine celluloid
comments: true
---

*This the first post of a series on Ruby concurrency. Next up is a post about* [EventMachine internals](/2014/07/15/eventmachine_internals_and_the_reactor_pattern).

## Setting the stage

For a while now, I've been part of a team that has been building a somewhat complex web app in Ruby. In addition to conventional web app requirements, this app has to connect to and keep open many simultaneous long-lived streaming connections towards third party data sources along with polling various REST APIs.

The big requirement that makes this problem interesting is the long-running process that must deal with many "actions" at the same time. While not a particularly unusual problem it was a novelty because as web developers we mostly deal with request/response cycles and don't have to manage long running processes. So, naturally, it took us some time to recognize its inherent concurrent nature and get the architecture right.

## Concurrency primitives <small>(threads and fibers)</small>

Ruby is not poor in language primitives dealing with concurrency. It has support for native threads[^1] as do many other programming languages. Nothing new there, apart from the GIL. But many words and tears have been shed over the GIL, so we'll skip it here and only give you [this](http://www.jstorimer.com/blogs/workingwithcode/8085491-nobody-understands-the-gil) to get you started. 

In addition to threads, it has a [coroutine](http://www.ruby-doc.org/core-2.1.1/Fiber.html) implementation named "fibers"[^2]. The most important difference between threads and fibers is that threads depend on the scheduler to manage execution windows, and fibers are cooperative, meaning that being in a fiber the programmer must explicitly yield execution to some other Fiber. As such, they are a nice tool to handle concurrently happening things because they can be suspended and resumed as priorities of handling those things change.

Threads and fibers alone are enough to build sophisticated systems, and although you could build one using only these constructs, I think it's ill-advised to do so. There are a couple of noteworthy higher-level concurrency constructs in Ruby that manage many of the common problems one comes across while building a concurrent system - synchronization of shared state, cross-thread messaging, etc.

## Concurrency abstractions <small>(reactors and actors)</small>

#### EventMachine

The first of the mentioned abstractions is the venerable *EventMachine* - Ruby implementation of the [Reactor pattern](http://en.wikipedia.org/wiki/Reactor_pattern). So, basically, node.js built in Ruby, before node.js was cool.

Curiously, EM doesn't use fibers in its internal implementation, and it uses Threads very sparingly, mainly because it's older than both the native thread implementation and the fiber implementation. It uses an event loop, a combination of non-blocking system calls for IO, and a thread pool when it has to manage calls that are blocking.

First iteration of the fore-mentioned system was implemented using *EventMachine* as the main concurrency crutch. It kept many non-blocking connections open, and reacted to data as it came in. Each endpoint had an associated handler that got called when new data came in. 

This worked fine for a while, but as requirements changed and the domain grew bigger, it became hard to model and implement the requirements using handlers and callbacks. These techniques, while not inherently bad, often result in code that is very nested and execution that is non-sequential, especially if you're not using deferrables/promises as a facelift for the asynchronous parts of the code.

Another problem with the *EventMachine* is that the library ecosystem is considerably poorer, since every library that does any form of IO must do non-blocking operations and has to be specifically tailored to work properly with EM's reactor loop. A prime example of that is ```em-http-request```. Since we can't do blocking IO in a reactor loop (because it blocks the whole app by blocking the loop), we must rely on a library built specifically for EM, which robs us of using any of the cool HTTP libraries in Ruby.

That particular problem hugely affects its usability, since one of Ruby's most important selling points is exactly the richness of the library ecosystem. This was the primary reason to start looking for other options to deal with the concurrent domain of the problem.

#### Celluloid

Fortunately, alternatives exist, the most notable being Celluloid. Inspired by the Erlang programming language and Scala's Akka library, it brings a lot of nice features to Ruby. Most importantly a very OOP-like concurrency model and fault-tolerance subsystem by bringing supervisors and supervision trees to the table. Along with these, it has support for futures (analogous to EM's deferrables and node's promises) so you get the best of both worlds.

Since Ruby is a "pure" OO language (everything is an object), model conceptualization is considerably easier when thinking in objects than in handlers and callbacks. So instead of relying on callbacks, Celluloid provides a way to build a model out of concurrently running objects that are able to talk among themselves - ie. **Actors**.

Celluloid uses Threads internally to represent Actors, and Fibers to represent tasks[^3] (messages) that the actor processes. Since each task can be suspended, actors can process multiple messages and interleave them if needed.

**We'll analyze and explain how it does that in a future post in the series.**
* Ovdje necu ulaziti u detalje jer toliko kompleksna tema zasluzuje novi post, koji je on moj todo-list.

If you need some of the basics explained, there's a nice intro on github wiki [page](https://github.com/celluloid/celluloid/wiki), or you can look at at [this](https://practicingruby.com/articles/gentle-intro-to-actor-based-concurrency) finely crafted blog post about Actor based concurrency in Ruby that even implements a minimal actor system at the end. 

### Abstractions, one-on-one

Direct comparison is not possible, since these two concurrency frameworks work in a fundamentally different ways, but, as we said before




We won't cover basic concurrency in Celluloid since it's covered nicely in other places already. We'll instead skip directly to it's other very interesting feature - **supervisors** and fault tolerance, which is covered in the [2nd part](http://pltconfusion.com/2014/06/10/let_it_crash_ruby_style) of the series.

---
[^1]: Up until version 1.9 Ruby only had a [green thread](http://en.wikipedia.org/wiki/Green_threads) implementation. Native threads were introduced in version 1.9

[^2]: In some languages, they are called Green Threads, but total equivalence between these terms does not stand because in some of these languages they are scheduled by the language itself.

[^3]: Celluloid [defaults](https://github.com/celluloid/celluloid/blob/64e46ee0ecbd848249d0476e8ac512b93bf18485/lib/celluloid.rb#L509) to Fibers to represent tasks, although it also supports a Thread representation.
 
