---
layout: post
title: "EventMachine internals and the Reactor pattern"
tags: concurrency ruby eventmachine
comments: true
---

Continuing from the previous posts about [primitives and abstractions](/2014/07/14/concurrency_primitives_and_abstractions), in this part of the series we'll pick a few interesting internals from EventMachine's source code, and explain the core ideas behind these snippets. If you're not familiar with EventMachine, [this](http://rubylearning.com/blog/2010/10/01/an-introduction-to-eventmachine-and-how-to-avoid-callback-spaghetti) is a solid post to get you started.

As we mentioned in a previous post, the main idea behind EventMachine is the reactor loop. EventMachine itself has several implementations of this idea targeting different platforms. In this post we'll focus on **pure Ruby implementation**.

## The Reactor pattern

A Reactor is a **process running in an infinite loop reacting to stimulii from the outside world**. Its reactions are codified in the form of registered callbacks that get executed when appropriate conditions are met or certain events occur. 

Let's see what Wikipedia has to say about the topic:

> The reactor design pattern is an event handling pattern for handling service requests delivered concurrently to a service handler by one or more inputs. The service handler then demultiplexes the incoming requests and dispatches them synchronously to the associated request handlers.

In other words, it's a way of managing concurrently occurring events in the outside world. We have various events coming in through one or more channels, and we use the Reactor to cherry-pick events we want to react to by defining callbacks.

In EM, whenever you provide a block to execute under a certain condition, you're actually registering a callback in the Reactor. A condition is usually the completion of an lingering operation, for example:

* an HTTP request has been completed,
* a DB table has been read from,
* a file has been written to.

There's a pattern to be noticed here. We use EM when we want to avoid waiting for IO operations to complete. Instead on waiting, we register a callback that gets executed once the operation is completed and move on to doing other things.

It's an efficient way of doing IO and a nice way of abstracting away the gory details of multi-threading that's usually deployed in similar scenarios.

Let's now see how this pattern works under the hood.

## EventMachine internals

A Reactor in EventMachine is implemented as an object that wraps an infinite loop and provides methods for managing execution. Ruby implements the Reactor as a [Singleton](http://sourcemaking.com/design_patterns/singleton) object with several methods for starting the loop, registering callbacks and timers and signaling loop breaks.

#### Initialization

First, the constructor that initializes the Reactor.

{% highlight ruby %}
  module EventMachine
    class Reactor
      include Singleton
      def initialize_for_run
        @running = false
        @stop_scheduled = false
        @selectables ||= {}; @selectables.clear
        @timers = SortedSet.new
        set_timer_quantum(0.1)
        @current_loop_time = Time.now
        @next_heartbeat = @current_loop_time + HeartbeatInterval
      end
    end
  end
{% endhighlight %}

As we can see in the snippet above, quite a few variables are initialized in the constructor, which are as follows:

* ```@selectables``` -- references to various IO capable things (network connections, files, pipes... sockets in general),
* ```@timers``` -- references to code blocks that needs to be run at a certain point in time, sorted by *time-to-execution*,
* ```@next_heartbeat``` -- a variable that holds the point in time when next the selectable-aliveness check should be done (explained later),
* ```@stop_scheduled``` -- a variable that, if set to ```true```, tells the Reactor to shut down in the next iteration.

Other variables are almost self-explanatory. Let's now introduce and explain a few terms used across EM's source code.

#### The Selectable

A ```@selectable``` is an instance of the ```Selectable``` class which wraps a [socket](http://en.wikipedia.org/wiki/network_socket) (or a [file descriptor](http://en.wikipedia.org/wiki/file_descriptor)) and abstracts away IO operations on it. Since it's assumed that all IO operations should be non-blocking by default, the non-blocking flag[^1] is set in the constructor for anything the ```Selectable``` class wraps.

The class additionally implements methods for controlling, reading and writing from a socket in a non-blocking way (for example, only writing a few packets at a time so the reactor loop doesn't get blocked).

#### The Loop

Next in line is the Loop, a method that actually runs the Reactor and starts the infinite loop.

{% highlight ruby %}
  module EventMachine
    class Reactor
      def run
        raise Error.new( "already running" ) if @running
        @running = true
        begin
          open_loopbreaker
          loop do
            @current_loop_time = Time.now
            break if @stop_scheduled
            run_timers
            break if @stop_scheduled
            crank_selectables
            break if @stop_scheduled
            run_heartbeats
          end
        ensure
          close_loopbreaker
          @selectables.each {|k, io| io.close}
          @selectables.clear
          @running = false
        end
      end
    end
  end
{% endhighlight %}

Looking at the ```run``` method, we can see that all it does is open a loopbreaker (purpose of which we'll explain later) and start an infinite loop which in each iteration: 

1. ```run_timers``` --> runs the registered ```@timers``` when a specified point in time has been reached,
2. ```crank_selectables``` --> checks the read/write state of ```@selectables``` (IO objects) and performs IO operations on them if they're ready, and
3. ```run_heartbeats``` --> checks if all selectables are alive (by invoking a heartbeat method on a selectable).

Finally, the cleanup is done after the Reactor is finished running (usually done when the process is terminated).

Apart from the initialization and the loop itself, the following three methods called in each iteration of the loop are the foundation of EventMachine.

{% highlight ruby %}
  module EventMachine
    class Reactor
      def run_timers
        @timers.each do |t|
          if t.first <= @current_loop_time
            @timers.delete t
            EventMachine::event_callback "", TimerFired, t.last
          else
            break
          end
        end
      end
      def run_heartbeats
        if @next_heartbeat <= @current_loop_time
          @next_heartbeat = @current_loop_time + HeartbeatInterval
          @selectables.each {|k,io| io.heartbeat}
        end
      end
      def crank_selectables
        readers = @selectables.values.select {|io| io.select_for_reading?}
        writers = @selectables.values.select {|io| io.select_for_writing?}
        s = select( readers, writers, nil, @timer_quantum)
        s and s[1] and s[1].each {|w| w.eventable_write }
        s and s[0] and s[0].each {|r| r.eventable_read }
        @selectables.delete_if do |k,io|
          if io.close_scheduled?
            io.close
            true
          end
        end
      end
    end
  end
{% endhighlight %}

The ```crank_selectables``` method is the only interesting one here. It partitions all selectables into two arrays, one with readable and one with writable sockets. It then invokes the ```select``` system call and passes these two arrays to it. The [select](http://ruby-doc.org/core-2.1.2/IO.html#method-c-select)[^2] system call then selects the sockets that are ready for reading or writing and invokes *eventable* reading or writing on them. *Eventable* in this context means a certain number of bytes per cycle.

Other two methods are pretty straightforward in their purpose. The ```run_timers``` method only executes all code blocks from the ```@timers``` set once their time-to-execute has been reached, while the ```run_heartbeats``` runs the ```heartbeat``` method on each selectable every ```HeartbeatInterval``` seconds. The details of the ```heartbeat``` method are covered further down in this post.

These few snippets cover the basic setup and operation of EventMachine, but there are a few more patterns to cover at this point so we can see a clearer picture of what EM in its totality is. Most importantly, we'll cover the Loopbreaker, the Heartbeat and the threadpool.

#### The Loopbreaker

A **loopbreak is a way to signal the Reactor it should do something**. Why do it this way? Well, since it's running an infinite loop, it can't respond to calls - you can think of it as being deaf to all messages while running. Loopbreaker is a form of communication channel between the reactor and the outside world. When something outside the loop signals a loopbreak, the reactor stops for a moment and lets other things happen. It does not terminate the loop, it just allows other things to run.

The loopbreak is signaled whenever we schedule something via the ```next_tick``` or ```defer``` methods, since we need to tell the reactor that something has been scheduled. It can then, in turn, run those tasks. 

However, since it is stuck in an infinite loop, the reactor can't confirm a "hey, you have new stuff to do" type of messages, so we produce a loopbreak signal to command it to check if there are tasks to run.

Here's a (slightly modified [^3]) EventMachine implementation of the loopbreaker:

{% highlight ruby %}
  module EventMachine
    class Reactor
      # ...
      def open_loopbreaker
        @loopbreak_writer.close if @loopbreak_writer
        @loopbreak_reader, @loopbreak_writer = IO.pipe
        LoopbreakReader.new(@loopbreak_reader)
      end
      def close_loopbreaker
        @loopbreak_writer.close
        @loopbreak_writer = nil
      end
      def signal_loopbreak
        @loopbreak_writer.write 'hey!' if @loopbreak_writer
      end
    end
  end
{% endhighlight %}

Internally, a loopbreaker is usually implemented as a one-way IO pipe between entities that are communicating. So, a loopbreak signal is in effect just a few bytes sent over that pipe. When the reactor checks selectables for IO activity, it will see that it has something in the ```LoopbreakReader```, the receiving end of the pipe wrapped in a ```Selectable```, and will start the method to run scheduled blocks.

#### The Heartbeat

A **heartbeat is a way to continuously check the staleness of the selectables**. Every selectable for which a stale state is possible (ie. a long-lived connection) has a ```heartbeat``` method implementation. This method inspects if the connection is stale by checking if there was any activity in a certain specified interval.

Here's the implementation for a ```StreamObject``` that represents any socket used for long-lived data streaming.

{% highlight ruby %}
  module EventMachine
    class StreamObject < Selectable
      # ...
      def heartbeat
        if @inactivity_timeout and @inactivity_timeout > 0
          and (@last_activity + @inactivity_timeout) < Reactor.instance.current_loop_time
          schedule_close true
        end
      end
    end
  end
{% endhighlight %}

The ```heartbeat``` method checks for inactivity and if it deduces that the selectable is inactive, it schedules it for closing.

#### The threadpool

The last thing of interest is the threadpool. Basically, a combination of a queue of tasks, ie. callable objects (blocks) and a pool of Threads that are used to perform those tasks. As we mentioned in the previous post, the threadpool is used to handle blocking IO. Whenever we need to perform a blocking IO call, we do it in a separate Thread. EM's threadpool abstracts this away. We only need to ```defer``` the callable object and EM will take care of it when it can.

Let's see the implementation of this mechanism...

{% highlight ruby %}

module EventMachine
  def self.defer op = nil, callback = nil, &blk
    unless @threadpool
      @threadpool = []
      @threadqueue = ::Queue.new
      @resultqueue = ::Queue.new
      spawn_threadpool
    end
    @threadqueue << [op||blk,callback]
  end
  def self.spawn_threadpool
    until @threadpool.size == @threadpool_size.to_i
      thread = Thread.new do
        Thread.current.abort_on_exception = true
        while true
          op, cback = *@threadqueue.pop
          result = op.call
          @resultqueue << [result, cback]
          EventMachine.signal_loopbreak
        end
      end
      @threadpool << thread
    end
    @all_threads_spawned = true
  end
  def self.run_deferred_callbacks
    until (@resultqueue ||= []).empty?
      result,cback = @resultqueue.pop
      cback.call result if cback
    end
    size = @next_tick_mutex.synchronize { @next_tick_queue.size }
    size.times do |i|
      callback = @next_tick_mutex.synchronize { @next_tick_queue.shift }
      begin
        callback.call
      ensure
        EM.next_tick {} if $!
      end
    end
  end
end

{% endhighlight %}

The three methods in the above code snippet represent the entirety of the EM's Thread pool implementation.

The ```defer``` method to test if there is a threadpool, and if there is none, it creates the ```@taskqueue```, the ```@resultqueue``` and starts the threadpool creation process. Afterwards, it simply pushes a task given to it on a task queue.

The ```spawn_threadpool``` is the one that actually creates the thread pool. It creates a number of threads and sets up each of them to continuously fetch tasks from the ```@threadqueue```. Since ```Queue#pop``` blocks whenever there are no tasks to fetch, threads will wait until there is a task that has to be executed. Once a task to execute is produced, the first thread to fetch [^4] it executes it and pushes the result along with the callback that handles the result to the ```@resultqueue```. It then signals a loopbreak to notify the reactor that it should run the callback that handles the result.

The ```run_deferred_callbacks``` method is not strictly a part of the threadpool system, but since it's tied to it, we'll cover it as such. It's used for running the result handling callbacks. It pops results (along with callbacks) from the ```@resultqueue``` and runs them [^5]. The reason that it's not strictly a part of threadpool system is that it's also used to run the callbacks scheduled via the ```next_tick``` mechanism. It (thread-safely) consumes the ```@next_tick_queue``` for executables to run until it empties the queue. That odd little ```next_tick``` call in the ensure block is just a way of telling the reactor to keep running and bubble up the exception (and not immediately stop) if one happens.

## Next up

I hope I gave a somewhat understandable explanation of EM's internals and that you have a clearer picture in your mind of what's happening under the hood. If you have any questions or comments be sure to leave them in the comments, and I'll give my best to answer them.

Next up, Celluloid and the Actor pattern.

---
[^1]: By using the [```IO#fcntl```](http://ruby-doc.org/core-2.1.2/IO.html#method-i-fcntl) method which is just a ruby wrapper over the [```fcntl```](http://linux.die.net/man/2/fcntl) system call for manipulating file descriptors.
[^2]: The [```IO#select```](http://ruby-doc.org/core-2.1.2/IO.html#method-c-select) method is just a thin wrapper over the [```select```](http://linux.die.net/man/2/select) system call.
[^3]: EventMachine previously used a Pipe, but now uses a UDP socket to signal a loopbreak because pipes aren't available on Windows machines. I find the pipe implementation cleaner and more elegant so I present it as such here.
[^4]: Fetching is done in a thread-safe manner since Ruby's Queue implementation is thread-safe. That's why you should always use it instead of rolling your own, custom Array backed queue.
[^5]: It's important to note that there needn't actually be a callback since we don't need to provide one when deferring, especially if we don't care about the result.
