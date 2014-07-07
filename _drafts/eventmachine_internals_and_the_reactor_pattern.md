---
layout: post
title: "EventMachine internals and the Reactor pattern"
tags: concurrency ruby eventmachine celluloid
comments: true
---

Continuing from previous posts about [primitives and abstractions](link), in this part of the series we'll cover handpicked internals from EventMachine, and explain the gist of it's ideas. You should be familiar with EventMachine's API to grok this.

## The reactor loop

As we mentioned in a previous post, the main idea behind EventMachine is the reactor loop. Eventmachine itself has several implementations of the reactor loop, one in C++, one in Java and one in pure Ruby (pure, meaning without native extensions of any kind). These implementations are used when running in runtimes that provide native extensions. C++ implementation when running in MRI, Java when inside JRuby and finally pure Ruby whenever native extensions cannot be used. We'll focus on pure Ruby implementation in this post.

So, what's a reactor loop? It's basically an infinite loop that continuously checks for registered callbacks to run. Whenever you provide a block to run on some condition, you register a callback in the Reactor.

Ruby implements the Reactor as a [Singleton](http://sourcemaking.com/design_patterns/singleton) object with several methods for starting the loop, registering callbacks and timers and signaling loop breaks.

First, a method that initializes the Reactor.

{% gist neektza/6d4c14f433ae1ab09ed4 reactor_init.rb %}

As can be seen in the code snippet above, the loop is initialized by setting a few variables, mainly:

* ```@selectables``` -- references to various IO capable things (network connections, files, pipes ie. sockets),
* ```@timers``` -- references to code that needs to be run at a certain point in time,
* ```@next_heartbeat``` -- point in time when next the selectable-aliveness check should be done (explained later).

A ```Selectable``` is a class that abstracts and deals with IO operations on a socket. When initialized it sets the socket to be non-blocking (see [fnctl()](http://www.beej.us/guide/bgnet/output/html/multipage/fcntlman.html)) and adds that socket to a list of watched selectables. It aditionally implements methods for controlling, reading and writing from a socket in EM specific context (for example only writing few packets at a time so the reactor loop doesn't get blocked).

Then there's the method that actually runs the Reactor.

{% gist neektza/6d4c14f433ae1ab09ed4 reactor_run.rb %}

Looking at the ```run``` method we can see that all it does is open a loopbreaker (purpose of which we'll explain a bit later) and start an infinite loop which in each iteration: 

1. runs ```@timers``` when a specified point in time has been reached,
2. checks the read/write state of ```@selectables``` (IO objects) and does IO on them if they're ready and
3. checks if all selectables are alive (by invoking a heartbeat method on a selectable).

There are a few more patterns to cover so we can see a clearer picture of what EM is, most importantly the Loopbreaker, the Heartbeat and the Threadpool.

#### The loopbreaker

A loopbreak is a way to signal the Reactor it should do something. Why do it this way? Well, since it's running an infinite loop it can't respond to calls - you can think of it as being deaf to all messages while running. Loopbreaker is a form of communication channel between the reactor and the world outside it. When something outside the loop singals a loopbreak, the reactor stops for a moment and let's other things happen. It does not terminate the loop, it just allows other things to run.

When do we signal a loopbreak? Whenever we schedule something via the ```next_tick``` or ```defer``` methods we need to tell the reactor that something has been scheduled so it can run those things. And, as it can't respond to messages that would tell it "hey, you have new stuff to do" while it's running an infinite loop, we produce a **loopbreak signal** to tell it to run something.

Here's a (slightly modified [^1]) EventMachine implementation:

{% gist neektza/6d4c14f433ae1ab09ed4 reactor_loopbreak.rb %}

Internally, a loopbreaker is ususally implemented as an one-way IO pipe between entities that are communicating. So a loopbreak signal is in effect just a few bytes sent over that pipe. When the reactor checks selectables for IO, it will see that it has something in ```LoopbreakReader```, which is the receiving end of the pipe wrapped in a ```Selectable```, and start the process to run scheduled blocks.

#### The heartbeat

A heartbeat is a way to continuosly check state of selectables. Every selectable for which a stale state is possible (ie. a long-lived connection) has a ```heartbeat``` method implementation. This method checks if the connection is stale by checking if there was any activity in a certain specified interval.

Here's the implementation for a ```StreamObject```.

{% gist neektza/6d4c14f433ae1ab09ed4 selectable_heartbeat.rb %}

The ```heartbeat``` method checks for inactivity and if it deduces that the selectable is inactive it schedules it for closing.

#### The threadpool

The last thing of intereset is the Threadpool. Basically a combination of a queue of tasks, ie. callable objects (blocks) and a pool of Threads that are used to perform those tasks. As we mentioned in the previous post, the threadpool is used to handle blocking IO. Whenever we need to perform a blocking IO call, we do it in a separate Thread. EM's threadpool abstracts this away. We only need to ```defer``` the callbale object EM will take care of it when it can and store the result

Let's see the implementation of this mechanism...

{% gist neektza/6d4c14f433ae1ab09ed4 em_threadpool.rb %}

The three methods in the above code snipped represent the entirety of the EM's Thread pool implementation.

The first method check if there is a threadpool, and if there is none, it creates the ```@taskqueue```, the ```@resultqueue``` and starts the threadpool creation process. Afterwards it pushes a task on a task queue.

The second method is the one that actually creates the thread pool. It creates a bunch of threads and sets up each of them to continuosly fetch tasks from the ```@threadqueue```. Since ```Queue#pop``` blocks if there are no tasks to fetch, threads will wait until there is something for them to do. Once a task to execute is produced, the first thread to fetch [^2] it executes it and pushes the result along with the callback that handles the result to the ```@resultqueue```. It then signals a loopbreak to notify the reactor that it should run the callback that handles the result.

The third method runs the result handling callbacks. It pops results (along with callbacks) from the ```@resultqueue``` and runs them [^3]. Afterwards it runs callbacks that were scheduled via the ```next_tick``` mechanism. It consumes the ```@next_tick_queue``` for blocks to run until it empties the queue.

## Sinteza - eto zato ga je okvord koristit

non-blocking socketi + threadovi


---
[^1]: EventMachine previosly used a Pipe, but now uses an UDP socket to signal a loopbreak because pipes aren't available on Windows machines. I find the pipe implementation cleaner and more elegant so I present it as such here.
[^2]: Fetching is done in a thread safe manner since Ruby's Queue implementation is threadsafe. That's why you should always use it instead of rolling your own custom, Array backed queue.
[^3]: It's important to note that there needn't actually be a callback since since we don't need to provide one when defering if we don't care about the result.
