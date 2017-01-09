---
layout: post
title: Shaking Things Up with Shake and Shakers in a Simple Haskell Project
---

{{ page.title }}
================

*8 January 2017*

[shake][shake] is a Haskell library for writing build systems to replace `make`, supporting rich and expressive project building functionality. We use `shake` in our Haskell projects to provide various simple and complex rules:

+ Formatting, linting, preprocessing, compiling source
+ Running tests
+ Installing binaries
+ Publishing libraries
+ Migrating databases
+ Building containers
+ Deploying applications
+ Running utilities

To avoid boilerplate and promote reusability among our Haskell projects, we have developed [shakers][shakers] as a Haskell library around `shake` to capture defaults and common build patterns. This post walks you through setting up `shake` in a simple Haskell project with the help of `shakers`. The complete code for the project can be found at [shakers-example][shakers-example].

[shake]:           http://hackage.haskell.org/package/shake
[shakers]:         http://hackage.haskell.org/package/shakers
[shakers-example]: https://github.com/mfine/shakers-example
