---
layout: post
title:  "await in c++ - await the future"
date:   2015-03-13 11:00:00
categories: [async, rxcpp, await, c++,]
---

I will start with the sleep_for example from Gor's presentation ([PDF](https://github.com/CppCon/CppCon2014/blob/master/Presentations/await%202.0%20-%20Stackless%20Resumable%20Functions/await%202.0%20-%20Stackless%20Resumable%20Functions%20-%20Gor%20Nishanov%20-%20CppCon%202014.pdf), [YouTube](https://www.youtube.com/watch?v=KUhSjfSbINE)). The schedule function is a small extension that adds a lambda that can return a value once the time arrives. The schedule function implements the Awaitable Concept on its return type `awaiter` via the  `await_ready`, `await_suspend` and `await_resume` functions.

```cpp
// await.cpp : Defines the entry point for the console application.
//
// In VS 2015 x64 Native prompt:
// CL.exe /Zi /nologo /W3 /sdl- /Od /D _DEBUG /D WIN32 
//   /D _CONSOLE /D _UNICODE /D UNICODE /Gm /EHsc /MDd 
//  /GS /fp:precise /Zc:wchar_t /Zc:forScope /Gd /TP /await await.cpp
//

#include <iostream>
#include <future>

#include <chrono>
namespace t = std::chrono;
using clk = t::system_clock;
using namespace std::chrono_literals;

#include <experimental/resumable>
namespace ex = std::experimental;

#include <windows.h>
#include <threadpoolapiset.h>

// usage: await schedule(std::chrono::system_clock::now() + 1s, [](){. . .});
template<class Work>
auto schedule(clk::time_point at, Work work) {
    class awaiter {
        static void CALLBACK TimerCallback(PTP_CALLBACK_INSTANCE, void* Context, PTP_TIMER) {
            ex::resumable_handle<>::from_address(Context)();
        }
        PTP_TIMER timer = nullptr;
        std::chrono::system_clock::time_point at;
        Work work;
    public:
        awaiter(std::chrono::system_clock::time_point a, Work w)
            : at(a), work(std::move(w)) {}
        bool await_ready() const {
            return std::chrono::system_clock::now() >= at;
        }
        void await_suspend(ex::resumable_handle<> resume_cb) {
            auto duration = at - std::chrono::system_clock::now();
            int64_t relative_count = -duration.count();
            timer = CreateThreadpoolTimer(TimerCallback, resume_cb.to_address(), nullptr);
            if (timer == 0)
                throw std::system_error(GetLastError(), std::system_category());
            SetThreadpoolTimer(timer, (PFILETIME)&relative_count, 0, 0);
        }
        auto await_resume() {
            return work();
        }
        ~awaiter() {
            if (timer) CloseThreadpoolTimer(timer);
        }
    };
    return awaiter{ at, work };
}

template<class... T>
void outln(T... t) {
    std::cout << std::this_thread::get_id();
    int seq[] = {(std::cout << t, 0)...};
    std::cout << std::endl;
}

std::future<int> schedule_test() {
    outln(" - schedule_test ");
    auto answer = __await schedule(clk::now() + 1s, []() {
        outln(" - schedule_test - lambda");
        return 42;
    });
    outln(" - schedule_test - answer = ", answer);
    return answer;
}

int wmain() {
    try {
        outln(" - main ");
        auto pending = schedule_test();
        outln(" - main - returned");
        int answer = pending.get();
        outln(" - main - answer = ", answer);
    }
    catch (const std::exception& e) {
        outln("schedule_test exception ", e.what());
    }
}
```

### diagram

{% plantuml %}
"caller" -> "schedule_test()" 
"schedule_test()" -> "promise<int>": "promise<int>()"
"schedule_test()" -> "promise<int>": "get_return_object()"
"promise<int>" -> "future<int>"
"schedule_test()" <-- "promise<int>": "future<int>()"
group initial
  "schedule_test()" -> "promise<int>": "await initial_suspend()"
  "schedule_test()" <-- "promise<int>": "suspend_never()"
  "schedule_test()" -> "suspend_never": "await_ready"
  "schedule_test()" <-- "suspend_never": "true"
  "schedule_test()" -> "suspend_never": "await_resume"
  "schedule_test()" -> "suspend_never": "~suspend_never()"
end
"schedule_test()" -> "promise<int>": "cancellation_requested()"
"schedule_test()" -> "schedule()": "await schedule(. . .)"
"schedule_test()" <-- "schedule()": "schedule::awaiter()"
"schedule_test()" -> "schedule::awaiter": "await_ready"
"schedule_test()" <-- "schedule::awaiter": "false"
"schedule_test()" -> "schedule::awaiter": "await_suspend(resumable_handle)"
"schedule::awaiter" -> "os": "CreateThreadpoolTimer()"
"caller" <-- "schedule_test()": "return future<int>"
"caller" -> "future<int>" : "get()"
... one second later ...
"schedule::awaiter" <-- "os": "TimerCallback()"
"schedule_test()" <-- "schedule::awaiter": "resumable_handle"
"schedule_test()" -> "schedule::awaiter": "cancellation_requested()"
"schedule_test()" -> "schedule::awaiter": "await_resume"
"promise<int>"  <-- "schedule_test()": "set_result()"
"caller" <-- "future<int>" 
group final
  "schedule_test()" -> "promise<int>": "await final_suspend()"
  "schedule_test()" <-- "promise<int>": "suspend_never()"
  "schedule_test()" -> "suspend_never": "await_ready"
  "schedule_test()" <-- "suspend_never": "true"
  "schedule_test()" -> "suspend_never": "await_resume"
  "schedule_test()" -> "suspend_never": "~suspend_never()"
end
"schedule_test()" -> "schedule::awaiter": "~awaiter()"
"schedule_test()" -> "promise<int>": "~promise()"
"caller" -> "future<int>": "~future()" 
{% endplantuml %}

The `schedule_test()` function uses `await` and `return` to stitch the `std::future<T>` return type to the `awaiter` returned by `schedule()`. The compiler creates `resumable_traits<std::future<int>>::promise_type`, which is `std::promise<int>`, gets the `future<int>` from the promise, calls `awaiter::await_ready` and `awaiter::await_suspend` and returns the empty future. Once the TimerCallback resumes the `await` the compiler calls `awaiter::await_resume` to get the result of the lambda and then, when `return` is reached, `std::promise<int>::set_result()`.

The call to `schedule_test()` in `wmain()` returns a `std::future<T>` and the `.get()` call blocks until the answer is available. Since `wmain()` can only return void or int `await` cannot be used.

### output
```sh
  8068 - main
  8068 - schedule_test
  8068 - main - returned
  4324 - schedule_test - lambda
  4324 - schedule_test - answer = 42
  8068 - main - answer = 42
```
Notice that `schedule_test()` is running on two different threads because it returned after calling `CreateThreadpoolTimer()` and then was resumed by `TimerCallback()`. The complexity that the compiler is hiding makes the code much cleaner, but does not allow the user to ignore threads. For example, adding a `std::unique_lock<std::mutex>` to `schedule_test()` would lock in `8068` and unlock in `4324`.

Libraries that provide safe and efficient algorithms that compose with the await proposal are needed as much as the algorithms in the existing STL and the proposed RangeV3 libraries. I will show some options for these in subsequent posts.

### try it out
After installing [Visual Studio 2015 Preview](https://www.visualstudio.com/en-us/news/vs2015-vs.aspx) clone  [await](https://github.com/kirkshoop/await) and open the await solution. This code is in the **await** project in the solution. Currently the implementation requires that RTC be disabled and `/await` for the compiler and x64 machine target for the linker, which this solution has configured correctly.
