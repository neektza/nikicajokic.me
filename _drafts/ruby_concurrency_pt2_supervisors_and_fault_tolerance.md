---
layout: post
title: "Ruby concurrency Pt2 - Supervisors and Fault Tolerance"
tags: concurrency ruby eventmachine celluloid
comments: true
---

Continuing from the previous [introduction post](link) on concurrency primitives and abstractions in Ruby...

## Recovering without rescuing

Since Celluloid's concurrency features have described many times over, we'll focus on its other very important feature - fault tolerance and supervisors.

So what are supervisors? They're basically actors that manage other actors that actually do usefull stuff - workers. If so happens that the worker crashes, the supervisor is responsible for restarting it and notifying other responsible entities about the crash.

Supervisors can supervise more than one worker. All workers under a supervisor are what's called a supervision group. If any of the workers in a certain supervision group crash, the supervior decides[^1] what to do with the supervision group as a whole, since a worker crash can impact other workers in the group. Celluloid uses whats known as the "one-for-one" restart strategy, meaning that if a worker crashes, a supervisor only restarts that particular worker.

#### Supervision trees

In any decently complex system there are usually supervisors under supervisors under supervisors... The architecture, consequently, starts to reseble a tree of these workers and supervisors that each have their own resposibilities. We'll take a look at one such example.

<br/>
![Actor architecture example](/images/celluloid_architecture_example.svg)
<br/>
<br/>

In the example above, there are two supervisors and four workers. ```Scraper```, the top level supervisor in the example supervises two workers, a ```Publisher``` and a ```Commander```, and an additional supervisor ```SourceSupervisor```. ```SourceSupervisor``` in turn supervises two more workers, the ```Poller``` and the ```Streamer``` for a certain data source.

The ```Publisher```, as indicated by the diagram, keeps a connection to the PostgreSQL database open, and "publishes" data when it is ready to do so. Similarly, the ```Commander``` has a connection to the RabbitMQ message broker. It reacts to certain messages which tell it to reboot or reconfigure parts of the system while it's running.

The nice thing about these two is that they can keep these connections open, event if the driver creator wrote the database or message bus driver to block on wait. You can do that because each actor runs in its own thread, so if a IO call blocks, it will block only the actor from which the call was initiated.

Marked in green we can see an example of a supervision group. The master of the group is SourceSupervisor. When we initialize it, it in turn boots two more workers - a Poller and a Streamer. We have many such grops, one for each data source, running independently and concurrently. Whenever an error happens in one of the data sources, only a worker that handles that source crashes. At that time the supervisor figures out the worker crashed, it just restarts it.

You're probably thinking that that's not gonna solve anything, but you'd be suprised how often a clean slate will get the system in a stable state again.

So how does this translate into code?

{% gist neektza/a97b02d50b8f85869ce9 supervision_tree.rb %}

#### Explicit linking and monitoring

If you want more control, and want make actors that are not in the same supervision group interdependent and enable one to react to errors of the other, you can use explicit linking. When two actors are linked, the one that does the linking dies if the linked one dies for some reason. 

---
[^1]: Originaly, Erlang supervisors have a couple built-in of strategies (one-for-one, one-for-all, etc.) to deal with crashes in a supervision group. These strategies are described in the [OTP design priciples](http://www.erlang.org/doc/design_principles/sup_princ.html). There's other useful stuff there about building fault tolerant systems, so you should definitely check it out.
