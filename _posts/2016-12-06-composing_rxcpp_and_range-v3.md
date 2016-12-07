---
title: Composing rxcpp and Range-v3
layout: post
date: 2016-12-06
---

[![twitter for president-elect](/assets/twitter_analysis_president_elect.gif)](https://www.youtube.com/watch?v=QFcy-jQpvBg)
[youtube](https://www.youtube.com/watch?v=QFcy-jQpvBg)

```cpp

    auto viewModel$ = model$ |
        // if the processing of the model takes too long, skip until caught up
        filter([=](const shared_ptr<Model>& m){
            return m->timestamp <= mainthread.now();
        }) |
        rxo::map([](shared_ptr<Model>& m){
            return ViewModel{m};
        }) |
        publish() |
        ref_count();

    vector<observable<shared_ptr<Model>>> renderers;

```

## up next

