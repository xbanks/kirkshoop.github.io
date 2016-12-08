---
title: Realtime analysis using the twitter stream API
layout: post
date: 2016-12-04
---

[![twitter for president-elect](/assets/twitter_analysis_president_elect.gif)](https://www.youtube.com/watch?v=QFcy-jQpvBg)
[youtube](https://www.youtube.com/watch?v=QFcy-jQpvBg)

## rxcpp
I started working on [rxcpp](https://github.com/Reactive-Extensions/RxCpp) because I want a better way to write gui apps so that they do not block the main thread. I am now able to use rxcpp to build applications that do not block the main thread. I can build apps that use multi-core without writing any calls to `thread`, `mutex` or `condition_variable`. I can easily move work from one thread to another and back with small code changes that are much easier to reason about. I can adopt patterns like the _Flux architecture_ easily. I can easily test code that is working over _values-distributed-in-time_. I can use algorithms from rxcpp to collect _values-distributed-in-time_ into a model of _values-distributed-in-space_ and then use algorithms from [Range-v3](https://github.com/ericniebler/range-v3) to process the model into a View.

## why twitter analysis
I watched a [presentation](https://blog.niallconnaughton.com/2016/10/25/ndc-sydney-talk/) by @nconnaughton - [github](https://github.com/NiallConnaughton/rx-realtime-twitter), [twitter](https://twitter.com/nconnaughton) recently that inspired me. I am currently working in the Azure Machine Learning team and the live stream analysis using Rx was very attractive.

## other c++ libraries
My first task was to research C++ libraries for http requests, oauth, json and a gui. Historically, this has been a big source of pain. C++ libraries have impedence mismatches by differing on error handling, async and resource management patterns. This time I found a set of libraries that did not require a lot of glue work.
I have used [nlohmann json](https://github.com/nlohmann/json) before, so that was an easy pick.
I have been watching [dear, imgui](https://github.com/ocornut/imgui) and have really enjoyed using it.
I wanted to try out [beast](https://github.com/vinniefalco/Beast) and I have tried [c++ net lib](https://github.com/cpp-netlib/cpp-netlib) and [c++ rest sdk](https://github.com/Microsoft/cpprestsdk) before, but I searched and found this [c++ gist that calls the twitter stream api](https://gist.github.com/komasaru/9c78a278f6916548f146). The gist used the curl and oauth libraries, so I used those for this project.

## other resources
I used the [twitter stream etiquette](https://dev.twitter.com/streaming/overview/connecting) docs to code the error handling and this list of [stop words](http://xpo6.com/list-of-english-stop-words/) to omit common english words.

## up next
[calling the twitter stream api]({{ site.baseurl }}{% post_url 2016-12-05-calling_the_twitter_stream_api %})

## other posts
| [calling the twitter stream api]({{ site.baseurl }}{% post_url 2016-12-05-calling_the_twitter_stream_api %})
| [parsing json documents]({{ site.baseurl }}{% post_url 2016-12-05-parsing_json_documents_from_a_stream %}) 
| [Flux Architecture]({{ site.baseurl }}{% post_url 2016-12-06-flux_architecture_in_rxcpp %})
| [Composing rxcpp and Range-v3]({{ site.baseurl }}{% post_url 2016-12-06-composing_rxcpp_and_range-v3 %}) 
| [Rendering with 'Dear, ImGui']({{ site.baseurl }}{% post_url 2016-12-08-rendering_with_dear_imgui %}) 
