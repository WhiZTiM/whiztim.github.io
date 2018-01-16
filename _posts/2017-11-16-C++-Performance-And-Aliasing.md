---
layout: post
title: Performance in C++ - As local as possible
comments: true
---

C++'s selling point is performance and versatility, however, its versatility makes it possible to write code that doesn't live up to its expectations.

-----------------------

# A premier

Lets get one thing straight, you care about performance. There's no doubt, else you most likely wouldn't be using C++, neither would you be reading this post.

One great thing about most programming languages is the concept of a *pointer*, *reference*, *proxy*, or whatever the kommitee or [BDFL](https://en.wikipedia.org/wiki/Benevolent_dictator_for_life) of the language calls it. 

For C++, we have both *pointers* and *references*. There are subtle differences on how they work in the sense that a reference must be bound to an object at creation; and references can extend the lifetime of an object. These two tools (pointers and references) of C++ have a *potential* behavior to slow down your code.

We typically use some *cv* qualified references or pointers in our function arguments if we we do not want to make expensive copies of our argument, or non-*cv* qualified variants if we want to modify the passed argument. For example:

```c++
void do_something(int& data, const int& id){
    while(someWork())
        data += id;
}
```

for the purpose of this article we will use pointers.

----------------

Consider a structure known as `Data` defined below:

```c++
struct Data{
    int* value;
};
```

Let the member function simply be a pointer that points to an integer somewhere in memory. (A non-owning pointer in this case). Let us have some work class `Wx` with a member function `Wx::work(int*, Data*)` as defined below:

```c++
struct Wx{
    void work(int* values, Data* d){
        for(int i = 0; i < 16; i++)
            *d->value = *d->value + values[i];
    }
};
```

`Wx::work(int*, Data*)` simply does adds up an array of integers pointed by `values` from index *`[0 to 16)`*.
To what extents can an a great optimizing compiler compile this code to modern hardware (say, Intel Broadwell series and beyond)?

Lets deliver a test code to the compiler in ways that hides information about where both `values` and `d` points to:

```c++
Wx e1;

void f(int* a, Data* d)
{
    e1.work(a, d);
}
```

When compiled using, GCC version 8.0.1 on `-O3` flag: the assembly produced is:

```java
  mov rdx, QWORD PTR [rdx]
  mov eax, DWORD PTR [rdi]
  add eax, DWORD PTR [rdx]
  mov DWORD PTR [rdx], eax
  add eax, DWORD PTR [rdi+4]
  mov DWORD PTR [rdx], eax
  add eax, DWORD PTR [rdi+8]
  mov DWORD PTR [rdx], eax
  add eax, DWORD PTR [rdi+12]
  mov DWORD PTR [rdx], eax
  add eax, DWORD PTR [rdi+16]
  mov DWORD PTR [rdx], eax
  add eax, DWORD PTR [rdi+20]
  mov DWORD PTR [rdx], eax
  add eax, DWORD PTR [rdi+24]
  mov DWORD PTR [rdx], eax
  add eax, DWORD PTR [rdi+28]
  mov DWORD PTR [rdx], eax
  add eax, DWORD PTR [rdi+32]
  mov DWORD PTR [rdx], eax
  add eax, DWORD PTR [rdi+36]
  mov DWORD PTR [rdx], eax
  add eax, DWORD PTR [rdi+40]
  mov DWORD PTR [rdx], eax
  add eax, DWORD PTR [rdi+44]
  mov DWORD PTR [rdx], eax
  add eax, DWORD PTR [rdi+48]
  mov DWORD PTR [rdx], eax
  add eax, DWORD PTR [rdi+52]
  mov DWORD PTR [rdx], eax
  add eax, DWORD PTR [rdi+56]
  mov DWORD PTR [rdx], eax
  add eax, DWORD PTR [rdi+60]
  mov DWORD PTR [rdx], eax
```

Well, this looks great, because we can see the compiler did some [*loop unrolling*](https://en.wikipedia.org/wiki/Loop_unrolling) optimization. And we may think life is good. We've got a smart compiler. However, this isn't the best code that can be generated for modern CPUs. Considering the fact that we do not see any [vector instructions](https://en.wikipedia.org/wiki/Automatic_vectorization) issued in the assembly code above. yet again, the instructions generated seem to be dereferencing pointer address always (ex. `DWORD PTR[rdx]`).

The reason why GCC cannot issue vector instructions for the code above can be derived by looking at the code one more time:

```c++
struct Wx{
    void work(int* values, Data* d){
        for(int i = 0; i < 16; i++)
            *d->value = *d->value + values[i];
    }
};
```

We are more interested in this line:


```c++
    for(int i = 0; i < 16; i++)
        *d->value = *d->value + values[i];
```

Since we know `i` increases from *`[0, 16)`*, and the expression `values[i]` isn't undefined behavior, we can conclude that the memory of the elements at any `values[i]` and `values[i+1]` (where `i+1 < 16`) do not overlap. That conclusion makes it possible to [vectorize](https://en.wikipedia.org/wiki/Automatic_vectorization) the addition of non-overlapping elements then add the final results to `*d->value`; I mean hypothetically, transform the above code into:

```c++
    auto ans = vector_add(values, 0, 16); //move elements to vector registers and add all
    *d->value = ans;                      //assign the results.
```

However, the above code is not a valid transformation of the former code - because `d->value` may infact be pointing an element within `values` *`[0, 16)`*. meaning that each assignment of `d->value` may have subsequent side effect on an element in the rest of the array.

-----------------------------------

### What can we do if we know they'll never overlap?

Consider transforming `Wx` from this:

```c++
struct Wx{
    void work(int* values, Data* d){
        for(int i = 0; i < 16; i++)
            *d->value = *d->value + values[i];
    }
};
```

to another struct `Wy`:

```c++
struct Wy{
    void work(int* values, Data* d){
        int val = *d->value;
        for(int i = 0; i < 16; i++)
            val = val + values[i];
        *d->value = val;
    }
};
```

All we did here is to make a local copy of `*d->value` and have our loop around it. When we are done, we assign it back to `*d->value`; From this code, the compiler can tell that we can add `values[0 to 16)` without any side effects, then finally assign the result to `val`; which we in turn assign to `*d->value`; In fact `val` isn't needed here. We can go do our vectorization on adding `values[0 to 16)` first, then assigning the result to `*d->value`.

```c++
    auto ans = vector_add(values, 0, 16); //move elements to vector registers and add all
    *d->value = ans;                      //assign the results.
```

Now the above code is a legal transformation of the former. When we compiler our example code with our latest changes, GCC 8.0.1 at `-O3` yeilds an assembly of:

```java
  movdqu xmm0, XMMWORD PTR [rsi]
  movdqu xmm2, XMMWORD PTR [rsi+16]
  movdqu xmm1, XMMWORD PTR [rsi+32]
  paddd xmm0, xmm2
  paddd xmm1, xmm0
  movdqu xmm0, XMMWORD PTR [rsi+48]
  paddd xmm0, xmm1
  movdqa xmm1, xmm0
  psrldq xmm1, 8
  paddd xmm0, xmm1
  movdqa xmm1, xmm0
  psrldq xmm1, 4
  paddd xmm0, xmm1
  movd ecx, xmm0
  add eax, ecx
  mov DWORD PTR [rdx], eax
```

Now, we have vectorization :-).

## Caveats:

That we have smaller assembly generated doesn't necessarily mean faster execution. Some instructions spans several multiple CPU cycles more than others. Others may introduce some [stall](https://en.wikipedia.org/wiki/Bubble_(computing)). Hence **always measure** your code even after viewing generated assembly.

==============

Timothy