---
title: rxcpp - moderncpp events as observables
layout: post
date: 2015-08-04 09:21 PDT


categories: [async, rxcpp, c++,]
---

Kenny Kerr ([blog])(http://kennykerr.ca/), [twitter](https://twitter.com/kennykerr)) has released the excellent [moderncpp](http://moderncpp.com/) library. I wanted to apply rxcpp to handle async in moderncpp. rxcpp is cross-platform and can be applied to any UI event model and I hope to build examples for other c++ UI and Networking libraries as well. 

> I would love to showcase rxcpp usage - so please let me know if you have published code that combines rxcpp with other libraries, especially UI and Networking. 

#moderncpp

The moderncpp library is a complete C++ SDK for the Windows Modern API (WinRT). The moderncpp library is generated from from the winmd files shipped in the windows SDK. Kenny has written about the [improved perf](http://moderncpp.com/2015/04/13/modern-c-as-a-better-compiler/) and [ease-of-use](http://moderncpp.com/2015/04/07/a-classy-type-system-for-modern-c/) that moderncpp provides.

#Sample.Composition

moderncpp has some sample projects. I took one of the sample projects [Sample.Composition](https://github.com/kennykerr/modern/tree/master/10.0.10240.complete/Sample.Composition) and published it with  rxcpp usage. This sample will add a shape to the window on [Ctrl]+PointerPressed and existing shapes can be dragged around and will shift to the top of the z-order when moved. 

![screenshot of Sample.Composition](/assets/modern.sample.composition.png)

The rxcpp changes; removed state for the selected object and visuals collection from the View class, built an expression to add shapes and an expression to move shapes and made the AddVisual and SelectVisual members static.

## integration

The first step was to add the [rxcpp nuget package](http://www.nuget.org/packages/rxcpp/) to the Sample.Composition project. 

After that a header `rx.modern.h` was added to hold the adaptors that are needed to use rxcpp with moderncpp.

###Rx namespace

`rx.modern.h` adds the Rx namespace that includes the major rxcpp namespaces.

```cpp
#pragma once 

#include <rx.hpp>
namespace Rx {
    using namespace rxcpp;
    using namespace rxcpp::sources;
    using namespace rxcpp::operators;
    using namespace rxcpp::subjects;
    using namespace rxcpp::util;
    using namespace rxcpp::schedulers;
}
```

This allows all the components of rxcpp to be introduced with the svelt `Rx::`

###Rx::from_event

`rx.modern.h` also contains the `from_event()` adaptor function that is used to turn moderncpp events into `Rx::observable<Event>`.

```cpp
namespace Rx {
    template<class Event, class Add, class Remove>
    auto from_event(Add&& a, Remove&& r) {
        return create<Event>(
            [=](subscriber<Event> out) {
                auto token = a([=](auto const &, Event const & args) {
                    out.on_next(args);
                });
                out.add([=]() {r(token); });
            })
            .publish()
            .ref_count()
            .as_dynamic();
    }
}
```

`from_event()` is very simple. It uses `Rx::create` to call the supplied add function passing in a function that will call on_next for every event. It also calls the supplied remove function when `unsubscribe()` is called.

`publish()` and `ref_count()` are added to share the same event handler with all the observers subscribed.

`as_dynamic()` forgets the type of the stream (create, publish and ref_count) and returns the simple `Rx::observable<Event>`

###Example

The usage is simple if repetitive..

```cpp
auto pressed = Rx::from_event<PointerEventArgs>(
    [=](auto h) {
        return window.PointerPressed(h);
    },
    [=](auto t) {
        window.PointerPressed(t);
    });
```

However, the future may include builds of moderncpp that have overloads for each event that would result in this simplified usage.

````cpp
auto pressed = window.PointerPressed();
```

##Sample.Composition's App.cpp

The modified sample starts out the same way with includes and namespace references

```cpp
#include "pch.h"

using namespace Modern;

using namespace Windows::ApplicationModel::Core;
using namespace Windows::Foundation;
using namespace Windows::Foundation::Numerics;
using namespace Windows::System;
using namespace Windows::UI;
using namespace Windows::UI::Core;
using namespace Windows::UI::Composition;

static float const BlockSize = 100.0f;

Point PositionOf(PointerEventArgs const & args) {
    return args.CurrentPoint().Position();
}
```

The `PositionOf()` helper was added since it is needed more than once.

### View struct

There is only one member in the View struct after removing the three members that carried state in the original sample.

```cpp
struct View : IFrameworkViewT<View>
{
    CompositionTarget m_target = nullptr;

    void SetWindow(CoreWindow const & window);

    static void AddVisual(VisualCollection& visuals, Point point);
    static Visual SelectVisual(VisualCollection& visuals, Point point);
};
```

###AddVisual and SelectVisual

These were trivial changes that replaced the `this->m_visuals` usage with a visuals parameter and had SelectVisual return the selected Visual instead of setting `this->m_selected`

Once these edits were finished the functions were made static.

###SetWindow member

All the rxcpp usage is in `SetWindow()` the intro is similar, but does not mention `this->m_visuals`.

```cpp
void SetWindow(CoreWindow const & window)
{
    Compositor compositor;
    ContainerVisual root = compositor.CreateContainerVisual();
    m_target = compositor.CreateTargetForCurrentView();
    m_target.Root(root);
```

The events that will be used are defined with `Rx::from_event()`

```cpp
    auto pressed = Rx::from_event<PointerEventArgs>(
        [=](auto h) {
            return window.PointerPressed(h);
        },
        [=](auto t) {
            window.PointerPressed(t);
        });

    auto moved = . . . 

    auto released = . . .
```

visuals is defined to be an `Rx::observable<VisualCollection>` that will send the root.Children() collection to each subscriber once. visuals will send the same item again every time subscribe is called, observables that behave this way are called *cold* observables.

```cpp
    auto visuals = Rx::observable<>::from(root.Children());
```

adds and selects both use the pressed event observable. adds only keeps presses while [Ctrl] is held. selects only keeps presses while [Ctrl] is released.

Each event is converted to just the location where the press occured using PositionOf. This turns the original `Rx::observable<PointerEventArgs>` to `Rx::observable<Point>`

```cpp
    // get the position for a press when ctrl is pressed.
    auto adds = pressed
        .filter([](PointerEventArgs const & args) {
            return args.KeyModifiers() == VirtualKeyModifiers::Control; 
        })
        .map(&PositionOf);

    // get the position for a press when ctrl is NOT pressed.
    auto selects = pressed
        .filter([](PointerEventArgs const & args) {
            return args.KeyModifiers() != VirtualKeyModifiers::Control; 
        })
        .map(&PositionOf);
```

actually adding a new visual to the collection just involves combining the adds and visuals streams and calling AddVisual.

The `subscribe()` is required, without it there is no subscription to adds or visuals so no calls to combin_latest will occur.

```cpp
    // adds visuals
    adds
        .combine_latest(
            [](Point const & position, VisualCollection visuals) {
                View::AddVisual(visuals, position);
                return visuals.Count();
            }, 
            visuals)
        .subscribe();
```

moving an existing visual begins the same way as adding. The adds and visuals streams are combined and SelectVisual is called. This turns the original `Rx::observable<Point>` + `Rx::observable<VisualCollection>`  to `Rx::observable<std::tuple<Point, Visual>>`

A simple filter ignores any presses that did not select a Visual (the Point was not within an existing Visual).

`Rx::apply_to()` returns a function object that stores the supplied lambda. The returned function object is a function that takes a single `std::tuple<>` and calls the stored lambda with each tuple member as a separate parameter. This replaces `std::get<0>(arg)` pollution with nice named function arguments.

```cpp
    // moves selected visuals
    selects
        .combine_latest(
            [](Point const & position, VisualCollection visuals) {
                Visual selected = View::SelectVisual(visuals, position);
                return std::make_tuple(position, selected);
            },
            visuals)
        .filter(Rx::apply_to([](Point const &, Visual const & selected) {return !!selected; }))
```

Now comes the magic.

The `map()` will change `Rx::observable<std::tuple<Point, Visual>>` to `Rx::observable<Rx::observable<std::tuple<Visual, Vector3, Point>>>`

The nested observables returned for each selected visual produce a series of moves for that visual until the pointer is released.

The `merge()` that follows will subscribe to each new nested observable. This changes `Rx::observable<Rx::observable<std::tuple<Visual, Vector3, Point>>>` to `Rx::observable<std::tuple<Visual, Vector3, Point>>`

The return from `map()` is the nested observable. The location of every moved event is used to create a `std::tuple<Visual, Vector3, Point>`. The following `take_until()` will stop listing to moved and released as soon as the pointer is released. It is the subscription in the following `merge()` that starts listening to moved and released for the selected visual.

```cpp
        .map(Rx::apply_to([=](Point const & position, Visual const & selected) {
            . . .
            return moved
                .map(&PositionOf)
                .map([=](Point const & position) { 
                    return std::make_tuple(selected, mouse_offset, position); 
                })
                .take_until(released);
        }))
        .merge()
```

All that remains is to actually subscribe and use the output to change the position of the selected visual

```cpp
        .subscribe(Rx::apply_to([](Visual selected, Vector3 offset, Point position) {
            selected.Offset(Vector3
            {
                position.X + offset.X,
                position.Y + offset.Y
            });
        }));
}
```

#done

The moderncpp library gives a great experience for developing universal windows apps. I am so happy to see it built and released!

Also, rxcpp composes quite well with moderncpp and provides benefits in avoiding mutable state and just like the STL in std, rxcpp provides algorithms in the async domain that can and should be reused rather than rebuilt using raw callbacks and state. 

> Using raw callbacks + state instead of rxcpp's `map()` is equivalent to using a raw loop instead of `std::transform()`

#more to come

There are a lot of additional adaptors to be built for moderncpp. `from_asyncoperation()`, and schedulers for CoreDispatcher and Task would allow rxcpp to be used to compose Network and background Tasks with UI.
