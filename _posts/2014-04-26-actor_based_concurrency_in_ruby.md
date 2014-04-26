---
layout: post
title: "Actor based concurrency in Ruby"
date: 2014-04-26
tags: concurrency ruby celluloid
---

<blockquote>
In computer science, concurrency is a property of systems in which several computations are executing simultaneously, and potentially interacting with each other.
</blockquote>

# History

For a while now, I've been building a system that has to connect to and keep open many simultaneous long-lived connections towards third party data endpoints. Taking that into consideration, some form of concurrency is a must have. While not a specifically hard problem to solve, it still took some time to get the architecture right.

Ruby is somewhat poor in concurrency primitives (and abstactions). - threads, fibers 

# Problems

First iteration of the system was implemented using EventMachine. It kept many non-blocking connections open, and reacted to data as it came through. Each endpoint had an associated handler that got called when new data came in. It worked fine for a while, but as requirements changed and domain grew bigger, it became hard to model and implement the domain code using callback based techniques.

Another problem with EventMachine is that the library ecosystem is considerably smaller, since every library that has any form of IO must be non-blocking and specifically tailored to work propertly with EM's reactor loop.

Since Ruby is mainly OO language, model conceptualization is easier when thinking in concurrent objects than thinking deferable objects.







Concurrency is hard and it's better to deal with abstractions than OS primitives (threads and processes)

# Because ...

Reactor pattern is not (always) a good concurrency model for IO (EM prevents using already built client libraries)

Callbacks make OOP modeling and conceptualization harder

Problems with Celluloid:

Testing; but much better than in EM.
Knowing what to model as an actor and what not to.
