---
title: await in c++ - await, then yield, the next
layout: post
date: 2015-03-19 01:45

categories: [async, rxcpp, await, c++,]
---

`async_generator<T>` implements a new AsyncRange Concept. `begin()` and `++iterator` return an awaitable type that produces an iterator later, while `end()` returns an `iterator` immediately. A new set of algorithms is needed and a new `async-range-for` has been added that inserts `await` into `i = await begin()` and `await ++i`. A function that returns  `async_generator<T>` is allowed to use `await` and `yield_value`.

```cpp
// asyncop.cpp : Defines the entry point for the console application.
//

#include <iostream>
#include <future>
#include <sstream>

#include <chrono>
namespace t = std::chrono;
using clk = t::system_clock;
using namespace std::chrono_literals;

#include <experimental/resumable>
#include <experimental/generator>
namespace ex = std::experimental;

#include "../async_generator.h"
#include "../async_schedulers.h"
namespace as = async::scheduler;
#include "../async_operators.h"
namespace ao = async::operators;

template<class Work, typename U = decltype(std::declval<Work>()(0))>
async::async_generator<U> schedule_periodically(
    clk::time_point initial,
    clk::duration period,
    Work work) {
    int64_t tick = 0;
    auto what = [&tick, &work]() {
        return work(tick);
    };
    for (;;) {
        auto when = initial + (period * tick);
        auto result = __await as::schedule(when, what);
        __yield_value result;
        ++tick;
    }
}

template<class... T>
void outln(T... t) {
    std::cout << std::this_thread::get_id();
    int seq[] = {(std::cout << t, 0)...};
    std::cout << std::endl;
}

std::future<void> async_test() {
    auto start = clk::now();
    for __await(auto&& rt :
        schedule_periodically(start + 1s, 1s,
            [](int64_t tick) {return tick; }) ) {
        outln(" for await - ", rt);
        if (rt == 5) break;
    }
}

int wmain() {
    try {
        outln(" wmain start");
        auto done = async_test();
        outln(" wmain wait..");
        done.get();
        outln(" wmain done");
    }
    catch (const std::exception& e) {
        outln(" exception ", e.what());
    }
}
```
`schedule_periodically()` uses the awaitable `schedule()` explored in [await the future](http://kirkshoop.github.io/async/rxcpp/await/c++/2015/03/13/await-in-c++-await-the-future.html) and `await` + `yield_value` to implement a stable periodic tick.

`async_test()` uses `for await` to respond to the ticks produced by `schedule_periodically()`.

This simple code results in a monster diagram! A digression is in order.

### digression

The [github history](https://github.com/kirkshoop/await/commits/master) has a lot of deleted code from all my attempts to get a working `async_generator`. I started with the simple await examples and made small modifications. Then I wrote a very complex implementation of `schedule_periodically`.

```cpp
template<class Work>
auto schedule_periodically(std::chrono::system_clock::time_point initial, std::chrono::system_clock::duration period, Work work) {
    struct async_schedule_periodically
    {
        static void CALLBACK TimerCallback(PTP_CALLBACK_INSTANCE, void* Context, PTP_TIMER) {
            auto that = reinterpret_cast<async_schedule_periodically*>(Context);
            auto result = that->work(that->tick++);
            that->current_value = &result;
            that->waiter();
        }
        static void Next(async_schedule_periodically** ppParent, ex::resumable_handle<> cb) {
            auto that = *ppParent;

            that->waiter = cb;
            that->ppParent_ = ppParent;

            auto at = that->initial + (that->period * that->tick);
            auto duration = std::chrono::duration_cast<std::chrono::nanoseconds>(at - std::chrono::system_clock::now());
            int64_t relative_count = -(duration.count() / 100);

            that->timer = CreateThreadpoolTimer(TimerCallback, that, nullptr);
            if (that->timer == 0) throw std::system_error(GetLastError(), std::system_category());
            SetThreadpoolTimer(that->timer, (PFILETIME)&relative_count, 0, 0);
        }
        PTP_TIMER timer = nullptr;
        ex::resumable_handle<> waiter;
        int64_t tick = 0;
        async_schedule_periodically ** ppParent_;

        std::chrono::system_clock::time_point initial;
        std::chrono::system_clock::duration period;
        Work work;

        typedef decltype((*(Work*)nullptr)(int64_t{})) value_type;
        value_type const* current_value;

        struct async_iterator
        {
            async_schedule_periodically * parent_;

            auto operator++() {
                struct awaiter {
                    async_schedule_periodically ** parent_;
                    bool await_ready() {
                        return false;
                    }
                    void await_suspend(ex::resumable_handle<> cb) {
                        Next(parent_, cb);
                    }
                    void await_resume() {
                    }
                    ~awaiter(){
                        if ((*parent_)->timer) CloseThreadpoolTimer((*parent_)->timer);
                        (*parent_)->timer = 0;
                    }
                };
                return awaiter{ &parent_ };
            }

            async_iterator operator++(int) = delete;

            bool operator==(async_iterator const& _Right) const { return parent_ == _Right.parent_; }

            bool operator!=(async_iterator const& _Right) const { return !(*this == _Right); }

            value_type const& operator*() const {
                return *parent_->current_value;
            }

            value_type const* operator->() const { return std::addressof(operator*()); }

        };

        async_schedule_periodically(
            std::chrono::system_clock::time_point i,
            std::chrono::system_clock::duration p,
            Work w)
        : initial(i), period(p), work(std::move(w)), tick(0) {}

        auto begin() {
            struct awaiter {
                async_schedule_periodically * pChannel;

                bool await_ready() { return false; }

                void await_suspend(ex::resumable_handle<> cb) {
                    Next(&pChannel, cb);
                }
                auto await_resume() {
                    return async_iterator{ pChannel };
                }
                ~awaiter(){
                    if (pChannel->timer) CloseThreadpoolTimer(pChannel->timer);
                    pChannel->timer = 0;
                }
            };
            return awaiter{ this };
        }

        async_iterator end() { return{ nullptr }; }
    };

    return async_schedule_periodically{ initial, period, std::move(work) };
}
```

This worked but the code at the beginning of this post hides so much complexity and, as I will show in a future post, `async_generator` will allow much more interesting functions than `schedule_periodically`. It took a long time to merge the `generator` and this complex `schedule_periodically` to arrive at `async_generator`. The key was splitting out `async_iterator`, `yield_to` and `yield_from`. It was much easier to assemble those pieces together than the monolithic code above. Along the way I ran into codegen bugs and lifetime bugs. Gor Nishanov was very kind and after much toil and tribulation I saw the simple code above start working.

But enough degression, on to the diagram.

### diagram

I spent a lot of time rearranging and refining this diagram because it contains a lot of information. In the next section I will try to build a story out of this diagram.

{% plantuml %}
participant compiler
box "awaitable's"
participant yield_to << async_generator >>
participant yield_from << async_generator >>
participant suspend_always << promise >>
participant awaiter << schedule >>
end box
participant promise << async_generator >>
participant os
box "function's"
participant caller
participant async_test
participant schedule_periodically
participant schedule
end box
participant async_iterator
participant async_generator
participant future
activate caller
"caller" -> "async_test" : "async_test()"
activate async_test
"async_test" -> "schedule_periodically" : "schedule_periodically()"
group "initial"
activate schedule_periodically
activate promise
"compiler" -> "promise" : "get_return_object()"
"compiler"<-- "promise" : "*this"
activate async_generator
"compiler" -> "promise" : "initial_suspend()"
"compiler"<-- "promise" : "suspend_always()"
activate suspend_always
"compiler" -> "suspend_always" : "await_ready()"
"compiler"<-- "suspend_always" : "false"
"compiler" -> "suspend_always" : "await_suspend(resumable_handle<>)"
"compiler"<-- "suspend_always" : "void"
end
deactivate schedule_periodically
"async_test" <-- "schedule_periodically" : "async_generator<int64_t>()"
group "iteration 0"
"async_test" -> "async_generator" : "await begin()"
"async_generator" -[#blue]> "schedule_periodically" : "from()"
"compiler" -> "promise" : "cancellation_requested()"
"compiler"<-- "promise" : "false"
"compiler" -> "suspend_always" : "await_resume()"
"compiler"<-- "suspend_always" : "void"
deactivate suspend_always
"schedule_periodically" -> "schedule" : "await schedule()"
activate schedule
deactivate schedule
"schedule_periodically" <-- "schedule" : "awaiter()"
activate awaiter
"compiler" -> "awaiter" : "await_ready()"
"compiler"<-- "awaiter" : "false"
"compiler" -> "awaiter" : "await_suspend(resumable_handle<>)"
"awaiter" -> "os" : "CreateThreadpoolTimer()"
"compiler"<-- "awaiter" : "void"
"async_generator" <[#green]-- "schedule_periodically" : "void"
activate yield_to
"compiler" -> "yield_to" : "await_ready()"
"compiler"<-- "yield_to" : "false"
"compiler" -> "yield_to" : "await_suspend(resumable_handle<>)"
"compiler"<-- "yield_to" : "void"
group "wait for completion"
"caller" <-- "async_test" : "std::future<void>()"
"caller" -> "future" : "get()"
end
... one second ...
"awaiter" <-- "os" : "TimerCallback()"
"compiler" -> "promise" : "cancellation_requested()"
"compiler"<-- "promise" : "false"
"compiler" -> "awaiter" : "await_resume()"
"compiler"<-- "awaiter" : "work()"
deactivate awaiter
"schedule_periodically" -> "promise" : "await yield_value()"
"compiler" -> "promise" : "yield_value()"
"compiler"<-- "promise" : "yield_from()"
activate yield_from
"compiler" -> "yield_from" : "await_ready()"
"compiler"<-- "yield_from" : "false"
"compiler" -> "yield_from" : "await_suspend(resumable_handle<>)"
"yield_from" -[#blue]> "async_test" : "to()"
"compiler" -> "yield_to" : "await_resume()"
"compiler"<-- "yield_to" : "async_iterator()"
activate async_iterator
deactivate async_iterator
deactivate yield_to
"async_test" <-- "async_generator" : "async_iterator<int64_t>()"
activate async_iterator
"async_test" -> "async_iterator" : "operator*()"
"async_test" <-- "async_iterator" : "in64_t()"
end
group "iteration 1..N"
"async_test" -> "async_iterator" : "await operator++()"
activate yield_to
"compiler" -> "yield_to" : "await_ready()"
"compiler"<-- "yield_to" : "false"
"compiler" -> "yield_to" : "await_suspend(resumable_handle<>)"
"yield_to" -[#blue]> "schedule_periodically" : "from()"
"compiler" -> "promise" : "cancellation_requested()"
"compiler"<-- "promise" : "false"
"compiler" -> "yield_from" : "await_resume()"
"compiler"<-- "yield_from" : "async_iterator()"
deactivate yield_from
"schedule_periodically" <-- "promise" : "void"
"schedule_periodically" -> "schedule" : "await schedule()"
activate schedule
deactivate schedule
"schedule_periodically" <-- "schedule" : "awaiter()"
activate awaiter
"compiler" -> "awaiter" : "await_ready()"
"compiler"<-- "awaiter" : "false"
"compiler" -> "awaiter" : "await_suspend(resumable_handle<>)"
"awaiter" -> "os" : "CreateThreadpoolTimer()"
"compiler"<-- "awaiter" : "void"
"yield_to" <[#green]-- "schedule_periodically" : "void"
"compiler"<-- "yield_to" : "void"
"yield_from" <[#green]-- "async_test" : "void"
"compiler"<-- "yield_from" : "void"
... one second ...
"awaiter" <-- "os" : "TimerCallback()"
"compiler" -> "promise" : "cancellation_requested()"
"compiler"<-- "promise" : "false"
"compiler" -> "awaiter" : "await_resume()"
"compiler"<-- "awaiter" : "work()"
deactivate awaiter
"schedule_periodically" -> "promise" : "await yield_value()"
"compiler" -> "promise" : "yield_value()"
"compiler"<-- "promise" : "yield_from()"
activate yield_from
"compiler" -> "yield_from" : "await_ready()"
"compiler"<-- "yield_from" : "false"
"compiler" -> "yield_from" : "await_suspend(resumable_handle<>)"
"yield_from" -[#blue]> "async_test" : "to()"
"compiler" -> "yield_to" : "await_resume()"
"compiler"<-- "yield_to" : "async_iterator()"
activate async_iterator
deactivate async_iterator
deactivate yield_to
"async_test" <-- "async_iterator" : "async_iterator<int64_t>()"
"async_test" -> "async_iterator" : "operator*()"
"async_test" <-- "async_iterator" : "in64_t()"
end
deactivate async_test
deactivate async_iterator
deactivate async_iterator
"compiler" -> "async_generator" : "~async_generator()"
group "final"
"async_generator" -[#blue]> "schedule_periodically" : "from()"
"compiler" -> "promise" : "cancellation_requested()"
"compiler"<-- "promise" : "true"
deactivate yield_from
"compiler" -> "promise" : "final_suspend()"
"compiler"<-- "promise" : "suspend_always()"
activate suspend_always
"compiler" -> "suspend_always" : "await_ready()"
"compiler"<-- "suspend_always" : "false"
"compiler" -> "suspend_always" : "await_suspend(resumable_handle<>)"
"compiler"<-- "suspend_always" : "void"
"async_generator" <[#green]-- "schedule_periodically" : "void"
"async_generator" -[#blue]> "schedule_periodically" : "from()"
"compiler" -> "suspend_always" : "await_resume()"
"compiler"<-- "suspend_always" : "void"
deactivate suspend_always
"compiler" -> "promise" : "~promise()"
"compiler"<-- "promise" : "void"
deactivate promise
"async_generator" <[#green]-- "schedule_periodically" : "void"
end
"compiler"<-- "async_generator" : "void"
deactivate async_generator
"yield_from" <[#green]-- "async_test" : "void"
"compiler"<-- "yield_from" : "void"
"caller" <-- "future" : "void"
deactivate caller
{% endplantuml %}

Whew! I wrote an [instrumented app](https://github.com/kirkshoop/await/blob/master/instrumented.cpp) to generate this diagram so it is pretty accurate. In the process it found a few bugs in the async_generator that are [fixed](https://github.com/kirkshoop/await/commit/e45ed5d65773a6c84f4f64e35a2fa50acea7e08f) now!

### breakdown the diagram

First of all, the diagram is divided into; 

* the left side, which is driven by the compiler
* the right side, which shows the function calls from the source at the top of this post.

Right in the middle is the `promise`. The `promise` provided by the `async_generator` is the rendezvous point that is used to coordinate the left and right sides. The `yield_to` and `yield_from` awaitables are used to ping-pong the execution between the consumer and the producer. 

`async_iterator`, `async_generator` and `promise_type` can be reduced to these core functions. 

```cpp
template<typename T, typename Promise>
struct async_iterator : public std::iterator<std::input_iterator_tag, T>
{
    async_iterator(Promise* p) : p(p) {}
    yield_to<T, Promise> operator++() {return{ p };}
. . .
    T const& operator*() const
    {
        if (p->Error) {std::rethrow_exception(p->Error);}
        return *p->CurrentValue;
    }
};

template<typename T, typename Alloc = std::allocator<char>>
struct async_generator
{
    struct promise_type {
        ex::resumable_handle<> To{ nullptr };
        ex::resumable_handle<> From{ nullptr };
    . . .
        promise_type& get_return_object() {return *this;}
        ex::suspend_always initial_suspend() {return{};}
        ex::suspend_always final_suspend() {return{};}
        bool cancellation_requested() const {return done;}
        void set_result() {done = true;}
        void set_exception(std::exception_ptr Exc) {Error = std::move(Exc);}
        yield_from<T, promise_type> yield_value(T Value)
        {
            CurrentValue = std::addressof(Value);
            return{ this };
        }
    };
. . .
    yield_to<T, promise_type> begin() {
        Coro();
        return{ std::addressof(Coro.promise()) };
    }
};
```

`yield_to` is returned by `async_generator::begin()` and `async_iterator::operator++()`. When suspended `yield_to` will save the consumer's resumption point in the `promise` and resume the producer if it has been saved in the `promise`. When resumed `yield_to` will return a new `async_iterator` for the produced value.

```cpp
template<typename T, typename Promise>
struct yield_to
{
    yield_to(Promise* p) : p(p) {}
    bool await_ready() noexcept {return false;}
    void await_suspend(ex::resumable_handle<> r) noexcept
    {
        p->To = r;
        if (p->From) {
            ex::resumable_handle<> coro{ p->From };
            p->From = nullptr;
            coro();
        }
    }
    async_iterator<T, Promise> await_resume() noexcept {return{ p };}
    Promise* p = nullptr;
};
```

`yield_from` is returned by `promise::yield_value()` (this is called by the compiler when the producer uses the '`__yield_value` *expression*'). When suspended `yield_from` will save the producer's resumption point in the `promise` and resume the consumer if it has been saved in the `promise`.

```cpp
template<typename T, typename Promise>
struct yield_from
{
    yield_from(Promise* p) : p(p) {}
    bool await_ready() noexcept {return false;}
    void await_suspend(ex::resumable_handle<> r) noexcept
    {
        p->From = r;
        if (p->To) {
            ex::resumable_handle<> coro{ p->To };
            p->To = nullptr;
            coro();
        }
    }
    void await_resume() noexcept {}
    Promise* p = nullptr;
};
```

> the blue and green arrows in the diagram show where each saved resumption point is called to achieve the interleaved execution of the consumer and producer. 

### read the diagram

Time goes from top to bottom.

Start in the top-right side at the `caller` column. This is `wmain` calling into `async_test`. Reading the right side from top to bottom shows the progression through the consumer loop over the `async_generator` that is found in the async_test function and the producer loop that is found in `schedule_periodically`.

> The '*wait for completion*' box on the right side calls out where `async_test` returns (as soon as the os timer is scheduled) the `future` and the caller waits on it.

On the left side the vertical lines display lifetimes of the objects and show all the work that the compiler is doing to satisfy the `async_test` consumer. Starting from the top again, there are boxes for the '*initial*' setup of `async_test`, '*iteration 0*' and '*iteration 1..N*' and then '*final*' where `async_test` completes and exits.

### output

```sh
3316 wmain start
3316 wmain wait..
8240 for await - 0
8240 for await - 1
8240 for await - 2
8240 for await - 3
8240 for await - 4
8240 for await - 5
3316 wmain done
```

### teaser
just a little something from the future..

```cpp
// also known as copy_if
template<typename T, typename P>
async_generator<T> filter(async_generator<T> s, P p) {
    for __await(auto&& v : s) {
        if (p(v)) {
            __yield_value v;
        }
    }
}
```

### try it out
After installing [Visual Studio 2015 Preview](https://www.visualstudio.com/en-us/news/vs2015-vs.aspx) clone  [await](https://github.com/kirkshoop/await) and open the await solution. This code is in the **asyncop** project in the solution.
