---
layout: post
title: "EventMachine internals and the Reactor pattern"
tags: concurrency ruby eventmachine
comments: true
---

Continuing from previous posts about [primitives and abstractions](/2014/07/14/concurrency_primitives_and_abstractions), in this part of the series we'll cover handpicked internals from EventMachine, and explain the gist of its ideas. You should be familiar with EventMachine's API to grok this. If you're not, [this](http://rubylearning.com/blog/2010/10/01/an-introduction-to-eventmachine-and-how-to-avoid-callback-spaghetti) is a solid post to get you started.

As we mentioned in a previous post, the main idea behind EventMachine is the reactor loop. EventMachine itself has several implementations of this idea, one in C++, one in Java and one in pure Ruby (pure, meaning without native extensions of any kind). Appropriate implementations are used when running in environments where native extensions are available. C++ implementation when running in MRI, Java when inside JRuby and finally pure Ruby whenever native extensions cannot be used. We'll focus on pure Ruby implementation in this post.

## The Reactor 

So, what's a Reactor? It's basically an infinite loop that continuously checks for registered callbacks to run. Whenever you provide a block to run on some condition, you register a callback in the Reactor. A condition can be anything that takes some time, for example:

* an HTTP request has been completed,
* a DB table has been read from,
* a file has been written to.

A Reactor is then an object that wraps the reactor loop and provides methods for managing execution. Ruby implements the Reactor as a [Singleton](http://sourcemaking.com/design_patterns/singleton) object with several methods for starting the loop, registering callbacks and timers and signaling loop breaks.

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

As can be seen in the snippet above, quite a few variables are initialized in the constructor:

* ```@selectables``` -- references to various IO capable things (network connections, files, pipes... sockets in general),
* ```@timers``` -- references to code blocks that needs to be run at a certain point in time, sorted by time-to-execution,
* ```@next_heartbeat``` -- a variable that holds the point in time when next the selectable-aliveness check should be done (explained later),
* ```@stop_scheduled``` -- a variable that, if set to ```true```, tells the Reactor to shut down in the nest iteration.

Other variables are self-explanatory.

#### The Selectable

A ```@selectable``` is an instance of the ```Selectable``` class which wraps a [socket](http://en.wikipedia.org/wiki/network_socket) (or a [file descriptor](http://en.wikipedia.org/wiki/file_descriptor)) and abstracts away IO operations on it. Since it's assumed that all IO operations should be non-blocking by default, the non-blocking flag[^1] set in the constructor for anything the ```Selectable``` class wraps.

The class aditionally implements methods for controlling, reading and writing from a socket in a non-blocking way (for example only writing few packets at a time so the reactor loop doesn't get blocked).

#### The Loop

Then there's the method that actually runs the Reactor and starts the infinite loop.

{% highlight ruby %}
  module EventMachine
    class Reactor
      # ...
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

Looking at the ```run``` method we can see that all it does is open a loopbreaker (purpose of which we'll explain a bit later) and start an infinite loop which in each iteration: 

1. ```run_timers``` runs the registered ```@timers``` when a specified point in time has been reached,
2. ```crank_selectables``` checks the read/write state of ```@selectables``` (IO objects) and performs IO operations on them if they're ready and
3. ```run_heartbeats``` checks if all selectables are alive (by invoking a heartbeat method on a selectable).

Finally, some cleanup is done after the Reactor is finished running (usually done when the process is terminated).

Apart from initialization and the loop itself, the following three methods are the foundation of eventmachine.

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

The ```crank_selectables``` method is the only interesting one here. It partitions all selectables into two arrays, one with readable and one with writable sockets. It then calls the ```select``` system call and provides it with these two arrays. The [select](http://ruby-doc.org/core-2.1.2/IO.html#method-c-select)[^6] system call then selects the sockets that are ready for reading or writing and invokes *eventable* reading or writing on them. *Eventable* in this context means a certain number of bytes per cycle.

Other two methods are pretty straghtforward in their purpose. The ```run_timers``` method only executes all code blocks from the ```@timers``` set once their time-to-execute has been reached and the ```run_heartbeats``` runs the ```heartbeat``` method on each selectable every ```HeartbeatInterval``` seconds. The details of the ```heartbeat``` method are covered further down in this post.

These few snippets cover the basic setup and operation of EventMachine, but there are a few more patterns to cover at this point so we can see a clearer picture of what EM in its totality is. Most importantly the Loopbreaker, the Heartbeat and the Threadpool.

#### The Loopbreaker

A loopbreak is a way to signal the Reactor it should do something. Why do it this way? Well, since it's running an infinite loop it can't respond to calls - you can think of it as being deaf to all messages while running. Loopbreaker is a form of communication channel between the reactor and the world outside it. When something outside the loop singals a loopbreak, the reactor stops for a moment and let's other things happen. It does not terminate the loop, it just allows other things to run.

When do is the loopbreak signaled? Whenever we schedule something via the ```next_tick``` or ```defer``` methods we need to tell the reactor that something has been scheduled so it can run those things. And, as it can't respond "okay, will do later" to messages that would tell it "hey, you have new stuff to do" while it's running an infinite loop, we produce a **loopbreak signal** to tell it to check if there's new stuff to do.

Here's a (slightly modified [^2]) EventMachine implementation:

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

Internally, a loopbreaker is ususally implemented as an one-way IO pipe between entities that are communicating. So a loopbreak signal is in effect just a few bytes sent over that pipe. When the reactor checks selectables for IO activity, it will see that it has something in ```LoopbreakReader```, which is the receiving end of the pipe wrapped in a ```Selectable```, and start the method to run scheduled blocks.

#### The Heartbeat

A heartbeat is a way to continuosly check state of selectables. Every selectable for which a stale state is possible (ie. a long-lived connection) has a ```heartbeat``` method implementation. This method checks if the connection is stale by checking if there was any activity in a certain specified interval.

Here's the implementation for a ```StreamObject``` that represents any socket that's used for long-lived data streaming.

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

The ```heartbeat``` method checks for inactivity and if it deduces that the selectable is inactive it schedules it for closing.

#### The Threadpool

The last thing of intereset is the Threadpool. Basically a combination of a queue of tasks, ie. callable objects (blocks) and a pool of Threads that are used to perform those tasks. As we mentioned in the previous post, the threadpool is used to handle blocking IO. Whenever we need to perform a blocking IO call, we do it in a separate Thread. EM's threadpool abstracts this away. We only need to ```defer``` the callbale object and EM will take care of it when it can.

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

The three methods in the above code snipped represent the entirety of the EM's Thread pool implementation.

The first method check if there is a threadpool, and if there is none, it creates the ```@taskqueue```, the ```@resultqueue``` and starts the threadpool creation process. Afterwards it pushes a task on a task queue.

The second method is the one that actually creates the thread pool. It creates a bunch of threads and sets up each of them to continuosly fetch tasks from the ```@threadqueue```. Since ```Queue#pop``` blocks if there are no tasks to fetch, threads will wait until there is something for them to do. Once a task to execute is produced, the first thread to fetch [^3] it executes it and pushes the result along with the callback that handles the result to the ```@resultqueue```. It then signals a loopbreak to notify the reactor that it should run the callback that handles the result.

The third method is not strictly a part of the thread-pool system, but it's used by it, so we'll cover it as such. It's used for running the result handling callbacks. It pops results (along with callbacks) from the ```@resultqueue``` and runs them [^4]. The reason that it's not strictly a part of thread-pool system is that it's also used to run the callbacks scheduled via the ```next_tick``` mechanism. It (thread-safely) consumes the ```@next_tick_queue``` for executables to run until it empties the queue. That odd little ```next_tick``` call in the ensure block is just a way tell the reactor to keep running and bubble up the exception (and not immediately stop) if one happens.

## Next up

With EventMachine done, we're left with analyzing the Celluloid way of concurrency, ie. the Actor pattern, so that's what we'll do next.

---
[^1]: FNCTL
[^2]: EventMachine previosly used a Pipe, but now uses an UDP socket to signal a loopbreak because pipes aren't available on Windows machines. I find the pipe implementation cleaner and more elegant so I present it as such here.
[^3]: Fetching is done in a thread safe manner since Ruby's Queue implementation is threadsafe. That's why you should always use it instead of rolling your own custom, Array backed queue.
[^4]: It's important to note that there needn't actually be a callback since since we don't need to provide one when defering if we don't care about the result.
[^6]: Just a wrapper over the C 
