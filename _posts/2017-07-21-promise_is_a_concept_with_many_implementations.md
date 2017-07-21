---
title: Promise is a concept with many implementations
layout: post
date: 2017-07-21 09:35 PDT

---

Perhaps one of the reasons that there are so many `promise` proposals is that each is an implementation of the same _Concepts_ with different tradeoffs.

> A `promise` will transport a single value across time.

To provide motivation for a _Concept_ based approach to `promise` standardization, I will compare and contrast `promise` with the STL.

Multiple implementations of _Containers_ and _Iterators_ were included in the Standard. This invocation of the [Rule of Twos](https://medium.com/capital-one-developers/rule-of-twos-and-microservice-architecture-3f57db7f6896), resulted in robust and stable concepts that have been used to successfully implement additional containers, iterators and algorithms outside of the Standard. 

## implementations

The different `promise` implementations are similar to `vector`, `list` and `string`. 
- some will block with `get()`
- some will poll on a thread and invoke the `then()` callback
- some will invoke the `then()` callback directly from the context of the producer of the value

Perhaps `string` is the most unfortunate comparison, since like `string`, the current `promise` proposals embed some algorithms into the implementation. The current `promise` implementations also embed execution context support into the implementation.

Specifying _Concepts_ instead of implementations will naturally extract algorithms and execution context support from the `promise` implementations so that these can be shared across multiple `promise` implementations.

## algorithms

In the current proposals some algorithms are embedded into the promise implementations (eg. `then()`) and others (eg. `when_any()` & `when_all()`) are external.

In the current proposals `then()` is overloaded to mean both `transform()` and `transform_flatten()` (aka `flat_map()`, `map() | merge()`, etc..)

With the right _Concepts_, `then()`, `get()`, `when_any()` and `when_all()` will all be implementations of the same _Concepts_. 

With the right _Concepts_, additional algorithms - `delay()`, `produce_on()`, `consume_on()`, `finally()`, `error()`, `retry()`, `tap()` will be implemented.

With the right _Concepts_, users can build any algorithm that they wish.

## execution context

There are two places in a promise where control over the execution context is desired. One is the producer of the value, the other is the consumer of the value.

With the right _Concepts_ for promise, these are just additional algorithms.

The `produce_on(Context)` algorithm is a promise implementation that queues the producer callback onto the provided context.

The `consume_on(Context)` algorithm is a promise implementation that queues the `then()` callback onto the provided context.

The true power of moving these out of the `promise` implementation is that overloads of the algorithms for different execution contexts can coexist and compose. 

 > I intentionally avoided using _Executor_ since with _Concepts_ there would be no coupling between a `promise` implementation and whatever _Executor_ turns out to be. Overloads of the above algorithms can be added over time as _Executor_ evolves.

## negative space

The negative space between various _Container_ implementations and the set of useful algorithms, exposed the shape of the several _Iterator_ categories. Similar to the _Rule of Twos_ this increases confidence in the completeness and correctness of the _Concepts_.

A set of _Concepts_ that fill the negative space between multiple `promise` implementations and the set of useful algorithms, will not look the same as the current proposals. However, all the current proposals could be implemented with these _Concepts_

## _Single_

_Single_ and its companions are _Concepts_ that will transport a value across time. 

_Single_ and its companions fill the negative space between producers, consumers and the set of useful algorithms.

_Single_ is used both to produce the value and to consume the value. 
 - A _Single_ is passed to the producer which will call `value()` or `error()`
 - A _Single_ is created by `then()` that calls the lambda when `value()` is called.

_Lifetime_ is a _Concept_ that will scope the data and state across time.

_SingleSubscription_ is a _Concept_ that binds a consumer to the _Lifetime_ scope.

_SingleDeferred_ is a _Concept_ that binds the producer and the consumer and the scope, invokes the producer, transports the value from the producer to the consumer and ends the scope.

> I am still trying to correctly describe these using concepts-lite. I need a way to allow `single::value()` and `single::error()` to leave their arguments unconstrained.

```cpp
template <typename L>
concept bool Lifetime() {
    return requires(const L& l) {
        { l.is_stopped() } -> bool;
        { l.stop() } -> void;
    };
}

template <typename S>
concept bool Single() {
    return requires(const S& s) {
        { s.value(auto) } -> void;
        { s.error(auto) } -> void;
    };
}

template <typename S>
concept bool SingleSubscription() {
    return requires(const S& s) {
        requires Lifetime<s.lifetime>;
        requires Single<s.destination>;
    };
}

template <typename S, typename D>
concept bool SingleDeferred() {
    return requires(const S& s, const D& d) {
        requires Lifetime<s.subscribe(d)>;
    };
}

struct alifetime
{
    bool is_stopped() const;
    void stop() const;
};
static_assert(Lifetime<alifetime>(), "not a Lifetime");

struct asingle
{
    template<typename T>
    void value(T&& t) const;

    template<typename E>
    void error(E&& e) const;
};
static_assert(Single<asingle>(), "not a Single");

struct asinglesubscription
{
    alifetime lifetime;
    asingle destination;
};
static_assert(SingleSubscription<asinglesubscription>(), "not a SingleSubscription");

struct asingledeferred
{
    Lifetime subscribe(SingleSubscription) const;
};
static_assert(SingleDeferred<asingledeferred>(), "not a SingleDeferred");

```

## still to come..

I hope this whets the appetite for upcoming explorations of how to implement a `promise` and the `then()`, `produce_on()` and `consume_on()` algorithms using these concepts.

In the fullness of time, the hint in the name _Single_ will be confirmed by the introduction of _Legion_

> My name is Legion .. because we are many

## references

- [Single](http://reactivex.io/RxJava/2.x/javadoc/io/reactivex/Single.html) as implemented in Java
- [P0701r0](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0701r0.html) - Back to the std2::future
- [P0676R0](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0676r0.pdf) - Towards a Good Future

## acknowledgements

This is informed by many discussions and prior art in the reactivex and C++ communities, as well as conversations with; _Bryce Lelbach_, _David Sankel_, _Gor Nishanov_ and many others..
