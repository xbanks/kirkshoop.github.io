---
title: await in c++ - await, then yield, the next
layout: post
date: 2015-03-17
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

### diagram
{% plantuml %}
participant compiler
participant caller
participant async_test
participant schedule_periodically
participant schedule
participant suspend_always
participant yield_to
participant yield_from
participant awaiter
participant promise
participant future
participant async_generator
participant async_iterator
participant os
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

### try it out
After installing [Visual Studio 2015 Preview](https://www.visualstudio.com/en-us/news/vs2015-vs.aspx) clone  [await](https://github.com/kirkshoop/await) and open the await solution. This code is in the **asyncop** project in the solution.
