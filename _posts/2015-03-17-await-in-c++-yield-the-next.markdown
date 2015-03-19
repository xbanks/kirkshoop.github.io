---
layout: post
title:  "await in c++ - yield the next"
date: 2015-03-17 08:21

categories: [async, rxcpp, await, c++,]
---

The fibonacci example from Gor's presentation ([PDF](https://github.com/CppCon/CppCon2014/blob/master/Presentations/await%202.0%20-%20Stackless%20Resumable%20Functions/await%202.0%20-%20Stackless%20Resumable%20Functions%20-%20Gor%20Nishanov%20-%20CppCon%202014.pdf), [YouTube](https://www.youtube.com/watch?v=KUhSjfSbINE)) is the subject of this post. `generator<T>` implements the Range Concept so it works with existing algorithms from STL and Rangev3 and the `range-for` feature. It can be found in the header `experimental/generator`. A function that returns `generator<T>` is allowed to use the `yield_value` keyword (which evaluates to `await generator<T>::promise_type::yield_value(T)`).

```cpp
// yield.cpp : Defines the entry point for the console application.
//

#include <iostream>

#include <experimental/resumable>
#include <experimental/generator>
namespace ex = std::experimental;

ex::generator<int> fibonacci(int n) {
    int a = 0;
    int b = 1;

    while (n-- > 0) {
        __yield_value a;
        auto next = a + b; a = b;
        b = next;
    }
}

int wmain() {
    for (auto v : fibonacci(35)) {
        if (v > 10)
            break;
        std::cout << v << ' ';
    }
}
```

### diagram
{% plantuml %}
"caller" -> "fibonacci" : "fibonacci()"
"fibonacci" -> "generator<int>::promise_type": "generator<int>::promise_type()"
"fibonacci" -> "generator<int>::promise_type": "get_return_object()"
"generator<int>::promise_type" -> "generator<int>" : "generator()"
"fibonacci" <-- "generator<int>::promise_type": "generator<int>"
group initial
  "fibonacci" -> "generator<int>::promise_type": "await initial_suspend()"
  "fibonacci" <-- "generator<int>::promise_type": "suspend_always()"
  "fibonacci" -> "suspend_always": "await_ready"
  "fibonacci" <-- "suspend_always": "false"
  "fibonacci" -> "suspend_always": "await_suspend"
end
"caller" <-- "fibonacci": "generator<int>"
group "iteration 0"
  "caller" -> "generator<int>" : "begin()"
  "generator<int>" -[#blue]> "fibonacci" : "resume fibonacci()"
  "fibonacci" -> "generator<int>::promise_type": "await yield_value()"
  "fibonacci" <-- "generator<int>::promise_type": "suspend_always()"
  "fibonacci" -> "suspend_always": "await_ready"
  "fibonacci" <-- "suspend_always": "false"
  "fibonacci" -> "suspend_always": "await_suspend"
  "fibonacci" -[#blue]> "generator<int>" : " resume begin()"
  "caller" <-- "generator<int>" : "generator<int>::iterator()"
  "caller" -> "generator<int>::iterator" : "operator*()"
  "caller" <-- "generator<int>::iterator" : "0"
end
group "iteration 1..N"
  "caller" -> "generator<int>::iterator" : "operator++()"
  "generator<int>::iterator" -[#blue]> "fibonacci" : "resume fibonacci()"
  "fibonacci" -> "generator<int>::promise_type": "await yield_value()"
  "fibonacci" <-- "generator<int>::promise_type": "suspend_always()"
  "fibonacci" -> "suspend_always": "await_ready"
  "fibonacci" <-- "suspend_always": "false"
  "fibonacci" -> "suspend_always": "await_suspend"
  "fibonacci" -[#blue]> "generator<int>::iterator" : " resume operator++()"
  "caller" <-- "generator<int>::iterator" : "generator<int>::iterator()"
  "caller" -> "generator<int>::iterator" : "operator*()"
  "caller" <-- "generator<int>::iterator" : "1..N"
end
group final
  "fibonacci" -> "generator<int>::promise_type": "await final_suspend()"
  "fibonacci" <-- "generator<int>::promise_type": "suspend_always()"
  "fibonacci" -> "suspend_always": "await_ready"
  "fibonacci" <-- "suspend_always": "false"
  "fibonacci" -> "suspend_always": "await_suspend"
end
"caller" -> "generator<int>" : "~generator()"
{% endplantuml %}

The `fibonacci()` function uses `yield_value` to interleave the `fibonacci()` loop and the `caller` loop on the current thread. The `begin()` and `operator++()` functions resume the `fibonacci()` loop while `yield_value` resumes the `caller` loop. `generator<T>` uses this interleaving to pass the value to the `caller` as a const reference to the value inside the `fibonacci()` loop. This avoids a copy while preventing the `caller` from modifiying the state of the `fibonacci()` loop.

### try it out
After installing [Visual Studio 2015 Preview](https://www.visualstudio.com/en-us/news/vs2015-vs.aspx) clone  [await](https://github.com/kirkshoop/await) and open the await solution. This code is in the **yield** project in the solution.
