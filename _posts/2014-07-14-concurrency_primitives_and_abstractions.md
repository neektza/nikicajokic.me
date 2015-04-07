---
layout: post
title: Concurrency primitives and abstractions in Ruby
excerpt: An overview (with examples) of Ruby's fundamental concurrency constructs, threads and fibers and how they relate to more complex concurrency abstractions
tags: ruby concurrency threads fibers
comments: true
---

This is part one of the series on Ruby concurrency. Next up, [part two](/2014/10/20/eventmachine_internals_and_the_reactor_pattern) abount EventMachine internals and the reactor pattern.

## Setting the stage

For a while now, my colleagues and I have been building a somewhat complex web app in Ruby. In addition to conventional web app requirements, this app has to connect to and keep open many simultaneous long-lived streaming connections towards third party data sources along with polling various REST APIs.

Recognizing the inherent concurrent nature of the problem domain, I decided to finally obtain a clear understanding of the finer details of Ruby's concurrency primitives and abstractions. This and the next few posts will deal with these topics and attempt to dispel confusion any potential readers may have about them.

## Concurrency primitives <small>(threads and fibers)</small>

Ruby is not poor in language primitives dealing with concurrency. It has support for native threads[^1] as do many other programming languages. Nothing new there, apart from the GIL. But many words and tears have been shed over the GIL, so we'll skip it here and only give you [this](http://www.jstorimer.com/blogs/workingwithcode/8085491-nobody-understands-the-gil) to get you started. 

In addition to threads, Ruby has a [coroutine](http://www.ruby-doc.org/core-2.1.1/Fiber.html) implementation named "fibers"[^2]. The most important difference between threads and fibers is that threads depend on the scheduler to manage execution windows, while fibers are cooperative, meaning that being in a fiber the programmer must explicitly yield execution to some other fiber. As such, they are a nice tool to handle concurrently happening things because they can be suspended and resumed as priorities of handling those things change.

To help differentiate  between thread behaviour vs. fiber behaviour, let's build the classic producer-consumer pattern using each of those to implement it.

We'll introduce a dummy AddOneTask implementation to serve our purposes of sending tasks to run over a queue. The task keeps an internal counter and on each invocation of `perform` just increments the counter.

{% highlight ruby linenos %}
  class AddOneTask
    @@total = 0
    def initialize
      puts "Scheduling task"
      @performed = false
    end
    def performed?
      @performed
    end
    def perform
      unless performed?
        @@total += 1
        @performed = true
      end
      puts "Task performed, state #{@@total}"
      self
    end
    def self.state
      @@total
    end
  end
{% endhighlight %}

Let's now build the producer and the consumer using threads.

{% highlight ruby linenos %}
  require 'thread'
  require_relative 'add_one_task'
  scheduled = []; performed = []; producers = []; consumers = []
  5.times do
    producers << Thread.new do
      scheduled.push AddOneTask.new
    end
  end
  5.times do
    consumers << Thread.new do
      loop do
        task = scheduled.pop
        performed << task.perform unless task.nil?
        break if scheduled.empty?
      end
    end
  end
  (producers + consumers).each { |t| t.join }
{% endhighlight %}


{% highlight text %}
~ lappy :: sandbox/ruby/concurrency > ruby producer_consumer_threads.rb
Scheduling task
Scheduling task
Scheduling task
Scheduling task
Task performed, state 1
Task performed, state 2
Scheduling task
Task performed, state 3
Task performed, state 4
Task performed, state 5
All threads dead? true
{% endhighlight %}

A few notes about the example above:

* shown output is only one of many possible outputs since each run of the example will produce different results (we don't control thread scheduling),
* race conditions are possible and indeed are present in the example (pushing and popping are not thread-safe operations in a regular Ruby array[^4]).

Next up, the fiber implementation of the pattern.

{% highlight ruby linenos %}
  require 'fiber'
  require_relative 'add_one_task'
  scheduled = []; performed = []
  consumers = []; scheduler = nil
  producer = Fiber.new do
    5.times do
      scheduled.push AddOneTask.new
      scheduler.transfer
    end
  end
  5.times do
    consumers << Fiber.new do
      task = scheduled.pop
      performed << task.perform unless task.nil?
      Fiber.yield
    end
  end
  scheduler = Fiber.new do
    consumers.each do |c|
      c.resume
      producer.transfer
    end
  end
  producer.resume
{% endhighlight %}

<br/>

{% highlight bash %}
lappy :: sandbox/ruby/concurrency > ruby producer_consumer_fibers.rb
Scheduling task
Task performed, state 1
Scheduling task
Task performed, state 2
Scheduling task
Task performed, state 3
Scheduling task
Task performed, state 4
Scheduling task
Task performed, state 5
All fibers dead? true
{% endhighlight %}

In contrast to the previous example, fibers need explicit scheduling (even starting them needs to be explicit). We explicitly transfer control from one fiber to another by using `fiber.transfer`, `fiber.resume` and `Fiber.yield`. In the example above, we have a ping-pong-like transfer of control between the `producer` and the `scheduler`. Whenever the `scheduler` gets the execution back, it picks a `consumer` to run. Even though they are harder to produce since we have control over execution, race conditions in fiber-based code are still possible (one example being [this](https://gist.github.com/raggi/1220800)).

Considering the simplicity of the implementations above, it seems that threads and fibers alone are enough to build complex systems, and although you could certainly build one using only these constructs, it's ill-advised to do so. There are a couple of noteworthy higher-level concurrency constructs in Ruby worth considering that solve many of the common problems one comes across while building a concurrent system - synchronization of shared state, cross-thread messaging, etc.

## Concurrency abstractions <small>(reactors and actors)</small>

### EventMachine

The first of the mentioned abstractions is the venerable *EventMachine* - Ruby implementation of the [Reactor pattern](http://en.wikipedia.org/wiki/Reactor_pattern). So, basically, node.js built in Ruby, before node.js was cool.

Curiously, EM doesn't use fibers in its internal implementation, and it uses threads very sparingly, mainly because it's older than both the native thread implementation and the fiber implementation. It uses an event loop, a combination of non-blocking system calls for IO, and a thread pool when it has to manage calls that are blocking.

First iteration of the aforementioned system was implemented using *EventMachine* as the main concurrency crutch. It kept many non-blocking connections open, and reacted to data as it came in. Each endpoint had an associated handler that got called when new data came in. 

This worked fine for a while, but as requirements changed and the domain grew bigger, it became hard to model and implement the requirements using handlers and callbacks. These techniques, while not inherently bad, often result in code that is very nested and execution that is non-sequential, especially if you're not using deferrables/promises as a facelift for the asynchronous parts of the code.

Another problem with the *EventMachine* is that the library ecosystem is considerably poorer, since every library that performs any form of IO must do non-blocking operations and has to be specifically tailored to work properly with EM's reactor loop. A prime example of that is `em-http-request`. Since we can't do blocking IO in a reactor loop (because it blocks the whole app by blocking the loop), we must rely on a library built specifically for EM, which robs us of using any of the cool HTTP libraries in Ruby.

That particular problem hugely affects its usability, since one of Ruby's most important selling points is exactly the richness of the library ecosystem. This was the primary reason to start looking for other options to deal with the concurrent domain of the problem.

### Celluloid

Fortunately, alternatives exist, the most notable being Celluloid. Inspired by the Erlang programming language and Scala's Akka library, Celluloid brings a lot of nice features to Ruby, most importantly a very OOP-like concurrency model and a fault-tolerance subsystem that brings supervisors and supervision trees to the table. Along with these, it has support for futures (analogous to EM's deferrables and node's promises) so you get the best of both worlds.

Since Ruby is a "pure" OO language (everything is an object), model conceptualization is considerably easier when thinking in objects than in handlers and callbacks. So instead of relying on callbacks, Celluloid provides a way to build a model out of concurrently running objects that are able to talk among themselves - ie. **actors**.

Celluloid uses threads internally to represent actors, and fibers to represent tasks[^3] (messages) that the actor processes. Since each task can be suspended, actors can process multiple messages and interleave them if needed. We'll not delve too deep into the analysis of how Celluloid manages tasks, since a post that will paint a clearer picture of this process is already in the making.

If you need some of the basics explained, there's a nice intro on github wiki [page](https://github.com/celluloid/celluloid/wiki). You can also look at [this](https://practicingruby.com/articles/gentle-intro-to-actor-based-concurrency) finely crafted blog post about Actor based concurrency in Ruby that even implements a minimal actor system at the end. 

**Next up**

Since there are many posts out there that focus on explaining how to build a concurrent system by relying on these abstractions, in the next two posts we'll focus instead on how these abstractions are implemented from basic building blocks - threads, fibers et al.

Next post will explain the main ideas behind EvenMachine and dig into its implementation details to paint a clearer picture of what EventMachine is all about.

---
[^1]: Up until version 1.9 Ruby only had a [green thread](http://en.wikipedia.org/wiki/Green_threads) implementation. Native threads were introduced in version 1.9.

[^2]: In some languages, they are called Green threads, but total equivalence between these terms does not stand because in some of these languages Green threads are scheduled by the language VM itself.

[^3]: Celluloid [defaults](https://github.com/celluloid/celluloid/blob/64e46ee0ecbd848249d0476e8ac512b93bf18485/lib/celluloid.rb#L509) to fibers to represent tasks, although it also supports a thread representation.
 
[^4]: That's why you should always use a Queue to communicate between threads.
