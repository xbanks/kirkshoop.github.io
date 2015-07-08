---
title: rxcpp - parsing bytes to lines of text
layout: post
date: 2015-07-07 19:16

---

I answered a question on [StackOverflow](http://stackoverflow.com/questions/31208418/split-iobservablebyte-to-characters-then-to-line/31213161#31213161). The question was for Rx.NET, so that is what I used in the answer. rxcpp has almost every operator that the answer required so I ported the answer to rxcpp and added it as an example.

#the question
given a source that sends chunks of bytes, how to produce a string for each line using `\r` as the end-of-line delimiter?

####given

```sh
[ 66 66 66 66 66 66 66 66 66 66 66 66 66 66 66 66 66 ] 
[ 13 67 67 67 67 67 67 67 67 67 67 67 13 68 68 68 68 ] 
[ 68 68 68 68 13 69 69 69 69 69 69 69 69 69 13 70 70 ] 
[ 70 70 70 70 70 70 70 70 70 70 70 70 70 70 13 71 71 ] 
[ 71 71 71 71 71 71 13 72 72 72 72 72 13 73 73 73 73 ] 
[ 73 73 73 73 73 73 73 13 74 74 74 74 74 74 74 13 75 ] 
[ 75 75 75 75 75 75 75 13 ] 
```

####expect

```sh
BBBBBBBBBBBBBBBBB
CCCCCCCCCCC
DDDDDDDD
EEEEEEEEE
FFFFFFFFFFFFFFFF
GGGGGGGG
HHHHH
IIIIIIIIIII
JJJJJJJ
KKKKKKKK
```

#the answer
There are only three steps in the answer;

  1. split on `\r`
  2. group into lines
  3. reduce each group to one string

The code is on [github](https://github.com/Reactive-Extensions/RxCpp/commit/a156c1abf553c9eb9efdb7654b748461c1b298c8)

####create strings split on `\r`
The first step uses regex and token_iterator to split each vector into N strings. 

```cpp
    auto strings = bytes.
        map([](vector<uint8_t> v){
            string s(v.begin(), v.end());
            regex delim(R"/(\r)/");
            sregex_token_iterator cursor(s.begin(), s.end(), delim, {-1, 0});
            sregex_token_iterator end;
            vector<string> splits(cursor, end);
            return iterate(move(splits));
        }).
        concat();
```

`strings` is now an `observable<std::string>` where each string has either no `\r` or has `back() == '\r'`

The `splits` vector turns the source cold, which is why `concat` works to keep the lines in order - at this stage the order is required to group the lines correctly.

I was quite happy with the ability to use a RAW literal string with `/` as a custom delimiter to make the regex look more natural, but what I was really hoping for was a std user defined literal to allow something like `auto delim = R"/(\r)/"re`

I was also quite happy with the token_iterator allowing control over which parts of the matched/unmatched expression to return. just adding `{-1, 0}` was enough to keep the delimiters in the result.

####group strings by line
Now create a separate observable for each line. In the C# answer I used Window, but here I use group_by with some mutable state to track the current group. rxcpp has window, but does not have the overloads that take observable triggers. I like that this version does not require publish, but I dislike the mutable state.

```cpp
    int group = 0;
    auto linewindows = strings.
        group_by(
            [=](string& s) mutable {
                return s.back() == '\r' ? group++ : group;
            },
            [](string& s) { return move(s);});
```

####reduce the strings for a line into one string
sum is used to add the strings that form each line together. sum will throw if a line is empty, but in this example there are no empty lines. 

```cpp
    auto lines = linewindows.
        map([](grouped_observable<int, string> w){ 
            return w.sum(); 
        }).
        merge();
```

####print result

```cpp
    lines.
        subscribe(println(cout));
```

####test source of byte chunks
To test this I used the following generator (and made sure that it did not produce empty lines)

```cpp
    random_device rd; 
    mt19937 gen(rd());
    uniform_int_distribution<> dist(4, 18);

    // produce byte stream that contains lines of text
    auto bytes = range(1, 10).
        map([&](int i){ 
            return from((uint8_t)('A' + i)).
                repeat(dist(gen)).
                concat(from((uint8_t)'\r'));
        }).
        merge().
        window(17).
        map([](observable<uint8_t> w){ 
            return w.
                reduce(
                    vector<uint8_t>(), 
                    [](vector<uint8_t>& v, uint8_t b){
                        v.push_back(b); 
                        return move(v);
                    }, 
                    [](vector<uint8_t>& v){return move(v);}).
                as_dynamic(); 
        }).
        merge().
        filter([](vector<uint8_t>& v){
            copy(v.begin(), v.end(), ostream_iterator<long>(cout, " "));
            cout << endl; 
            return true;
        });
```

#bonus
rxcpp is becoming more friendly when the global namespace has been polluted with the std namespace (`using namespace std;`). The code in this example works with the following setup.

```cpp
#include "rxcpp/rx.hpp"
using namespace rxcpp;
using namespace rxcpp::sources;
using namespace rxcpp::util;

#include <regex>
#include <random>
using namespace std;
```
