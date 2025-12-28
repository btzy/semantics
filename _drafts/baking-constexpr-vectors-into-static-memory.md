---
title: Baking Constexpr Strings into Static Memory
subtitle: Compile time string manipulation with C++26 reflection
layout: post
---

Seven (TODO check) papers on reflection have been voted into C++26, but one of them stands out.  One of them does not look like a reflection paper, at least from its title: [Pxxxx: ...](TODO) (TODO).

This blog post walks through an example use case of this new library feature, and then looks why the implementation of this feature requires reflection.  A basic level of familiarity with C++26 reflection is assumed.

## The example

We'll continue with the example from the [previous blog post](TODO) (but reading that post isn't necessary to understand this one, as the relevant portions are reproduced here).

In that post, we wanted to implement the `object_to_string` function, which should print the member field names and values of a struct instance:

```cpp
struct MyStruct {
    int a;
    std::string b;
    double c;
};

template <typename T>
std::string object_to_string(const T& obj) {
    // How do we implement this?
}

int main() {
    MyStruct obj{42, "Hello", 3.14};
    
    // This should print something like
    // {a: 42, b: Hello, c: 3.14}
    std::cout << object_to_string(obj) << std::endl;
}
```

In that post, we ended up with the following solution:

```cpp
template <typename T>
std::string object_to_string(const T& obj) {
    std::string res;
    res += '{';
    bool is_first = true;
    template for (constexpr std::meta::info field_def : std::define_static_array(std::meta::nonstatic_data_members_of(^^T, std::meta::access_context::unchecked()))) {
        if (!is_first) {
            res += ", ";
        } else {
            is_first = false;
        }
        res += std::meta::identifier_of(field_def);
        res += ": ";
        res += stringify(obj.[:field_def:]);
    }
    res += '}';
    return res;
}
```

We then reasoned that the compiler would expand this code into something like the following:

```cpp
template <>
std::string object_to_string<MyStruct>(const MyStruct& obj) {
    std::string res;
    res += '{';
    res += "a";
    res += ": ";
    res += stringify(obj.a);
    res += ", ";
    res += "b";
    res += ": ";
    res += stringify(obj.b);
    res += ", ";
    res += "c";
    res += ": ";
    res += stringify(obj.c);
    res += '}';
    return res;
}
```

However, the expanded code is not optimal.

## The problem

When we say that the expanded code is not optimal, we mean that we could have written something better by hand.  In this case, we could have written a specialisation for `object_to_string` as such:

```cpp
template <>
std::string object_to_string<MyStruct>(const MyStruct& obj) {
    return "{a: " + stringify(obj.a) + ", b: " + stringify(obj.b) + ", c: " + stringify(obj.c) + "}";
}
```

The difference here is that




Sketch:
- Intro
- Current reflection code
- Current generated code
- Expected generated code
- see disassembly to show that they are not identical
- The problem we want to solve
- How we would have dealt with it before reflection, and the issues and difficulties with that
- the magic define_static_string (and define_static_array and define_static_object) to the rescue
- new reflection code
- conclusion
