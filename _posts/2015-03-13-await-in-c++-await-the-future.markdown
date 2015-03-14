---
layout: post
title:  "await in c++ - await the future"
date:   2015-03-13 11:00:00
categories: async rxcpp await c++
---

I will start with this simple example from Gor's presentation ([PDF](https://github.com/CppCon/CppCon2014/blob/master/Presentations/await%202.0%20-%20Stackless%20Resumable%20Functions/await%202.0%20-%20Stackless%20Resumable%20Functions%20-%20Gor%20Nishanov%20-%20CppCon%202014.pdf), [YouTube](https://www.youtube.com/watch?v=KUhSjfSbINE)):

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

The await keyword connects the return type of `schedule_test()` (`std::future<T>`) to the return type of `schedule()` (`awaiter`). The compiler makes all the calls to `await_ready`, `await_suspend`, `await_resume` and `set_result`.

### output
```sh
  8068 - main
  8068 - schedule_test
  8068 - main - returned
  4324 - schedule_test - lambda
  4324 - schedule_test - answer = 42
  8068 - main - answer = 42
```
Notice that `schedule_test()` is running on two different threads because it returned after calling `CreateThreadpoolTimer()` and then was resumed by `TimerCallback()`. The complexity that the compiler is hiding makes the code much cleaner, but does not allow the user to ignore threads. For example, adding a `std::unique_lock<std::mutex>` to `schedule_test()` would lock in `8068` and unlock in `4324`. This is illegal usage.

Libraries that provide safe and efficient algorithms that compose with the await proposal are needed as much as the algorithms in the existing STL and the proposed RangeV3 libraries. I will show some options for these in subsequent posts.

### try it out
After installing [Visual Studio 2015 Preview](https://www.visualstudio.com/en-us/news/vs2015-vs.aspx) clone  [await](https://github.com/kirkshoop/await) and open the await solution. Currently the implementation requires that RTC be disabled and `/await` for the compiler and x64 machine target for the linker, which this solution has configured correctly.
