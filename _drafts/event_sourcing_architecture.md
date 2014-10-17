---
layout: post
title: "Data aggregation and Event Sourcing architecture"
tags: ruby event-sourcing
comments: false
---

A couple of weeks ago I presented on the WebCampZg conference, talking about applying Event Sourcing to a data aggregation domain. You can find the slides here, and the video here. 

A few people at the conference wanted to know more about the technique so I'm expanding it into a full blown blog post. Consequently, in this post I'll go into more details about what we used Event Sourcing for and how we integrated it with the rest of the system.

Writing things down
-------------------

Before going further, let's start of with a quote.

> "Every large enough software system has an badly implemented subset of git contained within itself."
>
> -- Someone on Hackernews

This quote illustrates what ES is about essentially. What I think it's trying to say is that every software system, once it grows to a certain size starts exhibiting certain patterns.

When we encounter complex problems that we can't hold in our heads, we usually respond by by wanting to write things down. When things are written down, we free up precious RAM in our heads that we can now use to break things apart and figure out what is going on.

That's what **git** does for codebases. It writes these changes down and makes it possible to manage them effectively. Analogous to that, Event Sourcing makes data mutations in your system manageable by writing the changes down. It provides insight into what is actually happening under the hood by providing a view of data before and after a change.

First, we'll define the problem we're trying to solve and a bit later we'll show how exactly Event Sourcing helps us solve this problem. 

The problem domain
------------------

The problem domain lies in aggregating data from multiple social networks. When you're handling large amounts of noise in your system and there are interactions between entities from differing sources, transient bugs tend to surface. 

Transient meaning that they come and go without user interaction which results in them being difficult to pin down. Those bugs usually come in form data inconsistencies where you have one state of things in the source, and another in your system. These inconsistencies can also get buried underneath all the noise coming in to your system, so then needn't be transient. It enough that they can are rare, and you will probably fail to spot them.

Now lets look at a few of those inconsistencies as they manifest themselves in our system.

> BUGOVI IZ KEYNOTEA

A sidenote about Patterns
-------------------------

Many people would react to these problems by looking at already known solutions in the forms of Design Patterns. And they wouldn't be wrong. Because patterns usually point to common problems, meaning that many people already encountered the same problem and found some kind of a solution. Then someone smart came along and extracted the most common denominator and codified it in a form of a pattern.

Ultimately, that means that these problems are solvable. That being said, it's not what we done. We didn't know about the Event Sourcing pattern when we started solving these problems and instead we rolled our own version of the solution that ended up looking similar to the original Pattern codified.


The solution?
-------------

ES can help you solve this problem by writing all the intermediate states of the data you're mutating. Every update for an entity in your system is being recorded stored in a database. Meaning that it keeps the history of entities in your system.
