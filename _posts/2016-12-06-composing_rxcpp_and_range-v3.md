---
title: Composing rxcpp and Range-v3
layout: post
date: 2016-12-06 21:31 PST

---

[![twitter for president-elect](/assets/twitter_analysis_president_elect.gif)](https://www.youtube.com/watch?v=QFcy-jQpvBg)
[youtube](https://www.youtube.com/watch?v=QFcy-jQpvBg)

## values-distributed-in-time
[__rxcpp__](https://github.com/Reactive-Extensions/RxCpp) is a set of algorithms that operate on values separated by time. The twitter application is emitting updated `Model`s every 200ms. __rxcpp__ algorithms are a good fit for orchestrating the flow of `Model`s into `ViewModel`s that are then delivered to the __rxcpp__ expression that will render. 

Use __rxcpp__ to convert `Model`s to `ViewModel`s

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

```

## values-distributed-in-space
[__Range-v3__](https://github.com/ericniebler/range-v3) is a set of algorithms that operate on values separated by space. The `ViewModel` constructor receives an instance of the `Model` which is a space that contains only values that have already arrived. While the values in space could be processed by __rxcpp__, __Range-v3__ is a much better fit.

Construct `ViewModel` state from a `Model` using __Range-v3__ algorithms

```cpp

    struct ViewModel
    {
        ViewModel(shared_ptr<Model>& m) : m(m) {

            if (idx >= 0 && idx < int(m->groups.size())) {
                auto& window = m->groups.at(idx);
                auto& group = m->groupedtweets.at(window);

                words = group->words |
                    ranges::view::transform([&](const pair<string, int>& word){
                        return WordCount{word.first, word.second, {}};
                    });

                words |=
                    ranges::action::sort([](const WordCount& l, const WordCount& r){
                        return l.count > r.count;
                    });
            }

            using group_type=pair<milliseconds, float>;

            vector<group_type> groups = m->groupedtweets |
                ranges::view::transform([&](const pair<TimeRange, shared_ptr<TweetGroup>>& group){
                    return make_pair(group.first.begin, static_cast<float>(group.second->tweets.size()));
                });

            groups |=
                ranges::action::sort([](const group_type& l, const group_type& r){
                    return l.first < r.first;
                });

            groupedtpm = groups |
                ranges::view::transform([&](const group_type& group){
                    return group.second;
                });
        }

        shared_ptr<Model> m;

        vector<WordCount> words;
        vector<float> groupedtpm;
    };

```

The result, is that __rxcpp__ orchestrates the calculation of a new `ViewModel` every 200ms and the calculation is done using __Range-v3__!

## up next
Rendering with 'Dear, ImGui'

