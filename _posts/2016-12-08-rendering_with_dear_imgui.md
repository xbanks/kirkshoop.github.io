---
title: Rendering with 'Dear, ImGui'
layout: post
date: 2016-12-08 08:47 PST

---

[![twitter for president-elect](/assets/twitter_analysis_president_elect.gif)](https://www.youtube.com/watch?v=QFcy-jQpvBg)
[youtube](https://www.youtube.com/watch?v=QFcy-jQpvBg)

## Dear, ImGui
[`Dear, ImGui'](https://github.com/ocornut/imgui) is a direct-mode gui library that can be rendered using OpenGL, DirectX and other 3d apis. direct-mode means that everything on screen is re-created every frame. This means that there is no need of a view-model diff/patch library to minimize changes to a retained view - such as react or snabbdom in the browser.

The result is very simple descriptive code to generate the view and easy composition of ux controls into a 3D scene.

## view

Collect the renderers

```cpp

    vector<observable<shared_ptr<Model>>> renderers;

```

Render once per frame, using latest `ViewModel`

> `with_latest_from()` tracks the latest `ViewModel` and calls the `take_at` function to extract the `ViewModel` during each frame.

```cpp

    // render analysis
    renderers.push_back(
        frame$ |
        with_latest_from(rxu::take_at<1>(), viewModel$) |
        // ...
```

Verify that the rendering is on the main thread

```cpp

    auto renderthreadid = this_thread::get_id();
    if (mainthreadid != renderthreadid) {
        cerr << "render on wrong thread!" << endl;
        terminate();
    }

```

Create a Window with a default size and position. When moved and resized the position is stored in `imgui.ini` and that stored value is used the next time the app is opened

> EDIT: This [tweet](https://twitter.com/nlguillemot/status/807152339035557892) caused me to update to the pattern below. The UNWIND handles cleanup on exception. but is `dismiss()`ed when there is no exception. I tried a couple of different patterns, but I do not like any of them.

```cpp

    ImGui::SetNextWindowSize(ImVec2(200,100), ImGuiSetCond_FirstUseEver);
    if (ImGui::Begin("Live Analysis")) {
        RXCPP_UNWIND(End, [](){
            ImGui::End();
        });

        // draw window contents

        End.dismiss();
    }
    ImGui::End();

```

Draw two labels in the window

```cpp

    ImGui::TextWrapped("url: %s", URL.c_str());
    ImGui::Text("Now: %s, Total Tweets: %d", utctextfrom().c_str(), m.total);

```

Create a collapsing section in the window and open it by default

```cpp

    if (ImGui::CollapsingHeader("Tweets Per Minute (windowed by arrival time)", 
        ImGuiTreeNodeFlags_Framed | ImGuiTreeNodeFlags_DefaultOpen))
    {
        // ...
    }

```

Convert the history of overlapping tweets-per-minute groups into a vector of floats using Range-v3.

```cpp

    vector<float> tpm;

    // Range-v3
    tpm = m.tweetsperminute |
        ranges::view::transform([](int count){return static_cast<float>(count);});

```

Plot the floats in a chart to visualize tweets-per-minute over time.

```cpp

    ImVec2 plotextent(ImGui::GetContentRegionAvailWidth(),100);
    ImGui::PlotLines("", &tpm[0], tpm.size(), 0, nullptr, 0.0f, fltmax, plotextent);

```

__the complete expression__

```cpp

    // render analysis
    renderers.push_back(
        frame$ |
        with_latest_from(rxu::take_at<1>(), viewModel$) |
        tap([=](const ViewModel& vm){
            auto renderthreadid = this_thread::get_id();
            if (mainthreadid != renderthreadid) {
                cerr << "render on wrong thread!" << endl;
                terminate();
            }

            auto& m = *vm.m;
            auto& words = vm.words;

            ImGui::SetNextWindowSize(ImVec2(200,100), ImGuiSetCond_FirstUseEver);
            if (ImGui::Begin("Live Analysis")) {
                RXCPP_UNWIND(End, [](){
                    ImGui::End();
                });

                ImGui::TextWrapped("url: %s", URL.c_str());
                ImGui::Text("Now: %s, Total Tweets: %d", utctextfrom().c_str(), m.total);

                // by window
                if (ImGui::CollapsingHeader("Tweets Per Minute (windowed by arrival time)", ImGuiTreeNodeFlags_Framed | ImGuiTreeNodeFlags_DefaultOpen))
                {
                    vector<float> tpm;

                    tpm = m.tweetsperminute |
                        ranges::view::transform([](int count){return static_cast<float>(count);});

                    ImVec2 plotextent(ImGui::GetContentRegionAvailWidth(),100);
                    ImGui::PlotLines("", &tpm[0], tpm.size(), 0, nullptr, 0.0f, fltmax, plotextent);
                }
                End.dismiss();
            }
            ImGui::End();
        }) |
        reportandrepeat());

```

## combine views
rxcpp is lazy. lambda's are lazy. `auto greet = []{cout << "Hello.";}` does nothing until called - `greet();`. All the rxcpp expressions we have defined thus far are dormant, waiting for a call to `subscribe()`.

Since all the expressions have been connected in a graph from rxcurl -> parsetweets -> reducers -> models -> viewmodels -> views, the only `subscribe()` required is to the merged views. This `subscribe()` will propagate to every obervable in the expression. 

```cpp

    // subscribe to everything!
    iterate(renderers) |
        merge(mainthread) |
        subscribe<shared_ptr<Model>>([](const shared_ptr<Model>&){});

```

## main loop
The main loop merely pumps the `mainthread` rxcpp scheduler (named `rl` for runloop) and triggers the renderer `with_latest_from()` expressions by calling `sendframe()`

```cpp

    // main loop
    while(lifetime.is_subscribed()) {
        SDL_Event event;
        while (SDL_PollEvent(&event))
        {
            ImGui_ImplSdlGL3_ProcessEvent(&event);
            if (event.type == SDL_QUIT) {
                lifetime.unsubscribe();
                break;
            }
        }

        if (!lifetime.is_subscribed()) {
            break;
        }

        ImGui_ImplSdlGL3_NewFrame(window);

        while (!rl.empty() && rl.peek().when < rl.now()) {
            rl.dispatch();
        }

        sendframe();

        while (!rl.empty() && rl.peek().when < rl.now()) {
            rl.dispatch();
        }

        // Rendering
        glViewport(0, 0, (int)ImGui::GetIO().DisplaySize.x, (int)ImGui::GetIO().DisplaySize.y);
        glClearColor(clear_color.x, clear_color.y, clear_color.z, clear_color.w);
        glClear(GL_COLOR_BUFFER_BIT);
        ImGui::Render();
        SDL_GL_SwapWindow(window);
    }

```

## up next
Counting Tweets

## more on this application
[Realtime analysis using the twitter stream API]({{ site.baseurl }}{% post_url 2016-12-04-realtime_analysis_using_the_twitter_stream_api %}) 
