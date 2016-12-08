---
title: Parsing json documents from a stream
layout: post
date: 2016-12-05
---

[![twitter for president-elect](/assets/twitter_analysis_president_elect.gif)](https://www.youtube.com/watch?v=QFcy-jQpvBg)
[youtube](https://www.youtube.com/watch?v=QFcy-jQpvBg)

## twitter stream contents
The twitter stream api emits json documents delimited by `\r\n`. Sometimes there will be partial or corrupt json documents. Sometimes there will be empty lines. Empty lines are used as a keep-alive mechanism for sparse streams.

The [json parser](https://github.com/nlohmann/json) that I selected does not support streaming natively, so there needs to be code that carves a set of strings into a set of json documents.

## parsetweets
`parsetweets` defines a new rxcpp operator that will take strings in and return json documents.

```cpp

    // parse tweets
    auto tweet$ = chunk$ | parsetweets(tweetthread);

```

To find the end of a tweet we need to know when a string ends in `\r\n`.

```cpp

    auto isEndOfTweet = [](const string& s){
        if (s.size() < 2) return false;
        auto it0 = s.begin() + (s.size() - 2);
        auto it1 = s.begin() + (s.size() - 1);
        return *it0 == '\r' && *it1 == '\n';
    };

```

Split each string on `\r\n` when it arrives. 

> `split()` uses `cregex_token_iterator` to take a string and split it into a `vector<string>`

```cpp

    // create strings split on \r
    auto string$ = chunk$ |
        concat_map([](const string& s){
            auto splits = split(s, "\r\n");
            return iterate(move(splits));
        }) |
        filter([](const string& s){
            return !s.empty();
        }) |
        publish() |
        ref_count();

```

Now create a stream that emits 0 when a string ending in `\r\n` occurs.

```cpp

    // filter to last string in each line
    auto close$ = string$ |
        filter(isEndOfTweet) |
        rxo::map([](const string&){return 0;});

```

Now use `close$` to signal the begin and end of a window. This will produce a window for each set of strings that make up one json document.

> `start_with()` opens the first window, after which each close also opens a new window.

```cpp

    // group strings by line
    auto linewindow$ = string$ |
        window_toggle(close$ | start_with(0), [=](int){return close$;});

```

Each window will have 1 or more strings. Now reduce each window to a single string.

> `sum()` concatenates the strings using `operator+()`. `start_with()` is required to prime the `sum()`, without it there would be an exception for empty windows.

```cpp

    // reduce the strings for a line into one string
    auto line$ = linewindow$ |
        flat_map([](const observable<string>& w) {
            return w | start_with<string>("") | sum();
        });

```

Now filter out strings that are empty or not properly delimited and json parse the remaining strings.

```cpp

    return line$ |
        filter([](const string& s){
            return s.size() > 2 && s.find_first_not_of("\r\n") != string::npos;
        }) | 
        observe_on(tweetthread) |
        rxo::map([](const string& line){
            return make_shared<const json>(json::parse(line));
        });

```

__the complete function__

```cpp

inline auto parsetweets(observe_on_one_worker tweetthread) 
    -> function<observable<shared_ptr<const json>>(observable<string>)> {
    return [=](observable<string> chunk$){
        // create strings split on \r
        auto string$ = chunk$ |
            concat_map([](const string& s){
                auto splits = split(s, "\r\n");
                return iterate(move(splits));
            }) |
            filter([](const string& s){
                return !s.empty();
            }) |
            publish() |
            ref_count();

        // filter to last string in each line
        auto close$ = string$ |
            filter(isEndOfTweet) |
            rxo::map([](const string&){return 0;});

        // group strings by line
        auto linewindow$ = string$ |
            window_toggle(close$ | start_with(0), [=](int){return close$;});

        // reduce the strings for a line into one string
        auto line$ = linewindow$ |
            flat_map([](const observable<string>& w) {
                return w | start_with<string>("") | sum();
            });

        return line$ |
            filter([](const string& s){
                return s.size() > 2 && 
                    s.find_first_not_of("\r\n") != string::npos;
            }) | 
            observe_on(tweetthread) |
            rxo::map([](const string& line){
                return make_shared<const json>(json::parse(line));
            });
    };
}

```

## algorithms for the win!
This is a fairly complex problem. We have chunks of text arriving over time and they need to be parsed so that json fragments that span chunks are stiched back together and multiple json documents in one chunk are split up. Writing all these algorithms in the raw using promises or callbacks would be a mess of code. Some simple algorithms can be composed to give the functionality with much less chance for confusion or error.

## up next
[Flux Architecture]({{ site.baseurl }}{% post_url 2016-12-06-flux_architecture_in_rxcpp %})

## more on this application
[Realtime analysis using the twitter stream API]({{ site.baseurl }}{% post_url 2016-12-04-realtime_analysis_using_the_twitter_stream_api %}) 
