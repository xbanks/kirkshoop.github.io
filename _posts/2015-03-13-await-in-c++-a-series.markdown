---
layout: post
title:  "await in c++ - a series"
date:   2015-03-13 3:09:00
categories: [async, rxcpp, await, c++,]
---

## Introduction
I have seen a few articles that introduce the [await proposal](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4134.pdf) as implemented in [Visual Studio 2015 Preview](https://www.visualstudio.com/en-us/news/vs2015-vs.aspx). In this series I will share my exploration of the await feature by showing how I built `async_generator<T>` and some algorithms and adaptors I built to use it. To skip the exposition, go straight to the code on [github](https://github.com/kirkshoop/await).

## A Look Ahead
```cpp
auto test = []() -> std::future<void> {
    for __await(auto&& rt :
        // tick every second
        as::schedule_periodically(clk::now() + 1s, 1s, 
            [](int64_t tick) {return tick; }) |
        // ignore every other tick
        ao::filter([](int64_t t) { return (t % 2) == 0; }) |
        // stop after 10 ticks
        ao::take(10)) {
        // the loop body will be run every other second - 10 times
        std::cout << "for " << std::this_thread::get_id()
            << " - " << rt
            << std::endl;
    }
};
// this function will return immediately before any text is printed
auto done = test();
// block until the loop has finished
done.get();
```
This looks similar to [Eric Niebler's](https://twitter.com/ericniebler) Range proposal ([GitHub](https://github.com/ericniebler/range-v3), [Blog](http://ericniebler.com/)), but these are async ranges. Not only are the types involved different, but also the for loop and the algorithms. Coordinating many Ranges from many threads over time has additional complexity and different algorithms. The [ReactiveExtentions](http://reactivex.io/languages.html) family of libraries provide a lot of algorithms useful for async Ranges. The [RxMarbles](http://rxmarbles.com/) site has live diagrams for many of the algorithms. [rxcpp](https://github.com/Reactive-Extensions/RxCpp) implements some of these algorithms in C++ without await and the series will produce adaptors that allow async Ranges and rxcpp Observables to be interchanged.

## Types in Time
Time is what the await proposal introduces into the C++ language.
This is a table of types that represent each combination of values and time.

 | Value | Sequence 
--------|----------|------------- 
past | `T` | `vector<T>`
lazy | `[]() -> T { . . . }` | `generator<T>`
later | `future<T>` | `async_generator<T>`

* past - A value has already been produced before the caller asks for it.
* lazy -  A value is produced when asked for by a caller.
* later - When a value is produced the caller is resumed.

The three that I will explore in the context of await are `future<T>`, `generator<T>` and `async_generator<T>`

## Resources
* [Gor Nishanov ](https://twitter.com/gornishanov) made a great presentation ([PDF](https://github.com/CppCon/CppCon2014/blob/master/Presentations/await%202.0%20-%20Stackless%20Resumable%20Functions/await%202.0%20-%20Stackless%20Resumable%20Functions%20-%20Gor%20Nishanov%20-%20CppCon%202014.pdf), [YouTube](https://www.youtube.com/watch?v=KUhSjfSbINE)) of the design, implementation and sample usage at [CPPCON 2014](http://cppcon.org/)
* [Visual Studio 2015 Preview](https://www.visualstudio.com/en-us/news/vs2015-vs.aspx) implements the await proposal (in these articles I am using CTP6).
* [Visual Studio Team Blog Post](http://blogs.msdn.com/b/vcblog/archive/2014/11/12/resumable-functions-in-c.aspx) introducing the await implementation.
* [Paolo Severini](https://paoloseverini.wordpress.com/2015/03/06/stackless-coroutines-with-vs2015/)  introducing await.
