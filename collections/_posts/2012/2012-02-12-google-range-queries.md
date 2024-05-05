---
layout: post
title: "Range queries in Google"
categories: blog
---

When using Google to search for stuff, most of the time you're just throwing keywords at it. But it can do much, much more. Here's a neat little feature most people don't know of (because let's face it, it doesn't have an awful lot of uses): range queries. What are range queries? Well, it's a fairly common data retrieval task, where you want all records where some value is between an upper and a lower bound.

In Google, it works like this:

![Google](/assets/img/blog/2012/02/range-query-1.png)

Here's some results this query gives:

![Google](/assets/img/blog/2012/02/range-query-2.png)

Pretty neat, no?

The following is a list of what can be done using this operator:

*   x..y matches all numbers between x and y (including x and y themselves)
*   x.. matches all numbers greater or equal to x
*   a.b..c.d matches decimal numbers between a.b and c.d

You might now expect that ..x would match all numbers smaller or equal to x, but oddly enough this is not the case; the .. operator appears to be simply ignored when you do this. Why? I don't know, but it's probably thought up by the same logic that says ** is a wildcard for two words, *** for three words, etc, but * for one or more.
