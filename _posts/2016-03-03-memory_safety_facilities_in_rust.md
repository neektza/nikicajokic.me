---
layout: post
title: "Memory safety facilities in Rust"
tags: meetup type_system memory_safety rust
excerpt: Follow up to a talk about Rust's memory safety features
class: talk
comments: false
---

For a while now I have wanted to build my own compiler, because, as Steve Yegge [said](http://steve-yegge.blogspot.hr/2007/06/rich-programmer-food.html):

> If you don't know how compilers work, then you don't know how computers work. If you're not 100% sure whether you know how compilers work, then you don't know how they work. 
>
> You have to know you know, you know.

The thing with building your own programming language is, you can do it in many ways. One way is to build just an interpreter and rely on the programming language you're implementing it in to execute your code. Another is to produce low-level machine code. This second approach is the one I want to take. Either that or I'll build my own bytecode virtual machine.

Since the purpose of building my own programming language is to learn something in the process of building it, I want to be able to go as close to the metal as I want. That means I have to build it in a systems programming language, such as C or C++. But, since I also want to work in a modern language with a powerful type system, I chose Rust.

This talk is my attempt to finally understand and internalize all the fundamental memory safety facilities Rust provides you with to keep you from shooting yourself in the feet[^1].


<iframe src="/talks/memory_safety_facilities_in_rust.html" width="600" height="450"></iframe>

<br/>

The slides were build using [remark.js](https://github.com/gnab/remark) and are hosted [locally](/talks/type_systems.html).

---
[^1]: Although it can sometimes feel like you're banging your head against a wall.
