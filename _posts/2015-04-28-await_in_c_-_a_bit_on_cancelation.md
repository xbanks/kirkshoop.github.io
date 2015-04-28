---
title: await in c++ - a bit on cancelation
layout: post
date: 2015-04-28 15:18


categories: [async, rxcpp, await, c++,]
---

When dealing with async, cancelation is essential. Cancelation is also really painful. Adding cancelation to `async_generator<T>` was just as painful as the problem space suggests. At times I have taken a break to add `async_observable<T>` and worked on some [rxcpp](https://github.com/Reactive-Extensions/RxCpp) issues and explored adding `async_generator<T>` support to Kenny Kerr's excellent [moderncpp](http://moderncpp.com/) library for writing modern windows apps in pure c++. 

Cancellation is not yet done, but this post will talk about two things that are being introduced to support cancellation.

#Access return value of function that awaits
The compiler constructs and orchestrates all calls to the return value of a function that uses await or yield. An async generator function that supports cancellation must receive a notification when the return value is canceled. I solved this by passing a function to the `async_generator<T>` being returned, that will be called when canceled.

The solution involves constructing an awaitable type that stores the callback function and then `await` the result. This causes the compiler to pass the promise for the current return type in a `resumable_handle<promise_type>` to `await_suspend` which then moves the stored function into the promise.

The following function is from my moderncpp exploration and demonstrates setting functions to be called when `begin` and `cancel` are called on the returned `async_generator`.

```cpp
auto enumeratedevices = async::async_observable<DeviceInformation>::create(
    []() -> async::async_generator<DeviceInformation> {
        DeviceWatcher watcher = DeviceInformation::CreateWatcher();

        __await async::set_onbegin([&](){
            watcher.Start();
        });

        __await async::set_oncancel([&](){
            if (watcher.Status() == DeviceWatcherStatus::Started) {
                watcher.Stop();
            }
        });

        auto devices = watcher.Added() | ao::take_until(watcher.EnumerationCompleted());

        for __await(auto& di : devices.subscribe()) {
            __yield_value di;
        }
});
```

The `set_onbegin` and `set_oncancel` functions return awaitable types that will set the supplied functions into the promise of the value to be returned.

Here is the `set_oncancel` implementation.

```cpp
namespace detail {
    template<class F>
    struct oncancel_awaiter
    {
        F f;

        bool await_ready() {
            return false;
        }

        template<class Promise>
        void await_suspend(ex::resumable_handle<Promise> r) {
            r.promise().p->oncancel = std::move(f);
            r();
        }

        void await_resume() {
        }
    };
}

template<class F>
auto add_oncancel(F&& f)
{
    return detail::oncancel_awaiter<F>{std::forward<F>(f)};
};
```

The await_suspend method has all the magic. Here is a break down.

```cpp
template<class Promise>
void await_suspend(ex::resumable_handle<Promise> r) {
```

Templating await_suspend on the resumable_handle `Promise` allows the promise to be accessed. A resumable_handle<> type-forgets the `Promise` which defeats the purpose here.

```cpp
    r.promise().p->oncancel = std::move(f);
```

With access to the promise the function can be moved into the proper member of the promise.

```cpp
    r();
}
```

This awaitable must resume the passed in handle or execution of the calling function will never resume.

####Using enumeratedevices

Here is a function that demonstrates how to use the `enumeratedevices` defined earlier. The cancellation is exercised by the `ao::take` operator used in `dumpnames` and the `ao:take_until` operator used inside `enumeratedevices'.

```cpp
auto dumpnames = [](auto& devices) -> std::future<void> {

    auto devicenames = devices | 
        ao::map([](auto& di){return di.Name();}) | 
        ao::filter([](auto& name){return name.Length() < 6 || name.Substring(0, 6) != L"VMware";}) | 
        ao::take(5);

    for __await (auto& dname : devicenames.subscribe()) {
        printf("%ls\n", dname.Buffer());
    }
};

auto done = dumpnames(enumeratedevices);

// wait for dump to finish.
done.get();
```

#Tracing
I added simple tracing functionality to `async_generator<T>` that has been really helpful when tracking down bugs.

The first change is to allow each source in the expression to be assigned an id. 

```cpp
auto test = ((((schedule_periodically(start + 1s, 1s, [](int64_t tick) {return tick; }).set_id("ticks") |
    ao::filter([](int64_t t) { return (t % 2) == 0; })).set_id("even ticks") |
    ao::map([st](int64_t tick) {
        auto ss = std::make_unique<std::stringstream>();
        *ss << std::this_thread::get_id() << " " << tick + (100 * (st + 1));
        return ss.release();
    })).set_id("tick as string") |
    ao::map([](std::stringstream* ss) {
        *ss << " " << std::this_thread::get_id();
        return ss;
    })).set_id("add thread id") |
    ao::take_until(schedule_periodically(start + 5s, 5s, [](int64_t tick) {return tick; }).set_id("cancelation"))).set_id("take 5s");
```

Then inside the `async_generator<T>` add weak_ptr for the next and previous sources in the expression and for each state transition dump the state of the whole expression.

```cpp
std::weak_ptr<GeneratorStateBase> pto;
std::weak_ptr<GeneratorStateBase> pfrom;
```

The weak_ptr are connected when `__await begin()` is called. In yield_to::await_suspend

```cpp
template<class Promise>
void await_suspend(ex::resumable_handle<Promise> r) noexcept
{
    p->pto = r.promise().p;
    r.promise.p->pfrom = p;
    . . .
}
```

####Output

Each state transition prints the state of each source in the expression. Here is an example.

```sh
>>                 tick as string -             from produce value
  [                               -             from produce value; done=0, cancel=0, error=0, to=1, from=0, final=0]
  [                 add thread id -             from produce value; done=0, cancel=0, error=0, to=1, from=0, final=0]
  [                tick as string -             from produce value; done=0, cancel=0, error=0, to=1, from=1, final=0]
  [                    even ticks -               to consume value; done=0, cancel=0, error=0, to=0, from=1, final=0]
  [                         ticks -               to consume value; done=0, cancel=0, error=0, to=0, from=1, final=0]
<<                 tick as string -             from produce value
```

The first and last line contains the id and the state being entered. Each line surrounded with '[' and ']' is the state of one source in the expression. The expression is traveled from output at the top to input at the bottom. In this case `ticks` is the start of the expression, and the source changing state is `tick as string` (the 3rd source).

The last 3 items in the status of each source indicate what resumable_handle members are valid. The `to` member indicates that begin or operator++ has been called form the outside, while `from` indicates that __yield_value has been called from the inside. The `final` member is used during cancelation. The `to` member is moved to `final` so that the caller is not resumed until the sources have been shut down.

#More to come

There is a lot more to cover on cancelation. As soon as it is working smoothly there will be more posts to describe the inner workings.