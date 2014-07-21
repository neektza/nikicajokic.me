---
layout: post
title: "How to not IO"
tags: haskell io
published: true
---

Let say we're building an API client library in Haskell. It has to abstract away all the gory details of making requests to certain API endpoints and has to get the data back in the form of a nice type. If we were building a library for accessing Meetup.com data, for example, we would need to abstract the process of getting "Events" from "api.meetup.com/2/events".

The first and obvious way to do it would be to construct the request from user provided details (such as the API key, the OAuth secret, etc.) using a preferred HTTP library, perform the constructed request and finally and parse JSON into a Haskell type. The most glaring problem with this approach is that we'd be polluting the code of the user of this library with the IO monad. While not a huge problem, I still find it prefereable not to do so if possible.

The second possible approach is to build the request details (again, such as authentication details etc.) into a type that represents the request the client has to make to get the data back. But that has many downsides, the most important being that the user of the library still has to construct the actual HTTP request using one of the available HTTP libraries.

So my question is this... how to structure such a program (an API client library) which would abstarct the ugly parts, but still provide you with something that doesn't pollute your code with the IO monad, if such a thing is possible? In another words, how to keep you library "pure" [^1].

WHAT?

---
[^1]: I'm aware that that the IO monad is not "unpure".
