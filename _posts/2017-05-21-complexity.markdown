---
layout: post
title:  "Hello!"
date:   2017-05-21 10:18:00
categories: Rails
---

I've been working on various Rails code bases over the last few years and every so often jot down notes about lessons I've learned along the way. One of my favorite experiences is coming across a piece of code and thinking "*Who wrote this hot garbage?*" ... and then doing a `git-blame` and realizing it was me, 12 months ago. When this happens to me, I don't feel shame, I feel pride because it's tangible evidence that I'm growing.

Most of the things I'll be writing about you may already know, but I believe they're worth writing about because to me they were decidedly **non-obvious** to me at the time. While all these articles will be about relatively specific code patterns in Rails, they all are drawn from these general ideas that I've internalized over time:

1. ***A code base should always be evaluated on the the ability for humans to be able to understand it, debug it, reason about it, and improve it.***<br/>
   Writing clever and even just "efficient" code is great until the second you ship it and someone else has to figure out what's going on.

2. ***Code complexity is hard to define.***<br/>
   *"Just stop making your code complicated, duh!"*  The issue with this is that code complexity is frequently confused with code *volume*. Patterns that reduce complexity in smaller apps add a ton of it when the app gets large. Speaking of patterns ...

3. ***The worst code I've written is the result of following some absolute rule I've convinced myself I should be following.***<br/>
   Don't blindly follow any patterns and practices handed down from above without evaluating them for your current sitution.

Let's dive in, starting with *Callbacks*.
