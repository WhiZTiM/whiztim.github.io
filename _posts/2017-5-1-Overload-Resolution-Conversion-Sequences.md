---
layout: post
title: Overload Resolution, Implicit Conversion Sequences
comments: true
---

#### The overloading story of `std::string` and `bool` for `"Sweet Melon"`

Every now and then we expose overloaded API's in C++. But we seldom do consider How overload resolution works. The rules are more complex than you think, even the ISO C++ standard dedicated several pages of technical text describing it's rules. However, in this article we will only consider The interaction of overload resolution with Implicit Conversion Sequences ..

Consider the following snippet:

```c++
void foo(int){ 
    std::cout << "foo(int)" << '\n;
}

void foo(bool){
    std::cout << "foo(bool)" << '\n;
}

int main(){
    foo(NULL);
    foo(nullptr);
}
```

What do you think of its output?

    foo(int)
    foo(bool)

This is probably no brainer if you have programmed in C++ using C++03 standard and you've migrated to post C++11. For a simple remainder, `NULL` was simply inherited from C, which is typically a MACRO defined as `#define NULL 0`.

So, basically the call in the `main()` boils down to (after Preprocessor pass):

```c++
int main(){
    foo(0);
    foo(nullptr);
}
```

The above, `0` as with any other "value" arabic numeral or integer is known as an *integer literal*. whose type is deduced by another set of rules as described [here](http://en.cppreference.com/w/cpp/language/integer_literal#The_type_of_the_literal)

-------------------

### The Overloading story of `std::string` and `bool`

Let's get into the main point, consider the code below:

```c++
void foo(std::string){
    std::cout << "foo(std::string)" << '\n;
}

void foo(bool){
    std::cout << "foo(bool)" << '\n;
}

int main(){
    std::string str = "Sweet Melon!";
    
    foo("Sweet Melon!");
    foo(nullptr);
    foo(str);
}
```

What do you think of it's output?:

    foo(bool)
    foo(bool)
    foo(std::string)

....WTH! Not intuitive right? It turns out that the "string" `"Sweet Melon"` is not a `std::string` object, but rather a *string literal* of type `const char[12]`. It's type is deduced by the compiler as a `const` array of 12 characters. (remember, *string literals* are **always** implicitly "Null `\0`" terminated by the compiler).

The problem we have now is ranking the type `const char[12]` against viable overloads of:

- `std::string`  (Has many Constructor overload, User Defined Type)
- `bool` ( built-in type, Non-class type, but special.)

Unfortunately, there is none that is an exact match. So the compiler considers an [implicit conversion sequence (ICS)](http://eel.is/c++draft/over.best.ics) which also includes exploration of [converting constructors](http://en.cppreference.com/w/cpp/language/converting_constructor) of the class types we are up against...  


It also turns out that there are three ordered ranks of Implicit Conversion Sequences: 

- [Standard conversion sequences](http://eel.is/c++draft/over.best.ics#over.ics.scs),
- [User-defined Conversion sequences](http://eel.is/c++draft/over.best.ics#over.ics.user), and 
- [Ellipsis Conversion Sequence](http://eel.is/c++draft/over.best.ics#over.ics.ellipsis)

And these rank in the order they appear. Note the "s" in the "sequence**s**" above.

> When comparing the basic forms of implicit conversion sequences (as defined in [[over.best.ics]](http://eel.is/c++draft/over.best.ics))
> 
> - a standard conversion sequence is a better conversion sequence than a user-defined conversion sequence or an ellipsis conversion sequence, and
> - a user-defined conversion sequence is a better conversion sequence than an ellipsis conversion sequence.

So, as the compiler's overload resolution subsystem considers ICS and tries to apply them in order they appear, the compiler is required to go through viable permutations of the above and rank them.

It turns out that the only possible ICS for `const char[12]` is "Array-to-pointer" conversion which is a *Standard Conversion Sequence*. "Array to pointer" conversion is typically known as array "decay".

-----------------

So, the compiler's new quest is to find a matching overload for `const char*` among:

- `std::string`
- `bool`

... Unfortunately, we still don't have a best viable function.

<br />

If after one *Standard Conversion Sequence* happens, and a best viable function among the overload set hasn't been produced, the compiler is required to:

1. perform a *User-Defined Conversion Sequence*, or
2. perform additional conversions, one from each of the remaining [categories](http://eel.is/c++draft/over.best.ics#tab:over.conversions) of *Standard Conversion Sequence* in order.

##### 1. Perform a *User-Defined Conversion Sequence*:

`std::string` has a constructor (simplified):

```c++
namespace std{
    template<typename CharT, 
             typename Traits = std::char_traits<CharT>,
             typename Allocator = std::allocator<CharT>>
    class basic_string{
    public:
        basic_string(const CharT*);   //Our constructor of interest
        ....
    };
    using string = basic_string<char>;
} 
```

In this parital procedure, we can then say that, for a `const char*` type, we have viable paths of:

- `std::string` from `std::string(const char*)`
- `bool` from ??

##### 2. Perform Additional Standard Conversion from proceeding Categories:
|
We can convert any pointer to `bool` regardless of its qualification.

We can then say that, for a `const char*` type, we have viable paths of:

- `std::string` from `std::string(const char*)` (User defined conversion)
- `bool` from `const char*` (standard conversion)

Implicit conversion of a pointer type to a `bool` is a *Standard Conversion Sequence* and it ranks higher than `std::string`'s converting constructor which is a *User-defined Conversion Sequence*. (Remember, anything not defined by the core language, is assumed to be user defined in C++'s pureview, even an STL type!)


---------------
Feedbacks, corrections to my mail below \\