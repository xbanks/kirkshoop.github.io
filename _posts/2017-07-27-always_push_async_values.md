---
title: always push async values
layout: post
date: 2017-07-27
---

The goal is of this post is to agree that; _Single_, `future.then()` and yes, even callbacks, all push async values from a producer to a consumer. After that, the claim - _always push async values_ - should seem rather humdrum.

> This is motivated by the [discussion](https://gitter.im/kirkshoop/kirkshoop.github.io) with __Sean Parent__ about [promise is a concept with many implementations](https://kirkshoop.github.io/2017/07/21/promise_is_a_concept_with_many_implementations.html)

In the post I teased a set of concepts (_Single_) that represent all the implementations of a promise.

The discussion exposed that some more groundwork was needed to connect and contrast _Single_ with other _Concepts_ and libraries.

## Single

For reference:

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

## Push and Pull

push models call a function on the consumer passing the value as a parameter. _Single_, _Legion_ and _Transducer_ are push models. every future/promise with `then()` is also a push model. even callbacks are a push model.

pull models return a value from a function on the producer. _Range_ and _Iterator_ are pull models.

All the push and pull models above are modeling an iteration over `0..N` values. All the push and pull models above provide a surface that supports the application of algorithms to arbitrary values (e.g. POD) 

The async property model, that __Sean Parent__ describes in this [paper](http://sean-parent.stlab.cc/papers-and-presentations#title-generating-reactive-programs-for-graphical-user-interfaces-from-multi-way-dataflow-constraint-systems), appears to model a graph of dynamic values (they change over time). This seems similar to the FRP model of _Behaviors_, which is a pull model.

As an analogy - If a system was built with the async property model the result would be like a DOM.
If the same system was built with a push model I would expect the result to be like a V-DOM (React, Cycle.js) or direct-mode (ImGui).

> Personal Note: I prefer to structure operations on values as algorithms iterating over values rather than methods on values (e.g. a DirectXDraw algorithm passed a graph rather than a DirectXDraw method on a graph or an asPng algorithm passed a graph instead of an asPng method on a graph). the algorithm model also makes better extensibility tradeoffs IMO. Thus I prefer an async model for iteration over an async property model.

## Push is the mathmatical dual of Pull

This has been explored repeatedly in the reactivex community. _Subject/Observer is Dual to Iterator_ by __Erik Meijer__ 
([PDF](http://csl.stanford.edu/~christos/pldi2010.fit/meijer.duality.pdf))  states that the math exists and in this [video](https://channel9.msdn.com/Events/Lang-NEXT/Lang-NEXT-2014/Keynote-Duality), __Erik__ walks through the math for _Enumerable_ and _Observable_. this [post](http://www.lirico.co.uk/tag/reactivex.html) seems to do a good job showing the steps from _Iterable_ to _Observable_ in python. 

Here, I will show the same progression for _Single_. 

Without getting hung up on the math, the intent is to show that _Single_ is the inverse of a function returning a value.

> to focus on the arrows better, the math will
> - omit the concepts for lifetime scope and cancellation.
> - assume that `producer()` & `subscribe()` do not throw. 

| notation | pull value | push value |
| :--- | :--- | :--- |
| usage example | `try {` <br/> `auto v = producer.make().get();` <br/> `} catch (exception e) {}` | `producer.subscribe(` <br/> `[](auto v){}, [](auto e){});`
| __step 1__ <br/>reverse arrows | `() -> ( ` <br/> `() -> (value | error) ` <br/> `)` | `() <- ( ` <br/> `() <- (value | error) ` <br/> `)` |
| __step 2__ <br/>rewrite with right-arrows | `() -> ( ` <br/> `() -> (value | error) ` <br/> `)` | `( ` <br/> `(value | error) -> () ` <br/> `) -> ()` |
| define <br/>__producer__ | `struct producer { ` <br/> `result make(); ` <br/> `}; ` <br/> `struct result { ` <br/> `value get(); ` <br/> `};` | `struct single_deferred {` <br/> `void subscribe(single);` <br/> `};` |
| define <br/>__consumer__ | `_` | `struct single {` <br/> `void next(auto); ` <br/> `void error(auto);` <br/> `};` |

> the `producer.make().get()` pattern looks odd here, but it replicates `*producer.begin()` for one value instead of many. <br/>`make()` returns a `result` that represents both the value and the error. 

this table demonstrates that _Single_ and a _Result_, both represent the same _Concept_ with the roles of consumer and producer reversed. _Result_ is implemented on the producer. _Single_ is implemented on the consumer.

## _Iterator_ _Concepts_

The properties of _Iterator_ in C++ are well known. Since _Single_ and _Legion_ are the same as _Iterator_ with the arrows reversed how to the different _Iterator_ _Concepts_ apply to _Single_ and _Legion_?

| _Iterator_ | applicability to _Single_ & _Legion_ |
| :--- | :--- |
| _RandomAccessIterator_ | indexing support could be added to the push model _Concepts_. However, what does it mean to random-access values distributed in time? An async producer would never be able to implement random-access. |
| _BidirectionalIterator_ | decrement support could be added to the push model _Concepts_. However, decrement generally requires accumulation of previously produced values and is better represented by an _Iterator_ on the _Container_ that has stored the accumulated values. __Note:__ use ranges TS for values distributed in space.  |
| _ForwardIterator_ | this is supported. this is almost the same as a COLD producer. each call to `subscribe()` calls the producer to create a new result. |
| _InputIterator_ | this is supported. this is almost the same as a HOT producer. the producer is run once, each call to `subscribe()` copies the result. |
| _OutputIterator_ | this is supported. The other supported _Iterator_ are supported by _SingleDeferred_ (producers). This one is supported by _Single_ (consumers). `subscribe()` can be thought of as binding an input to an output _Iterator_ with `std::copy(producer.begin(), producer.end(), consumer.begin())`.

<br/>

## Push and Pull for asynchronous producers

I reproduced this table from slides made by __Erik Meijer__ ([video](https://channel9.msdn.com/Events/Lang-NEXT/Lang-NEXT-2014/Keynote-Duality))

||One|Many|
|---:|:---|:---|
|__sync__|`T`|_`Enumerable`_`<T>`|
|__async__|_`Future`_`<T>`|_`Observable`_`<T>`|

<br/>

The __sync__ types are pull models, the __async__ types are push models (when _Future_ has `then()`)

push and pull producers can be implemented __sync__. (e.g. a push model `subscribe()` would directly call `value()` or `error()` and return). 

pull model is great for __sync__ values distributed in space, but push model works well too.

pull model is not a good fit for __async__ producers of values distributed in time, it requires blocking the consumption context. 

push is a very common pattern used for __async__ producers. for example, callbacks are very common, but suffer from a lack of composibility. The usability and composibility are poor, because the callback contract varies from producer to producer. creating a formal set of push model _Concepts_ for producers and consumers, provides a stable contract that allows; composition, shared algorithms and extensibility.

## coroutines

coroutines rewrite code to make a push producer appear to be pull. In doing so, coroutines serialize async work. 

```cpp
auto x = co_await produce();
auto y = co_await produce();
```

the async work to produce y will not start until x is produced and delivered to the consumer. As a result, it is quite challenging to __implement__ `when_any()` or `when_all()` using coroutines.

## its a wrap

- push is dual to pull
- push and pull allow composable algorithms to operate on POD values.
- The _Single_ _Concepts_ support _Forward_, _Input_ and _Output_ iteration implementations.
- _Single_ is push for 1 async value
- use push for async values
- use _Single_ to implement `promise`

## continued soon

hopefully with the code promised last time.
