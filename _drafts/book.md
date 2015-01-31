---
layout: post
title: "Our process"
tags: process
comments: false
---


Building better apps, faster
============================

This book is an effort to document our approach to building apps. Floating Point is a development company focused on the single page apps backed with an API, so all of the examples and techniques are based on this approach. I'm not trying to market this as **the** process for building apps, but it is our process and it works for us. My wish is to share this process with you, not only in a technical sense, but as a collection of ideas that resulted with these technical solutions.

I hope that this book will help you with streamlining your process, and even if it won't completely fit your situation I'm sure you'll get some good ideas out of it.

Why projects fail
=================

In more then a decade of working with clients (small business and huge corporations) it always boils down to bad planning and lack of communication. As developers or designers we always try to fix these with more code, more design, more stuff in general, but essentially, the most important thing is **understanding what you're trying to build**. Only after understanding the problem you're trying to solve, you'll be able to split it to bunch of small problems that can be reasoned about and tackled independently.

> The secret to building large apps is to never build a large app.
> 
> _Justin Meyer, Bitovi CEO_

Although these ideas come from a programming perspective, applying them to all aspects of the development process is crucial for success.

In this book, I'll assume you have a team built from three parts:

1. frontend developers
2. backend developers
3. designers

and each of this parts should be to work independently. Later, I'll show you how to anchor the process in a few important places so they all work towards the same goal, while being independent.

Pipeline
========

For the most of my career, I've worked in the companies that followed seemingly simple and logical process of building stuff (apps, sites, whatever):

1. a client would come to us with an idea or a project,
2. we would design whatever was needed,
3. developers would take over and make the whole thing alive.

The problem with this process, as you can probably guess, is that everything is sequential. Designers would eat as much time as possible - redesigning on the go, because they lacked the deep understanding of the project, and development would be squished in whatever time was left. Developers usually didn't get any kind of project documentation - that's what the PSDs were for. This resulted in constant back and forth between the designers and developers, mostly because designers didn't talk early enough with the developers and designed the stuff which was either impossible or over complicated.

While this process isn't as bad when you're building a site or something really simple, if you're building an app (or even a responsive site), it starts to crumble really soon.

Another issue I've encountered a lot of times is designing for the happy path alone. If you're, for example, designing a form of some kind, this results in a form that doesn't have all states defined, or content boxes designed for certain amount of text and so on. All these things have to be fixed, and usually it was developer's task to hunt down the designer and get all this stuff designed.

Final result of this approach is missed deadlines and overall poor quality of the final product.

>  There aren't any silver bullets, but months can be shaved off of a development project with a few percent improvement in productivity.
>
> John Carmack

While this might look like I'm bashing designers, I don't think it's their fault. All parties in the development process will use up as much time as possible, and I'm sure that building the whole UI by developers first, and then giving it to designers to put pretty colors on it wouldn't produce a better result.

Thankfully I have some experience with design, and while I don't consider myself a designer, I'm aware of the issues designers have to solve every day. They are designing the part that the client is actually seeing, and they need any help they can get to produce a quality product.

Dumb, alone and isolated
------------------------

When I started working in Bitovi, I got introduced to a different way of building apps, one that made me think about all of this and resulted in a different and, in my opinion, a better approach (and this book). Instead of tackling the whole app at once, we build apps component by component. Each component dumb, alone and isolated. This approach allows you to parallelize the development across bigger teams and allows you to focus at one small problem at time. After you have all the components working, you are then able to relatively easily glue them together to form a final product.

Another thing I was introduced to was the *API first* approach. We were building the frontend part of the application and we communicated with the backend through a clearly defined API. CanJS (framework developed by Bitovi) comes with a library that is used to emulate the API responses. This allows frontend and backend to be developed in parallel, and everything just works as long as both backend and frontend teams respect the same API contract.

These two ideas completely changed how I look at application development, and were a solid groundwork for the whole process.

To reiterate, you should be:

1. building your apps out of small, isolated components,
2. defining the API upfront and implementing the backend and frontend in paralel.

With these two techniques alone, you're parallelising the development. But still, we didn't solve all of our problems. In the next chapter I'll take a step back and start from the beginning - understanding the problem.

Understanding the problem
=========================

One of the biggest mysteries in the development process seems to be how to write a good project plan. Some projects don't even have one and rely on the email threads with client which somehow trickle down to designers and developers who have to build the actual thing.

If your answer to this is "Just use Basecamp" or "Just use JIRA" or any other tool, you're trying to solve a communication problem just by throwing more technology at it, which brings us nowhere. The question is what to write in these tools, and how to organize the whole thing.

There are two really good techniques out there that help with this. First one is [Readme Driven Development](http://tom.preston-werner.com/2010/08/23/readme-driven-development.html) which advocates that you write your project's README file first. This will give you a high level overview of your project and enable all of the people working on it to have the same clear vision of what that project is. By constraining it to one file you're ensuring that you won't overdesign the whole thing.

The main thing here is that it's really, really important that every stakeholder in the project **shares and understands the project vision** and the problem the project is trying to solve.

Another idea is personas used by the designers to describe the users of the system.

> In user-centered design and marketing, personas are fictional characters created to represent the different user types that might use a site, brand, or product in a similar way.
>
> http://en.wikipedia.org/wiki/Persona_%28user_experience%29

How far will you go with the persona definition is up to you, but it is crucial to think about them when building a project plan. Each of the personas will use your app in a different way, and you need to account for these.

Usually, we make a readme file per persona, and fill it up with the current understanding of their needs. We don't spend too much time on this upfront, but we do keep it updated as the understanding of the project grows.

If I was defining an "Author" persona for a CMS I would start with something like this:

> ### Author
>
> Authors are the main users of the product. They will produce the content which will be consumed by the readers and moderated by the editors.
>
> Main tasks include:
>
> 1. Writing and editing articles
> 2. Organizing the content in the correct sections
> 3. Tagging the content
> 4. Uploading and organizing photos and other media
>
> Support tasks include:
>
> 1. Editing their user account (username / password)
>
> Authors need access only to the sections they are assinged to (for instance Sports or World News), and shouldn't be able to create or edit content in other sections.

This is pretty short and straight to the point. It's written from a user's perspective and it doesn't talk about the technology at all. The goal of this is not to make a super detailed plan for development, it is supposed to allow you to keep in mind what are you building and for whom.

Having this info in place is a good way to facilitate discussion with your team. Get them all in the same room and discuss what you have. All of them will probably have questions which will help you to either fill in the personas readme file or get the clarification from the client. **Write all of this down.**

With this high level understanding, we're ready to move on to the more technical aspects of the project.


Formalizing the understanding
=============================

(REST) API First
----------------

The time has come to write some code. But we're not doing development yet. The next step is to define the API for your project. With the high level understanding in place you're ready to reason about the app determine what kind of API it needs.

While popular frameworks (like Ruby on Rails) make it easy to map your database to the REST API, your API shouldn't reflect your database schema. It should reflect the business logic of your app, and neither the backend or the frontend should affect it's design.

To continue with the CMS example these are some of the resources we might need:

##### Article:

- title (string)
- excerpt (text)
- body (text)
- cover image (url)
- author
- tags

##### Section:

- name (string)
- articles
- subsections

##### Tag:

- name (string)

##### Author:

- username (string)
- password (string)
- email (string)
- articles

Although this kind of writedown is better than none, we prefer to use [APItizer](http://github.com/retro/apitizer) which allows you to define the resources using the JSON schema. One of the advantages of this approach is that besides the documentation you get the ability to generate the fake data that can be used to develop the frontend application. Another feature is the validation of the backend API endpoints. This way you can use these schema definitions as one of the anchors of your process.

Whenever the backend or the frontend people need to make any changes to the API layer, they can update these definitions to ensure that everybody is aware of these changes.

All of the stuff we talked about in this chapter is a subject to change, and will be changed during the project development. That's why it's important not to use too much time on it. Write down as much as you know and need at the moment, and keep updating during the development.

In this moment you should have a general understanding of the project. Backend people can start building the backend since the API is defined, and as long as they follow the contract, the integration with the frontend will be seamless.

In this book I will not cover the backend development. It will depend on your tech stack, so it's hard to generalise. But the important thing is that the backend is completely independent of the rest of the system, and it doesn't affect the timeline of the frontend development and design.

In next chapter we'll talk about design and frontend teams. In this part of the process, their close cooperation is the key.

Prototyping
===========

In the first chapter we talked about the dumb and isolated components. They are the key in the stramlined frontend development, and we'll use that approach for the design too.


With the general understanding of the projects, personas and the API your teams should have a good idea of what they're building. The time has come to start designing the app. When I say desigining, I'm refering both to the visual / experience part and the architecture of the frontend app itself.

The key is to split the app to as much components as possible. Each of these components should be designed and developed so it works independently of the rest of the system. Let's continue with the CMS example. If we talk about the form page we could split it to the following components:

1. List of form fields (title, excerpt, body) - each field a mini component with it's own design
2. Date picker
3. Tagger

Each of these components is a small isolated world for it's own. But it's important to define all of the possible states of the component too. For instance the date picker components has the following states:

1. Default state - date is empty
2. Date is picked
3. Error state
4. Calendar is open
5. ...

Both the developers and the designers need to be aware of these states, otherwise they won't be able to do their work correctly. Important thing is not to focus on the visual design at this point. You should focus on the experience and flow of the components and app itself.

How are the designers and developers going to colaborate on these is up to you and your team but it's important that they communicate clearly between themselves.

Here are some of the options:

1. Designer does a quick sketch (paper, balsamiq, etc.) that communicates enough to the developers and they build the prototype in the HTML, CSS and JavaScript
2. Your designer does code so he can prototype directly in the HTML, CSS and JavaScript
3. Your designer builds high fidelity prototypes in a tool like AXure and passes it on to the developer.

For us the best workflow is this:

1. Designer is doing the sketches on the whiteboard / paper
2. Developer immediately takes over and implements it
3. Iterate

In cases when the problem can't be described by a static sketch we use AXure. Another advantage of building the components in the isolated and dumb way is that you can reuse them across the projects. A lot of the components will overlap and it can really shorten this process.

We decided to base all of our projects on the Bootstrap which allows us to optimize even more. Everybody is familiar with the CSS and the structure, and a lot of stuff is available out of the box.

To ensure that the client doesn't think that it's the finished app, we use the Balsamiq like theme in this part of the process.

States
------

In the previous part we mentioned that each component can exist in different states. Transition between these states are handled by the business logic of your app, but if you are building the app in the right way this logic will be isolated in the domain part of your code, and the UI will just reflect whatever happens there.

Since we didn't start work on the domain logic yet, we need a way to document and test the components in all of the possible states. Functional tests are a great fit for this problem, and we're using the combination of [FuncUnit](http://funcunit.com) and [Cucumber](http://cukes.info) for this.

FuncUnit is allowing you to emulate user interaction on the page, and Cucumber is a way of declarative test definition in the language that is very similar to English. With a bit of code we combined these tools so they serve a dual purpose:

1. Testing of the component
2. Documentation of all the possible states of a component

This is the workflow we'll use:

1. Write functional tests - one test per state. These tests will mock the state transitions in this phase
2. Write your component's HTML. Write versions for each state. Focus on the logic just enough to power the transitions.

Again, don't spend too much time on this. The idea is to prototype quickly, not to write the detailed implementation. The benefit of this approach is that you get real components in front of your clients as soon as possible. They can click through the test cases and actually see how the component will look in each state. It solves a lot of communication problems.

When your clien signs of on the component, the logic can be fleshed out. Since you have your tests already, it will be easy to make sure you are implementing the correct thing.\

Designer can also benefit from this approach. When the time comes to focus on the visual aspects of the components, they can always consult the docs and the tests to ensure they're designing for all states.

This approach allows us to do little work upfront and get massive benefits for the rest of the process. By building as small components as possible and acknowledging their multiple state nature we can truly optimize the process around it.

If each component has the same structure (and it does) we can approach development of them in the same way, document how the file structure should look, what should be in the documentation and tests, and how to structure the actual code of the component itself. In the next chapter I'll show a practical example of this approach, but I'm sure it can be adjusted to your needs. Thinking about the work we do and optimizing even the small parts of it can give us huge results.

Design
======

In the previous chapters I mentioned that in the prototyping phase designers should work as closely with the frontend developers as possible. This is important even if your designer writes the HTML and CSS for you because they probably won't (and they shouldn't) know all technological constraints of your project.

In the first phase it's important to put the visual aspect of the project aside. One of the worst things you can do is show some visuals to the client, and then later when you figure out that they don't fit the business case explain to them how you were wrong. Iterating on the flow and experience without the burden of the visual aspects allows you to communicate clearly to the client what they should focus on.

![](screenshot.png)

Usual approach to this problem is to make a prototype in one of the tools designed for that, like Balsamiq or AXure. While this approach has it's merit it comes with some obvius downsides:

1. You're working in a wrong medium. Most of these tools produce static images which don't behave like the web apps do.
2. Even if you get the HTML out (like with AXure) everything you build will be thrown away.

This doesn't mean they don't have their place in the process, but be aware of the downsides and include them in the cost calculation.

Another approach is to use something really basic, like sharpie and paper or a whiteboard. [37Signals (now Basecamp)](https://signalvnoise.com/posts/466-sketching-with-a-sharpie) [are a big advocates of this approach](https://signalvnoise.com/archives2/getting_real_ignore_details_early_on.php) and for a good reason. When you're starting to build the app your understanding is to shallow to be able to design everything perfectly.

Another reason to spend as little time on (visual) design as possible upfront is that most apps have very similar components. Seriously, how many different carousels are there to design? If you are building your components in a way that allows reuse, you'll be able to build something that is clickable pretty quickly. Your designers can then click around and determine how to improve the workflow of the app.

After the functional prototype is done (in it's ugly glory), you should have the app's flow figured out. By using something like Bootstrap, you'll get a lot of the information hierarchy nailed down early on, and now you can devote the time to polish.

With all components prototyped, states implemented and tests in place it's easy to dive in adjust the CSS to match the designer's vision. Designers will have everything they need to do the good job. Docs, demos, everything is there.

When working on the visual aspects of the design you should start with the visual direction first. A lot of components will be similar (tables, forms) so there is no need to design each of these separately (Bootstrap helps with this too). Design the common components first and get the feedback from the client. After your design is signed of, frontend people can start working on the implementing it in CSS and you can focus on those few tricky parts of the app that have to be thought out.

Timeline
========

A lot of ideas and techniques in this book aren't original. They were collected through more then a decade of work. Some of them are based on the experience and some of them on the work of other people.

Development process sucks, and there are a lot of approaches to it, and many of them have their uses. In my opinion, this approach is probably not for everyone, and it's constrained both by our team and by the type of applications we build. Other rules come in play if you're building just an API, or a mobile app, but I'll left these to some other authors.

In this chapter I want to talk about another benefit that this approach gives us. About the timeline of development. In the usual process everything is sequential, including the client's feedback.

![](traditional.png)

_In a traditional development process, you'll ask for feedback after you design something and after it's done_

If you do things in a way that was described in this book, you'll have a lot more opportunities to get the feedback from your client. Also, by asking for a feedback on one component level, you'll be able to get answers focused on the problem. Timeline for this process looks something like this:

![](process.png)

More time for development and design, more opportunities for feedback.


# Transparency

Transparency is an extremely important quality for us. We want to keep the process of building apps transparent both for our teams, and for our clients. Working with clients is bound to have problems and there is nothing we can do to completely eliminate them. But we have to do as much as we can to limit the amount of problems comming up and we need to have enough time to resolve these problems.

The process described in this book tries to keep client in the loop as much as possible. Getting feedback early in the development, when changing direction is cheap, is crucial for the success of the project. There is also the benefit of working on small, isolated components. If you need to change something it is much easier when that change is contained in one or two components instead of affecting the whole app.

Communication is the key. People often think that sitting in a room together will result in a good communication, but what it actually gives you is a sense of communication, while a lot of stuff gets lost in the noise. Remote work has it's own challenges, but at least it's a good way to identify communication problems early on. Experience with the remote work made us to think about this process. We want to allow our clients to have a complete insight in what is getting built, how and why.

Writing things down is a must, but this also has to be streamlined. While bad notes are better then no notes, you must be able to get back to the project quickly after months of working on something else. Again, writing things down, and splitting the app to bunch of small components help immensly with this problem. Each of this component must be able to stand on it's own and both you and your clients must be able to understand it based on your notes and demos.

When each component has a readme that explains it's functionality, demos and tests that can easily show you how it's used, it is easy to get in and fix a bug or add another feature.

Reward good practices. When Rails came out some ten years ago people didn't start to use the MVC pattern because it is a good thing to do, they started to use it because Rails rewarded the good app architecture. You got a lot of stuff for free by following a few rules. In this book I tried to show you how we implemented this in our process, and hopefully after getting here you have some ideas of your own.

Conclusion
==========

This book is a collection of my ideas about the development process. They are not engraved in stone and they will be changed as we do more projects and tackle new problems. But the underlining idea will stay the same: How can be streamline our process to produce better work, with better test coverage, with better docs and with more transparency to our clients.

Thinking about the stuff you do every day opens a lot of opportunities to improve. There is a better way to do everything. And don't be afraid to put time in your own tools. They will help you optimize and streamline your process.

Find a way to reward good decisions, and document your best practices. When they survive a few projects automate them and move on to the next thing that is ripe for improvement.
