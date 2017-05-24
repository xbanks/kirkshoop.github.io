---
title: C++Now 2017 report
layout: post
date: 2017-05-23 22:09 PDT

---

This was my second time at C++Now. I was again amazed by the experience. The schedule allows for deeply technical 90min sessions interleaved with plenty of time for direct interaction with presenters and other attendees.

Jon Kalb, Bryce Lelbach, Jackie Kay and many others worked hard to put on a great conference this year.

# Sessions

Here are some of the sessions that I attended.

## Rust keynote
_Niko Matsakis_ - [sched](https://cppnow2017.sched.com/event/ANuD/rust-hack-without-fear)

I had been watching Rust from afar for years. I have a soft spot for requiring code to be explicit about what is intended so that the compiler can detect code bugs. I have built libraries that attempt the same in C++. I had heard that at the start Rust was too explicit and that this placed too high a burden on the programmer, but that Rust was actively changing to find the best balance. 

Niko talked about Rust's contracts for Ownership and the defaults used and how these also were the foundation for safe concurrency. Niko described how Rust started with a message-passing implementation built into the language to begin with but eventually removed it once the Ownership semantics proved to be a superior means of protecting data in a concurrency context.

Niko described how Rust will automatically apply the [Send](https://doc.rust-lang.org/nomicon/send-and-sync.html) trait to types and how Unsafe code like [Arc](https://doc.rust-lang.org/book/choosing-your-guarantees.html#arct) manually implement the Send trait. This send trait allows the compiler to fail when a type that does not implement Send is being copied to another thread context.

I was quite impressed by this talk and Rust.

## C++11’s Quiet Little Gem: <system_error>
_Charles Bay_ - [sched](https://cppnow2017.sched.com/event/A8IW/c11s-quiet-little-gem-ltsystemerrorgt)

I have been interested in error code handling for a while and have built libraries to make it harder to mishandle errors. I had spent some time looking at system_error in the past and it never made sense to me. I wanted to learn how it was intended to be used.

Charles explained how to extend the library to handle additional error code 'categories'. Charles also explained what `error_condition` is for and that the only thing that should be visible in user code is the `error_code` type.

I was able to use what I learned here to improve my _Errors - forgotten, but not gone_ lightning talk.

## Networking TS Workshop
_Michael Caisse_ - [sched](https://cppnow2017.sched.com/event/A8Ie/networking-ts-workshop-part-1-of-2), [sched](https://cppnow2017.sched.com/event/A8Ie/networking-ts-workshop-part-2-of-2)

This two part workshop covered usage of the current Networking TS proposal. I have long wanted to learn something about ASIO and since the networking TS is based on ASIO I thought that it would be a great way to get introduced.

In the first session, Michael explained how the Networking TS works while trying to avoid the current controversy over the executors included in the library. :)

I was somewhat dismayed to see the read method overloads that attempted to embed crippled algorithms and how those really did not make the code any easier to write. I have always been leary of the focus on polling for reads even when a callback or `future` is used so I was not impressed seeing that pattern in the networking TS.

In the second session Michael passed out USB sticks with the networking TS source and proposal and docs and the examples from the first session. The goal was to build an app that would http POST registration into a raffle for a robot to take home.

Once I had the POST finished I began to work on wrapping async_read_some so that the rxcpp algorithms could be used with the networking TS. I did not complete this during the session, but Michael was helpful and over the next day I managed to get it working. Once I showed this to Michael, who was also running the lightning talks, he asked me if I would give another lightning talk on this.

## Haskell taketh away: limiting side effects for parallel programming
_Ryan Newton_ - [sched](https://cppnow2017.sched.com/event/AQ4h/haskell-taketh-away-limiting-side-effects-for-parallel-programming)

I did not take much away from this talk. I was distracted by the code I was writing for the networking TS.

## (Ab)using C++17
_Jason Turner_ - [sched](https://cppnow2017.sched.com/event/A8IY/abusing-c17)

This was a fun talk for me. I like pushing the compiler hard. Jason provided a few new ways to force the compiler to do more work for me! I will be looking for the slides to extract the code.

## The Mathematical Underpinnings of Promises in C++
_David Sankel_ - [sched](https://cppnow2017.sched.com/event/A8JA/the-mathematical-underpinnings-of-promises-in-c)

David is my primary source for learning about the math used to describe systems of computation. After the talk Jackie posted a link in the Slack channel that explains the [Denotational Semantics](https://www.cs.colorado.edu/~bec/courses/csci5535/reading/densem.pdf) math David used.

Once David covered the math describing a Promise that supported all the operations needed for identity, map, apply and bind he walked through how to implement all these using overloads of `then()`. These methods are not included in the Promise, but David stated that being able to implement them in terms of `.then()` ensured that the Promise interface was complete.

David also showed math for the Operational Semantics of `first()` that resolves to the first promise to be fulfilled. This math was debugged by the audience during the talk.

The code for this Promise is on [github](https://github.com/camio/dpl/tree/master/dplp)

## Postmodern Immutable Data Structures
_Juan Pedro Bolivar Puente_ - [sched](https://cppnow2017.sched.com/event/A8J0/postmodern-immutable-data-structures)

I first saw Juan talk about his transducer library [github](https://github.com/Ableton/atria/tree/master/src/atria/xform), [youtube](https://www.youtube.com/watch?v=vohGJjGxtJQ). The transducer talk was quite good and I had just been looking for good immutable collections in C++ for my twitter app, so I was quite excited to attend this talk. 

Juan did a great job of describing the data structure for the immer immutable vector and exploring the performance of various operations.

The end of the talk was a spectacular demo of a text editor using the immutable vector to store the document. Juan loaded a huge file and then copied a large section of the doc over and over again until a save+load would not be able to fit the document in memory and then used undo to return the document to the original state and show that the dirty indicator cleared. This showed how the vector shared the copied nodes and supported efficient undo and made it easy and efficient to detect change.

I look forward to using immutable collections in my apps!

## Type Based Template Metaprogramming is Not Dead
_Odin Holmes_ - [sched](https://cppnow2017.sched.com/event/A8Ix/type-based-template-metaprogramming-is-not-dead)

Odin's talk was based on his [blog](http://odinthenerd.blogspot.com/2017/03/start-simple-with-conditional-why.html) series that I had been following. Odin showed some great data about how a variety of language features affect compile-time performance. This is great information and the techniques that he showed for building template libraries that use the features that compile the fastest was music to my ears. I learned several important things from this talk that I will apply to my code.

## Call: A Library that Will Change the Way You Think about Function Invocations
_Matt Calabrese_ - [sched](https://cppnow2017.sched.com/event/ARYz/call-a-library-that-will-change-the-way-you-think-about-function-invocations-part-1-of-2), [sched](https://cppnow2017.sched.com/event/A8It/call-a-library-that-will-change-the-way-you-think-about-function-invocations-part-2-of-2)

The Call library is an expression template library that allows the arguments to a function to be composed in ways that the language does not allow. expanding tuples and packs interleaved with fixed arguments without recursion. The Call library is assembled from primitives and is extensible.

Matt showed the code required to achieve certain calls, like de-serialization, without the library and then showed how the same code looked with Call usage. After the case had been made, Matt walked through the implementation showing how the arguments were assembled.

Matt insisted that the library is not ready for use and says that he intends to submit it to boost before taking it through the std process. I hope I do not have to wait long..

## Postmodern C++
_Tony Van Eerd_ - [sched](https://cppnow2017.sched.com/event/A8Iq/postmodern-c)

Tony talked about software development in C++ in a general sense and came at it from the side in a very entertaining talk. Tony seemed to focus on the Postmodern movement but in ways that tied back to real programming advice. 

## Promises in C++: The Universal Glue for Asynchronous Programs
_David Sankel_ - [sched](https://cppnow2017.sched.com/event/A8J9/promises-in-c-the-universal-glue-for-asynchronous-programs)

David's second talk eschewed the math behind his Promise library on [github](https://github.com/camio/dpl/tree/master/dplp). Instead, David walked through code using the library. David's Promise library follows the javascript design. This means that it is structured differently. There is no future type because the Promise is not a future value (Promise does not have a `.get()` only `.then()`). To separate the producer from the consumer the Promise constructor takes a lambda and the constructor calls the lambda passing in the fulfill and reject functions as arguments. These can be saved into an async context and called later to produce the result.

I prefer this pattern to the promise/future pattern, but I still miss the lazyness that would allow richer algorithms like `repeat()`, `retry()`, etc.. I also pointed out during this talk that the library allows errors to be ignored and related how rxcpp exits the process if an error is delivered to an observer that did not supply an on_error implementation. This makes it much harder to have silent errors that result is strange effects far removed from the source of the error.

## Competitive Advantage with D
_Ali Çehreli_ - [sched](https://cppnow2017.sched.com/event/AODJ/competitive-advantage-with-d)

This talk covered the history of D and some of its creators. Ali stated that Andrei Alexandrescu convinced him to leave C++ and use D instead. Ali talked about writing a book about D. Ali talked about D's support for linking to C and C++. Ali talked about the compiler being made open source thanks to Symantec. Ali talked about the GC being optional. Ali talked through a list of apps using D.

At the end of this talk my passion for C++ remained. Rust is far more compelling language than D to me.

## A vision for C++20, and std2 (part 3 of 3)
_Alisdair Meredith_ - [sched](https://cppnow2017.sched.com/event/A8Ic/a-vision-for-c20-and-std2-part-3-of-3)

I attended only this last session and participated in some discussion and voting on std vs std2 for the future standard.

During the discussion someone (I do not know his name) proposed that std should always be the latest and each time a breaking change is made a new namespace added to retain the state before the breaking change. I liked this idea.

I posed the question: 
 
 * would you rather have every dev opening a new cpp file spend the mental energy to pick a version of std?
 * would you rather have a dev maintaining old source code spend the energy to pick an old version of the std the first time she experiences a breaking change in std?

Another person asked me if I use header-only libraries - I said exclusively! ;)
They followed up with the question: ok, what if you try to use a header library that was written to use std and it does not support the latest std? would you be able to edit the header library?
I said, yes - by definition..
I agreed that this would be broken.

The conversation was a bit awkward to continue since we were passing a mic around to keep the comments on the video.

However, I believe that the direction of the versioning is unrelated to this issue of composing any libraries (not just header-only) that use different stdN. With the existing proposal to put new stuff in std2, what happens when a library using std and a library using std2 are used together? I think that once we begin to split up the library to allow breaking changes incompatibilities will be introduced no matter which way stdN is applied (forward or backward).

It was interesting to watch this discussion and voting play out. I look forward to seeing how std and std2 are built.

## The Holy Grail - A Hash Array Mapped Trie for C++
_Phil Nash_ - [sched](https://cppnow2017.sched.com/event/A8Iw/the-holy-grail-a-hash-array-mapped-trie-for-c)

I was looking forward to Phil's talk. He is the author of the excellent Catch test library and I have enjoyed his past talks.

I was even more happy to be there when I finally realized that persistent meant immutable! This is an immutable set which would be perfect to use with the immer immutable vector!

Phil did a great job explaining the data structure and then showing the current performance of various operations on the code in comparison to the std mutable containers. insert was the worst performer at the moment, but Phil outlined ways to improve each of the operations.

Phil also walked through some of the code and solicited feedback on the atomic operations in use to make it concurrency safe.

Two complementary immutable collections @ C++Now 2017 - How marvelous!

## Type-safe Programming
_Jonathan Müller_ - [sched](https://cppnow2017.sched.com/event/A8Is/type-safe-programming)

I was not sure that I would be able to attend the Saturday sessions as I was leaving Saturday. I am so glad I went to Jonathan's talk. 
I have seen some of Jonathan's work online before and I was looking forward to see more.

The premise of this talk was that using the C++ type system to constrain usage patterns to safe patterns at compile-time would make code better. This has been my approach for years. I found this talk very familiar :)

Jonathan talked through the implicit integer conversions in the language and then showed a library that disabled them. Then he showed how to use the library to build types with an explicit subset of operators implemented.

Jonathan contrasted `explicit` constructors with a set of functions to construct and convert and he favored the function set. I mentioned that the function set seemed more explicit than `explicit`.

By the end Jonathan was showing the advantages of defining an `index_t` and `difference_t` for `operator[]` indexing and a `safe_flag` type that I mentioned was just adding lifetime scope to the flag value, just as I had done in my own `unique_error`.

## Nbdl: Generic Library for Managing State Seamlessly Across Network  
_Jason Rice_ - [sched](https://cppnow2017.sched.com/event/A8J8/nbdl-generic-library-for-managing-state-seamlessly-across-network)

In this talk Jason explored a library based on Hana that was used to build the slides for this talk. There were a few pieces of the puzzle missing, but the gist was that the html was assembled by traversing meta-types like div and ul/li

# Me

I had one regular talk and one lightning talk planned for this conference. Michael asked me to present another lightning talk on the networking TS + rxcpp prototype I made after his workshop.

## Errors - forgotten but not gone
[slides](https://kirkshoop.github.io/norawthread/errors.html), [github](https://github.com/kirkshoop/GSL/commit/a8c46e58d318158aec5d9c53236428d0ae39703a)

This talk covered the `unique_error` library I built many years ago to give error values a lifetime. This library uses `fail_fast` to make unhandled error values a thing of the past. 

The code on github is the attempt I made to send it as a PR to the GSL a while ago.

## Networking TS w/Algorithms
[slides](https://kirkshoop.github.io/norawthread/rxnetts.html), [github](https://github.com/kirkshoop/networkingts/blob/master/examples/rx.cpp)

This talk covered the code I wrote during the week to adapt the networking TS client api to work with the rxcpp algorithms.

## No raw std::thread! - Live Tweet Analysis in C++
[slides](https://kirkshoop.github.io/norawthread), [sched](https://cppnow2017.sched.com/event/A8Ik/no-raw-stdthread-live-tweet-analysis-in-c)

In this talk I explored an app I built using rxcpp and how the right set of algorithms can be used to add threads without using primitives like std::thread directly.

# Credits

Thank you to all those involved in putting C++Now together, I enjoyed my time there and left feeling energized and happy!

Thank you to all those who participated in discussion over dinners and in-between sessions.
