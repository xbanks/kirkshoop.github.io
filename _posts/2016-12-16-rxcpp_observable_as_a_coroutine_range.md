---
title: rxcpp observable as a coroutine range
layout: post
date: 2016-12-16 16:33 PST

---

The coroutine proposal is an exciting way to leverage the compiler to rewrite your code into a state machine useful for turning callbacks like `.then()` into sequential code blocks. Not only does it hide future syntax, but it naturally turns async code sequential without blocking a thread like `.get()` does.

## coroutine proposal 

The coroutine proposal includes __range-for-co_await__ syntax.

```cpp

for co_await (auto& v : asyncRange) {
    //...
}

```

Similar to the __range-for__ syntax, the __range-for-co_await__ is expanded by the compiler

```cpp

auto cursor = co_await begin(asyncRange);
auto end = end(asyncRange);
for (; cursor != end; co_await ++cursor) {
    auto& v = *cursor;
}

```

## __rxcpp__ and __range-for-co_await__

I recently added awaitable range support to __rxcpp__ in `rx-coroutine.hpp`. This allows __range-for-co_await__ syntax to be used with an __rxcpp__ observable instead of subscribe.

```cpp

#include <rxcpp/rx-lite.hpp>
#include <rxcpp/operators/rx-take.hpp>

#include <rxcpp/rx-coroutine.hpp>

using namespace rxcpp;
using namespace rxcpp::sources;
using namespace rxcpp::operators;
using namespace rxcpp::util;

using namespace std;
using namespace std::chrono;

future<void> intervals(){
    for co_await (auto c : interval(seconds(1), observe_on_event_loop()) | take(3)) {
        printf("%d\n", c);
    }
}

int main()
{
    intervals().get();
    return 0;
}

```

## awaitable range

The `std::begin()` and `std::end()` free functions must be overloaded for `rxcpp::observable<T>`

`std::end()` returns a sentinel iterator immediately.

`std::begin()` returns an awaitable that will return an iterator when used with `co_await`

```cpp

namespace std
{

template<typename T, typename SourceOperator>
auto begin(const rxcpp::observable<T, SourceOperator>& o)
    ->      rxcpp::coroutine::co_observable_iterator_awaiter<rxcpp::observable<T, SourceOperator>> {
    return  rxcpp::coroutine::co_observable_iterator_awaiter<rxcpp::observable<T, SourceOperator>>{o};
}

template<typename T, typename SourceOperator>
auto end(const rxcpp::observable<T, SourceOperator>&)
    ->      rxcpp::coroutine::co_observable_iterator<rxcpp::observable<T, SourceOperator>> {
    return  rxcpp::coroutine::co_observable_iterator<rxcpp::observable<T, SourceOperator>>{};
}

}

```

#### co_observable_iterator_awaiter

There are three functions to implement an awaitable.

`await_ready()` returns false to force `await_suspend()` to be called with a handle that can be used to resume the callers execution (makes the `co_await` step forward to the call to `await_resume()`).

```cpp

bool await_ready() {
    return false;
}

```

`await_suspend()` stores the handle and then calls `subscribe()` on the observable. The functions inserted into the observable invoke the stored handle to step the current `co_await` to call the correct `await_resume()`. There are two points in __range-for-co_await__ that use co_await and the state is shared between them. 

> To fix a lifetime bug in the initial code a `weak_ptr<>` was introduced here that is `lock()`ed when needed. This allows the state to `unsubscribe()` the observable in its destructor.

```cpp

void await_suspend(coroutine_handle<> handle) {
    weak_ptr<co_observable_iterator_state<Source>> wst=it.state;
    it.state->caller = handle;
    it.state->o |
        rxo::finally([wst](){
            auto st = wst.lock();
            if (st && !!st->caller) {
                auto caller = st->caller;
                st->caller = nullptr;
                caller();
            }
        }) |
        rxo::subscribe<value_type>(
            it.state->lifetime,
            // next
            [wst](const value_type& v){
                auto st = wst.lock();
                if (!st || !st->caller) {terminate();}
                st->value = addressof(v);
                auto caller = st->caller;
                st->caller = nullptr;
                caller();
            },
            // error
            [wst](exception_ptr e){
                auto st = wst.lock();
                if (!st || !st->caller) {terminate();}
                st->error = e;
                auto caller = st->caller;
                st->caller = nullptr;
                caller();
            });
}

```

`await_resume()` will throw a pending error or return a valid iterator

```cpp

co_observable_iterator<Source> await_resume() {
    if (!!it.state->error) {rethrow_exception(it.state->error);}
    return std::move(it);
}

```

#### co_observable_iterator

The salient function in the iterator is the pre-increment operator that returns an awaitable that will be resumed when the next value occurs

```cpp

co_observable_inc_awaiter<Source> operator++()
{
    return co_observable_inc_awaiter<Source>{state};
}

```

#### co_observable_inc_awaiter

`await_ready()` returns false to force `await_suspend()` to be called with a handle that can be used to resume the callers execution (makes the `co_await` step forward to the call to `await_resume()`).

```cpp

bool await_ready() {
    return false;
}

```

`await_suspend()` returns a bool. If the return is true, then execution is suspended. If the return value is false, `await_resume()` is called. The handle is stored when there is a value pending. This handle is used in the functions passed to the `subscribe()` to resume by calling this `await_resume()`.

```cpp

bool await_suspend(coroutine_handle<> handle) {
    if (!state->lifetime.is_subscribed()) {return false;}
    state->caller = handle;
    return true;
}

```

`await_resume()` will throw a pending error or return a valid iterator

```cpp

co_observable_iterator<Source> await_resume() {
    if (!!state->error) {rethrow_exception(state->error);}
    return co_observable_iterator<Source>{state};
}

```

## a matter of style

Which is better?

```cpp

try {
    for co_await (auto& v : o) {
        cout << v << endl;
    }
    cout << "done" << endl;
} catch (const exception& e) {
    cout << e.what() << endl;
}

```

vs.

```cpp

o | subscribe<int>([](int v){
        cout << v << endl;
    },  [](exception_ptr ep){
        cout << what(ep) << endl;
    },  [](){
        cout << "done" << endl;
    });

```
as with many things - season your code to taste.
