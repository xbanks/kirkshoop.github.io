---
title: Flux Architecture in rxcpp
layout: post
date: 2016-12-06 08:00 PST
---

[![twitter for president-elect](/assets/twitter_analysis_president_elect.gif)](https://www.youtube.com/watch?v=QFcy-jQpvBg)
[youtube](https://www.youtube.com/watch?v=QFcy-jQpvBg)

## Flux Architecture
[Flux Architecture](https://facebook.github.io/flux/docs/overview.html) is a pattern popularized by Facebook. The Flux pattern organizes the flow of an application into a pipe of action -> dispatcher -> model -> view. While it is primarily applied to client applications today, the same pattern applies to server applications as well. Incoming requests to a server are actions and outgoing responses are the view.

Several people have recognized that the dispatcher is a two line Rx expression.

```cpp

    scan(make_shared<Model>(), [=](shared_ptr<Model>& m, Reducer& f){
        return f(m);
    }) | 
    start_with(make_shared<Model>())

```

> `scan()` is like `reduce` and `accumulate` except that the state is emitted every time it is modified.

> A `Reducer` is `function<Model(Model&)>`. Every change to the `Model` is applied by emitting a `Reducer` into the `scan()`

## counting tweets
Store the count in the Model.

```cpp

    struct Model
    {
        // ...
        int total = 0;
        // ...
    };

```

Collect the actions that produce reducers.

```cpp

    vector<observable<Reducer>> reducers;

```

Split the incoming tweets into 1 second windows on a background thread.

```cpp

    reducers.push_back(
        ts |
        onlytweets() |
        window_with_time(milliseconds(1000), poolthread) |
        // ...

```

> `window_with_time()` emits a new observable for each window. The observables only emit tweets that arrived during that time.

Produce a count at the end of each window and then emit a Reducer to update the Model with the count.

```cpp

        // ...
        rxo::map([](observable<shared_ptr<const json>> source){
            return source |
                count() | 
                rxo::map([](int count){
                    return Reducer([=](shared_ptr<Model>& m){
                        m->total += count;
                        return std::move(m);
                    });
                });
        }) |
        merge() |
        nooponerror() |
        start_with(noop));

```

> `count()` is an accumulator, it only emits one value at the end.

> `nooponerror()` recovers from errors.

Take all the actions and merge them into one stream that emits `Reducer`s. Emit all `Reducer`s on the mainthread.

```cpp

    // combine things that modify the model
    auto reducers = iterate(reducers) |
        // give the reducers to the UX
        merge(mainthread);

```

Apply the `Reducer`s to the Model. Collect a series of changes to the Model and then emit one updated Model every 200ms.

```cpp

    // apply reducers to the model (Flux architecture)
    auto models = reducers |
        // apply things that modify the model
        scan(make_shared<Model>(), [=](shared_ptr<Model>& m, Reducer& f){
            try {
                auto r = f(m);
                r->timestamp = mainthread.now();
                return r;
            } catch (const std::exception& e) {
                cerr << e.what() << endl;
                return std::move(m);
            }
        }) | 
        start_with(make_shared<Model>()) |
        // view model updates every 200ms
        sample_with_time(milliseconds(200), mainthread) |
        publish() |
        ref_count();

```

> Initially I used `debounce()` instead of `sample_with_time()`. `debounce()` caused updates to occur only when there was a 200ms pause between updates. Changing to `sample_with_time()` made an instant improvement in the rendering fluidity.

There is now a Model being updated on the mainthread using data collected from other threads. The Model is emitted every 200ms and is ready to render. 

> The code to produce the Model is platform and gui-framework agnostic and can be shared to build apps for different targets.

## up next
[Composing rxcpp and Range-v3]({{ site.baseurl }}{% post_url 2016-12-06-composing_rxcpp_and_range-v3 %})

## more on this application
[Realtime analysis using the twitter stream API]({{ site.baseurl }}{% post_url 2016-12-04-realtime_analysis_using_the_twitter_stream_api %}) 
