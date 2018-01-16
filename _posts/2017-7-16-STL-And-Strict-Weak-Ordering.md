---
layout: post
title: STL and the Strict Weak Ordering
comments: true
---

When people write so called *generic* algorithms out there online, a lot of the the so-called generic fuctions uses more than one relational operator to order a sequence; This introduces unecessary strain on the clients of such code or makes some poor assumptions about the datatype we want to use.

-----------------------

# An Example of Binary Search

If we want to write a generic iterative version of binary search, that assumes the elements are sorted in a monotonically increasing order, we may write something like:

```c++
template<typename Iterator, typename T, typename Compare>
Iterator binary_search(Iterator first, Iterator last, const T& value){
    auto sentinel = last;
    while(first != last){
        auto mid = first + std::distance(first, last) / 2;
        if(*first == value)
            return first;
        if(*first <= value && value < *std::prev(mid))
            last = mid;
        else if(*mid <= value && value <= *std::prev(last))
            first = mid;
        else
            return sentinel;
    }
    return sentinel;
}
```

That seems all good. Until we realize that we want to extend this further. How about if we we want to do a binary search on a monotonically decreasing sequence (descending order)? We would have to modify our code!

Well, we should provide a flag or an `enum` you say? Yep, that will, except for one bit of trouble, we would be assuming that the types we would be doing binary search on, has defined some set of operators. If you look at the code above, you would see that we used 3 different operators in our `binary_search`

These are:

* `bool operator == (const T& x, const T& y);`
* `bool operator <= (const T& x, const T& y);`
* `bool operator < (const T& x, const T& y);`

This means, that for some custom type `ClassA`, we need to define all these three operators to use our `binary_search`. Doesn't sound like a great idea to me.

What is even worse, is that, when we want to adapt our `binary_search` to take a custom functor, we need to have three custom functors to satisfy the above. Either by making using some monkey techniques or by making it this ugly

```c++
template<
    typename Iterator,
    typename T,
    typename Equality,
    typename LessThanOrEqual,
    typename LessThan>
Iterator binary_search(Iterator first, Iterator last, const T& value,
    Equality eq, LessThanOrEqual le, LessThan lt);
```

That's a very bad API there. We can do much better.

Consider the concept of [Strict Weak Ordering](https://en.wikipedia.org/wiki/Weak_ordering#Strict_weak_orderings).

Using the strict Weak Order of **`<`** (less than operator). We can establish the following:

- `A < B;` `A` is less than `B`
- `B < A;` `B` is less than `A`
- `!(A < B);` `A` is not less than `B`. Hence, `A` is greater than or equal to `B`
- `!(B < A);` `B` is not less than `A`. Hence, `B` is greater than or equal to `A`
- `!(A < B) && !(B < A);` `A` is neither less than `B` nor vice-versa, Hence `A` is equal to `B`

--------
See how we were able to use one single operator to Cater for others?

- `A < B;` equivalent to: `A < B` ; `B > A`
- `B < A;` equivalent to: `B < A` ; `A > B`
- `!(A < B);` equivalent to: `A >= B` ; `B <= A`
- `!(B < A);` equivalent to: `B >= A` ; `A <= B`
- `!(A < B) && !(B < A);` equivalent to: `A == B`

When it comes to relative orders, C++ treats its STL comparators with the concept of Strict Weak Ordering. This makes it possible to use a single comparator for A variety of things 

Hence we can improve our code to:

```c++
template<typename Iterator, typename T, typename Compare>
Iterator binary_search(Iterator first, Iterator last, const T& value){
    auto sentinel = last;
    while(first != last){
        auto mid = first + std::distance(first, last) / 2;
        if(!(*first < value) && !(value < *first)) //They are same
            return first;
        if(!(value < *first) && value < *std::prev(mid))
            last = mid;
        else if(!(value < *mid) && !(*std::prev(last) < value))
            first = mid;
        else
            return sentinel;
    }
    return sentinel;
}
```

=============

The above code gives us opportunity to **simply** replace our use of `operator <` with a custom comparator:

A better code:

```c++
template<typename Iterator, typename T, typename Compare>
Iterator binary_search(Iterator first, Iterator last, const T& value, Compare cmp){
    auto sentinel = last;
    while(first != last){
        auto mid = first + std::distance(first, last) / 2;
        //Strict Weak Ordering
        if(!cmp(*first, value) && !cmp(value, *first))
            return first;
        if(!cmp(value, *first) && !cmp(*std::prev(mid), value))
            last = mid;
        else if(!cmp(value, *mid) && !cmp(*std::prev(last), value))
            first = mid;
        else
            return sentinel;
    }
    return sentinel;
}

//Default option
template<typename Iterator, typename T>
Iterator binary_search(Iterator first, Iterator last, const T& value){
    return (binary_search)(first, last, value, std::less<T>()); 
	// ^^^^^^^^^^^^^^^ I added an extra paranthesis to prevent ADL
}
```

You can double check this implementation with the C++ STL's `std::binary_search`.

==============

Timothy