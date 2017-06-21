---
layout: post
title: Layman's explanation to Functions and Stack
comments: true
---

In this post, I use everyday terminology to explain how functions work, and what a call-stack is.

-----------------------

## FUNCTIONS and STACK

*Note that, I posted the original version of this airtcle online over 2 years ago*

### A small picture

Consider a man named Musa who has to execute some tasks; those tasks are fully wrapped in a box. Think of each box like a portable version of your local Grinding Machines/Blender where you grind Tomatoes, maize, and so on; You know there are different kinds, sizes, and types of grinding Machines, so are there different type of boxes :-).

Musa is in a warehose full of Boxes; Now There is long enclosed rail with only one end accessible to Musa where he can push Boxes to, if he wishes; Each box cannot work on its own without a power supply; And the only power supply is at the end of the rail, where Musa sits.

### Relating the above.....

Your program starts from the *function* `main()`; in this case, you have given Musa a box to process;
Musa, places the Box on the end of the rail, plugs it and starts processing it. While processing the given items in the box, he reaches a point where he cannot proceed without the result of another function(Box); He then suspends the current items he is processing, in an attempt to obtain that very function (Box);

##### Calling a function

Musa unplugs `main()` and pushes it further into the rail... then gets up and walks through the warehouse in search of the Box he needs. He retrieves it and brings it to his desk; Lays it on the end of the rail *(Remember, "`main()`" was pushed further into the rail)*. Musa plugs this newly obtained function (Box) into the power supply, and if the box requires inputs (argument), Musa passes the inputs (say, Tomatoes) into Box and starts processing....

If He reaches a point where he needs, another function to proceed, ...then the entire process is repeated until the functions starts returning (i.e a function executed to completion)...

##### Returning from a function

When a function is returning... Musa collects the results(lets say result of grinding tomatoes and onions); then He cleans up that machine(Box). Unplugs it and returns it to the warehouse floor. He then rolls the rail forward until the last Box He was using pops out. He connects it, and continues from where He stopped, this time with the results of the previous function. Hence, he can proceed. :-)

##### Say what?

Now, that my friend, is the simplified behavior of a **function** and a **call-stack**. :-)!

- The rail, is your **stack**, the boxes are your **functions**!
- When we are returning from a function(cleaning up the machines), Destructors of objects are called. (In C++ and D).

We can simply view a "stack" as part of your RAM that is grows further and shrinks with respect to function calls;

### Extras
- There is something called **stack unwinding**. This simply means the minimum work you must do to remove boxes from the rail.
- There is something called **stack overflow**. This simply means having too much items on your rail to extents that it's full, therefore, preventing you to push more boxes into it.
- When a program is *multi-threaded*, it means, you have multiple *Threads* (box rails) and *Executors*(Musa)
- Functions may be inlined, meaning embedding the processing operations of a box in a *calling box*, thus Musa, doesn't have the burden of going to look for it. (His current Box does it.)
- Every program starts with a function. (some may be in a global disguise, like scripting languages)

### Technical
- local variables declared within functions are fast because, accessing them inside that function is addressed in a very simple fashion; basically, relative offsets;

The CPU simply reads from direct offsets in order to retrieve the value you want to read;
Interestingly, (most at times) the data doesn't need to be re-read from RAM, because while the function is fetched, the entire function block will (almost always) be fetched from RAM into the CPU's memory(cache).

- Years ago, One key difference that made C++ faster than Java(not always); and Java faster than Python(not always) was in the way they resolve their functions;

 - C++ knows exactly where the function is at every call site. and calls them directly (non virtual functions);
 - Java looks the function up in a Virtual table (jumps to an offset, to find the exact position of the function);
 - Python looks up a dictionary, does some calculations, and jumps to the function's call site

Note: Today, the compilers and interpreters of those other languages are heavily optimized to be able to make direct calls, hence making certain Java programs as fast as its C++ equivalent.

If You are REALLY interested in the ADVANCED stuff, head to http://en.wikipedia.org/wiki/Call_stack ...start from there and dig in deeper.

### ==============

From a Machine's point of view, a function is basically any well defined cluster of (instruction and data). What I mean is, A function is an executable region in memory.

==============

Timothy