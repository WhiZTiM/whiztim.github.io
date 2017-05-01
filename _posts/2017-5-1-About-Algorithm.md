---
layout: post
title: Just a Test!
comments: true
--- 
 
So, today, we want to understand the basic concept of `iterator_traits`..
I am kidding... I am only testing out the capabilities of Jekyll

C++ has a way of known what category an `iterator` belongs to. There is a class thus defined as:
```c++
    namespace std{
        template<typename IteratorType>
        struct iterator_traits{
            //Empty body for base case...
            // You should specialize for your type
        };

        //Specialization for all pointer types:
        template<typename IteratorTypePtr>
        struct iterator_traits<IteratorTypePtr*>{
            using difference_type = std::ptrdiff_t;
            using value_type = T;
            using iterator_category = std::random_access_iterator_tag;
        };
    }
```
![_config.yml]({{ site.baseurl }}/images/config.png)

The easiest way to make your first post is to edit this one. Go into /_posts/ and update the Hello World markdown file. For more instructions head over to the [Jekyll Now repository](https://github.com/barryclark/jekyll-now) on GitHub.