---
layout: post
title: Keep This In Mind Before Speeding On The Road
comments: true
---

You are much more likely to die in a vehicle speeding at 140Km/hr than you are at 100Km/hr.

-----------------------

## The Need for Speed

There's always a need for speeding on the streets


## Take Home Lesson:

- Most optimizations depends on how much the compiler can [proof](https://en.wikipedia.org/wiki/Mathematical_proof) that memory does not overlap and that side-effects are isolated.
- Don't help your compiler, but understand that pointers and references are harder for the optimizer to proof.
- Don't help the compiler, optimizers may go up the call graph to proof these things, my example above deliberately prevents that.
- Though we didn't talk much about [aliasing](https://en.wikipedia.org/wiki/Pointer_aliasing), *cv-qualified* `char*` or `std::byte*` are much harder to proof - because they can alias anything, Once you pass a such type to a function and you are equally manipulating some other proxy (whose lifetime predates and continues after the function call), some optimizations on such objects are typically impeded.

==============

Timothy