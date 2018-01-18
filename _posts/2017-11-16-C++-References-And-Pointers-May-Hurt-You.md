---
layout: post
title: C++ - References and Pointers may impede optimizers
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

For example, let us say `values` points to an array of integers starting out at address `0xb344e1`:

    Index: 0  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15 ..
    Value: 1  2  3  4  5  6  7  8  9  8  7  6  5  4  3  2  ..

And let us say `d->value` points to an integer outside the above array: say at address `0xcc2001`, and the value of the integer there is `9`; The above loop is simply saying add `9 + (1 + 2 + 3 ...)` (to the 15th index of `value`), then store in where ever `d->value` points to.

So for the example above, *after* each iteration of `i`:

    |  i  | *d->value | values[i] |              values              |
    ==================================================================
    |prior|      9    |    N/A    | 1 2 3 4 5 6 7 8 9 8 7 6 5 4 3 2  |
    ------------------------------------------------------------------
    |  0  |     10    |     1     | 1 2 3 4 5 6 7 8 9 8 7 6 5 4 3 2  |
    |  1  |     12    |     2     | 1 2 3 4 5 6 7 8 9 8 7 6 5 4 3 2  |
    |  2  |     15    |     3     | 1 2 3 4 5 6 7 8 9 8 7 6 5 4 3 2  |
    |  3  |     19    |     4     | 1 2 3 4 5 6 7 8 9 8 7 6 5 4 3 2  |
    |  4  |     24    |     5     | 1 2 3 4 5 6 7 8 9 8 7 6 5 4 3 2  |
    |  5  |     30    |     6     | 1 2 3 4 5 6 7 8 9 8 7 6 5 4 3 2  |
    |  6  |     37    |     7     | 1 2 3 4 5 6 7 8 9 8 7 6 5 4 3 2  |
    |  7  |     45    |     8     | 1 2 3 4 5 6 7 8 9 8 7 6 5 4 3 2  |
    |  8  |     54    |     9     | 1 2 3 4 5 6 7 8 9 8 7 6 5 4 3 2  |
    |  9  |     62    |     8     | 1 2 3 4 5 6 7 8 9 8 7 6 5 4 3 2  |
    | 10  |     69    |     7     | 1 2 3 4 5 6 7 8 9 8 7 6 5 4 3 2  |
    | 11  |     75    |     6     | 1 2 3 4 5 6 7 8 9 8 7 6 5 4 3 2  |
    | 12  |     80    |     5     | 1 2 3 4 5 6 7 8 9 8 7 6 5 4 3 2  |
    | 13  |     84    |     4     | 1 2 3 4 5 6 7 8 9 8 7 6 5 4 3 2  |
    | 14  |     87    |     3     | 1 2 3 4 5 6 7 8 9 8 7 6 5 4 3 2  |
    | 15  |     89    |     2     | 1 2 3 4 5 6 7 8 9 8 7 6 5 4 3 2  |

As we can see, `values[i]` *for* `0 <= i < 16` doesn't change thus, we can do a [vector add](https://en.wikipedia.org/wiki/Automatic_vectorization) of `values[i]` *for* `0 <= i < 16` - summing it up manually yields `80`, adding that to `*d->value`(`9`), we have `89`;

So, by [vectorization](https://en.wikipedia.org/wiki/Automatic_vectorization), we look at this line fulfilling the underlisted conditions:

```c++
    for(int i = 0; i < 16; i++)
        *d->value = *d->value + values[i];
```

1. Since we know `i` increases from *`[0, 16)`*, 
  * and we assume the expression `values[i]` doesn't invoke undefined behavior for any `0 <= i < 16`,
2. we can conclude that the memory of the elements at any `values[i]` and `values[i+1]` (where `i+1 < 16`) do not overlap. 
3. then it is possible to [vectorize](https://en.wikipedia.org/wiki/Automatic_vectorization)(assuming suitable alignments) the addition of non-overlapping elements then add the final results to `*d->value`; I mean hypothetically, transform the above code into:

```c++
    auto ans = vector_add(values, 0, 16); //move elements to vector registers and add all
    *d->value = ans;                      //assign the results.
```

However, the above code is not a valid transformation of the former code - because `d->value` may infact be pointing an element within `values` *`[0, 16)`*. meaning that each assignment of `d->value` may have subsequent side effect on an element in the rest of the array.

As an example:

For example, let us say `values` points to an array of integers starting out at address `0xb344e1`:

    Index: 0  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15 ..
    Value: 1  2  3  4  5  6  7  8  9  8  7  6  5  4  3  2  ..

And let us say `d->value` point to an integer inside the above array: say at address `0xb344e1 + sizeof(int) * 9` (the *8th* index whose value is `9`); So for the example above, *after* each iteration of `i`:

    |  i  | *d->value | values[i] |              values               |
    ===================================================================
    |prior|      9    |    N/A    | 1 2 3 4 5 6 7 8 9 8 7 6 5 4 3 2   |
    -------------------------------------------------------------------
    |  0  |     10    |     1     | 1 2 3 4 5 6 7 8 10 8 7 6 5 4 3 2  |
    |  1  |     12    |     2     | 1 2 3 4 5 6 7 8 12 8 7 6 5 4 3 2  |
    |  2  |     15    |     3     | 1 2 3 4 5 6 7 8 15 8 7 6 5 4 3 2  |
    |  3  |     19    |     4     | 1 2 3 4 5 6 7 8 19 8 7 6 5 4 3 2  |
    |  4  |     24    |     5     | 1 2 3 4 5 6 7 8 24 8 7 6 5 4 3 2  |
    |  5  |     30    |     6     | 1 2 3 4 5 6 7 8 30 8 7 6 5 4 3 2  |
    |  6  |     37    |     7     | 1 2 3 4 5 6 7 8 37 8 7 6 5 4 3 2  |
    |  7  |     45    |     8     | 1 2 3 4 5 6 7 8 45 8 7 6 5 4 3 2  |
    |  8  |     54    |     9     | 1 2 3 4 5 6 7 8 54 8 7 6 5 4 3 2  |
    |  9  |    108    |     8     | 1 2 3 4 5 6 7 8 108 8 7 6 5 4 3 2 |
    | 10  |    115    |     7     | 1 2 3 4 5 6 7 8 115 8 7 6 5 4 3 2 |
    | 11  |    121    |     6     | 1 2 3 4 5 6 7 8 121 8 7 6 5 4 3 2 |
    | 12  |    126    |     5     | 1 2 3 4 5 6 7 8 126 8 7 6 5 4 3 2 |
    | 13  |    130    |     4     | 1 2 3 4 5 6 7 8 130 8 7 6 5 4 3 2 |
    | 14  |    133    |     3     | 1 2 3 4 5 6 7 8 133 8 7 6 5 4 3 2 |
    | 15  |    135    |     2     | 1 2 3 4 5 6 7 8 135 8 7 6 5 4 3 2 |

As you can see, the results are different. This happens because each modification to `*d->value` affects some item in `values[i]`. Maybe in future, experts can come up with ways to improve opportunities for vectorizing this sorta thing.

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

Now the above code is a legal transformation of the former. When we compiler our example code with our latest changes, GCC 8.0.1 at `-O3` yields an assembly of:

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

## Take Home Lesson:

- Most optimizations depends on how much the compiler can [proof](https://en.wikipedia.org/wiki/Mathematical_proof) that memory does not overlap and that side-effects are isolated.
- Don't help your compiler, but understand that pointers and references are harder for the optimizer to proof.
- Don't help the compiler, optimizers may go up the call graph to proof these things, my example above deliberately prevents that.
- Though we didn't talk much about [aliasing](https://en.wikipedia.org/wiki/Pointer_aliasing), *cv-qualified* `char*` or `std::byte*` are much harder to proof - because they can alias anything, Once you pass a such type to a function and you are equally manipulating some other proxy (whose lifetime predates and continues after the function call), some optimizations on such objects are typically impeded.

==============

Timothy