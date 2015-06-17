---
title: User Defined Literal _idx and operator[] for tuple<>, pair<> and any<>
layout: post
date: 2015-06-16 22:37

---
## accessing tuple members

`std::get<0>(t)` is so verbose. 

`std::tie()` is clever, but requires default constructible types, `std::ignore` to skip members and typing the names twice to declare and then assign. 

`apply()` is susinct, but makes a nested function call which is awkward to use at times. 

## better worlds

I would like to use `t[0]` instead. Implementing `auto operator[](int i) -> decltype(std::get<i>(*this))` does not work because the template parameter to get must be resolved at compile-time but the value of the parameter `i` is not known until runtime. 

It was possible to implement `template<int i> auto operator[](std::integral_constant<int, i>) -> decltype(std::get<i>(*this))` but usage required a finite set of `std::integral_contant<int, . . .>` instances in a namespace that looked like this `using _0 = std::integral_constant<int, 0>;` . The result would be:

```cpp
using constants::ints;
t[_0]
```

Now that User Defined Literals are available in most compilers, the finite list of named int constants can be replaced with a raw literal.

## the _idx Literal
The following code defines; an alias of `std::integral_constant` for the index type, a recursive constexpr function pair to convert a raw literal template pack of char to an int and finally, _idx as a new literal that returns the index type.

```cpp
template<int i>
using index_constant = std::integral_constant<int, i>;

template<class T>
constexpr T make(T acc) { return acc; }
template<class T, char c, char...cn>
constexpr T make(T acc) { return make<int, cn...>((acc * 10) + (c - '0')); }

template<char... cn>
constexpr  index_constant<make<int, cn...>(0)> operator "" _idx () {
    return index_constant<make<int, cn...>(0)>();
}
```

## test by wrapping a value in a type that adds operator[]

Rather than modifiying tuple<>, pair<> and any<>, build an indexer struct that holds a value and implements operator[].

```cpp
template<class T>
struct indexer
{
    template<int i>
    auto operator[](index_constant<i>)
    -> typename std::add_lvalue_reference<decltype(get<i>(T()))>::type {
        return  get<i>(value);
    }
    T value;
};

template<class T>
auto indexable(T&& t) -> indexer<T> {return indexer<T>{std::forward<T>(t)};}
```

## test operator[]

```cpp
int main() {
    auto t = indexable(std::make_tuple(1, '2', "three", 4.0f, 5.0));
    std::cout << t[0_idx] << t[1_idx] << t[2_idx] << t[3_idx] << t[4_idx] << std::endl;

    auto p = indexable(std::make_pair(1, "second"));
    std::cout << p[0_idx] << p[1_idx] << std::endl;
}
```

## play with it

This code is published in biicode and was tested in clang and visual studio 15 RC. Install [biicode](http://www.biicode.com) and then

```sh
bii init
bii open kirkshoop/tupleindex
#OSX
bii buzz
bin/kirkshoop_tupleindex_main
#windows 
bii buzz -G"Visual Studio 14 2015 Win64"
bin\kirkshoop_tupleindex_main.exe
```

