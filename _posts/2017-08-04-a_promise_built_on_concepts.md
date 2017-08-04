---
title: a promise built on concepts
layout: post
date: 2017-08-04
---

I have written a [promise](https://github.com/kirkshoop/promise/blob/master/promise_1.h), a [promise/future pair](https://github.com/kirkshoop/promise/blob/master/promise_2.h) and a [packaged_task/future pair](https://github.com/kirkshoop/promise/blob/master/packaged_task.h) on top of the Single concepts!

I have written derivatives of this pattern 5 times now. each time it is painful. This one was no less painful.

> disclaimer - This code was written for exposition. 
> - it is not lock-free
> - it is not fully optimized 
> - it is not feature complete

__Sean Parent__ and I had a long discussion. Sean, challenged whether the lifetime needed to be a strong reference to the state. __Gor Nishanov__ has repeatedly asked me to fold the allocations and lifetime together.

This implementation was able to achieve both; lifetime as a weak_ptr and allocation folding. This required small changes to the concepts. Here are the concepts used in the implementation.

> I am still trying to correctly describe these using concepts-lite. I need a way to allow `single::value()` and `single::error()` to leave their arguments unconstrained.

```cpp
template <typename L>
concept bool Lifetime() {
    return requires(const L& l) {
        { l.is_stopped() } -> bool;
        { l.stop() } -> void;
    };
}

template <typename L>
concept bool Context() {
    return requires(const L& l) {
        { l.get_lifetime() } -> Lifetime;
    };
}

template <typename S>
concept bool Single() {
    return requires(const S& s) {
        { s.start(auto) } -> void;
        { s.value(auto) } -> void;
        { s.error(auto) } -> void;
    };
}

template <typename P, typename C>
concept bool SingleDeferred() {
    return requires(const P& in, const C& out) {
        requires Lifetime<in.subscribe(out)>;
    };
}

struct alifetime
{
    bool is_stopped() const;
    void stop() const;
};
static_assert(Lifetime<alifetime>(), "not a Lifetime");

struct acontext
{
    alifetime get_lifetime() const;
};
static_assert(Context<acontext>(), "not a Context");

struct asingle
{
    void start(acontext c) const;

    template<typename T>
    void value(int t) const;

    template<typename E>
    void error(std::exception_ptr e) const;
};
static_assert(Single<asingle>(), "not a Single");

struct asingledeferred
{
    alifetime subscribe(asingle) const;
};
static_assert(SingleDeferred<asingledeferred>(), "not a SingleDeferred");

```

## _Single_ implementation

A full description of the concepts and this implementation of them will eventually be written.

This post will focus on how they are assembled into a promise.

## _Promise_ implementation

> I am implementing subsets of existing promise proposals here. I believe that adopting _Single_ will involve changes to the design and surface of the final accepted promise.

Here is the code for a promise that does not have a future. 

> yep, avoiding all the entendres is just too much work

This is similar to the implementation that __David Sankel__ [presented](https://www.youtube.com/watch?v=pKMZjd9CFnw) @CppNow 2017.

```cpp
template<typename T>
struct promise
{
    using result_type = std::decay_t<decltype(
        std::declval<single_async_subject<T>>()
            .get_single_deferred())>;

    result_type result;
    lifetime<token_lifetime> token;

    promise(result_type r, lifetime<token_lifetime> t) : 
        result(r), token(t) {}

    template<typename F>
    explicit promise(F&& f, ...) {
        single_async_subject<T> sub;
        result = sub.get_single_deferred();
        token = single_create([f](auto s){
            f(  [s](auto v){ s.value(v); }, 
                [s](auto e){ s.error(e); });
        }) | 
        single_subscribe(sub.get_single());
    }
    template<typename F>
    promise(std::launch policy, F&& f) {
        single_async_subject<T> sub;
        result = sub.get_single_deferred();

        token = single_create([f](auto s){
                f(  [s](auto v){ s.value(v); }, 
                    [s](auto e){ s.error(e); });
            }) | 
            single_produce_on(policy) |
            single_subscribe(sub.get_single());
    }

    template<typename F>
    auto then(F&& f) const {
        using R = decltype(f(std::declval<T>()));
        single_async_subject<R> sub;
        auto l = result | 
            single_transform(std::forward<F>(f)) | 
            single_subscribe(sub.get_single());
        return promise<R>{sub.get_single_deferred(), l};
    }

    void cancel() const {
        token.stop();
    }

    void wait() {
        result | single_wait();
    }

};
```

There are two functions that perform the essential work.

### public `producer` constructor

This constructor takes a launch policy for `std::async()` and a function that will produce the result. 

__example usage__

```cpp
auto p = promise<int>{[](auto v, auto ){
    std::this_thread::sleep_for(1s);
    v(42);
}};
```

__implementation__

```cpp
template<typename F>
promise(std::launch policy, F&& f) {
    single_async_subject<T> sub;
    result = sub.get_single_deferred();

    token = single_create([f](auto s){
            f(  [s](auto v){ s.value(v); }, 
                [s](auto e){ s.error(e); });
        }) | 
        single_produce_on(policy) |
        single_subscribe(sub.get_single());
}
```

The producer function is packaged as a _SingleDeferred_. When subscribed, it will store the _Single_ consumer in two lambdas and call the function passing the two lambdas.

> Design Note: I think that splitting into lambdas discards too much information and introduces the likely hood that composition will suffer (e.g. when only one is stored or passed along to another function).

This overload of the `produce_on` algorithm uses `std::async()` to run the subscription (and therefore the function) using `std::async()`.

The subscribe invokes the expression and directs the result to the `async_subject`.

> hold on - the subject is important and has its own section

### public `consumer` algorithm

`then()` takes a transform function to run on the result value.

__example usage__

```cpp
p
    .then([](int v){ 
        return std::to_string(v); 
    })
    .then([](std::string s){ 
        std::cout << s << std::endl; 
        return 0; 
    });
```

__implementation__

```cpp
template<typename F>
auto then(F&& f) const {
    using R = decltype(f(std::declval<T>()));
    single_async_subject<R> sub;
    auto l = result | 
        single_transform(std::forward<F>(f)) | 
        single_subscribe(sub.get_single());
    return promise<R>{sub.get_single_deferred(), l};
}
```

This just builds the internals of a new promise and chains the transform onto the previous result and subscribes the results into the `async_subject`.

## `async_subject` 

A _SingleSubject_ is a conduit. It connects a producer and many consumers. It has a _Single_ that the producer calls and it has a _SingleDeferred_ that the consumer calls. When the producer calls a method on the _Single_, this is forwarded to the _Single_ that each consumer passed to `subscribe()`.

An `async_subject` is a _SingleSubject_ that implements an async rendezvous between the producer and consumers. it allows them to be connected in any order, but not miss the result.

An `async_subject` is the core implementation of all existing promises. Since they run the producer before a consumer subscribes they must implement unnecessary complexity and overhead to solve the race between production and consumption for each and every value. 

IMO the only race in the system should be the cancellation form a consumer to a producer. cancellation is rare compared to transferring a result from the producer to the consumer. 

Even worse is that `then()` is forced to create a whole new `async_subject` for each use. I might have multiple `.then()`, but instead of letting the compiler inline all the nested `value()` the implementations are forced to manage a race between each `value()` and the next.

> In a later post, I will show a promise based on these concepts that will be able to inline `value()` and `error()` across algorithms.

## more..

The next couple of posts will show `promise`/`future` and `packaged_task`/`future` code.

Then the contracts, dependencies, flows of the _Single_ concepts and the internals of this implementation of the _Single_ concepts will be explored.