---
title: Flavours of Reflection
subtitle: Comparing C++26 reflection with similar capabilities in other programming languages
layout: post
---

Reflection is imminent.  It is set to be part of C++26, and it is perhaps the most anticipated feature accepted into this version of the standard.  Lots of implementation work has already been done for [Clang](https://github.com/bloomberg/clang-p2996/tree/p2996) and [GCC](https://gcc.gnu.org/pipermail/gcc-patches/2025-November/700733.html) --- they pretty much already work (both are on Compiler Explorer), and it hopefully won't be too long before they get merged into their respective compilers.

Many programming languages already provide some reflection capabilities, so some people may think that C++ is simply playing catch-up.  However, C++'s reflection feature does differ in important ways.  This blog post explores the existing reflection capabilities of Python, Java, C#, and Rust, comparing them with one another as well as with the flavour of reflection slated for C++26.

In the rest of this blog post, the knowledge of C++ fundamentals is assumed (but not for the other programming languages, where non-commonsensical code will be explained).

## The example

Let's use a simple example that one would probably come across in a reflection tutorial --- printing the member field names and values of a struct instance:

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

Implementing the `object_to_string` function requires us to somehow iterate the member fields of the struct instance, and stringify and print out their names (e.g. "a") and values (e.g. 42).  The ability for code to inspect itself (in this case, the `object_to_string` function being able to inspect the member field names and values of the given struct instance, and then print them out) is known as reflection.

We'll look at how the `object_to_string` function may be implemented in various programming languages.

## Python

The Python equivalent looks something like this:

```python
class MyStruct:
    def __init__(self, a, b, c):
        self.a = a
        self.b = b
        self.c = c

def object_to_string(obj):
    pass
    # how do we implement this?

def main():
    obj = MyStruct(42, "Hello", 3.14)

    # This should print something like
    # {a: 42, b: Hello, c: 3.14}
    print(object_to_string(obj))
```

Python is meant to be an easily understood programming language that does not aim to be efficient, and therefore it makes two design choices that differ from systems programming languages such as C++:
- Python is dynamically typed
- Both member field names and values are stored in each object instance

The first design choice is more or less a prerequisite for the second, and the second design choice happens to make reflection extremely straightforward and natural in Python (in the sense that only very minimal magic is required to perform reflection).

Contrast these two properties with C++.

C++ is statically typed, which means that variables have types associated with them.  For example, when we write `int x;`, we are declaring a variable named `x` that stores an object of type `int`.  However, in Python, variables do not have types associated with them, and they may store objects of any type.  This extends to struct fields and container elements too --- in Python, anything that you can put a value into does not come with type information; type information comes with the value instead.  This makes it natural for Python to have a way to iterate the fields of "struct" objects --- the iterator would be able to yield the value of each field, without worrying about the type of the yielded values.  (In contrast, iterating the fields of a struct in C++ would be pretty unwieldy, because the reflection library author have to choose an element type that is capable of representing all the values being yielded.  (There are such types in C++, such as `std::any` or `std::variant<typename...>`, but both have drawbacks.))

Python gives us even more.  Not only does Python have a way to iterate the field values of an object, but it also provides a way to iterate field names too.  In Python, each object contains a *dictionary* (approximately a hash map that also remembers insertion order) that maps field names (which are strings) to field values. This dictionary can be obtained via the `__dict__` special attribute, allowing us to implement `object_to_string`:

```python
def object_to_string(obj):
    return "{" + ", ".join([name + ": " + str(value) for name, value in obj.__dict__.items()]) + "}"
```

(Note:  `obj.__dict__` is the dictionary, and calling `.items()` returns a view of key-value pairs, which are then transformed via list comprehension.  `str()` converts any object into a human-readable string, using the `__str__()` method if available, otherwise falling back to a default implementation.)

And there we have it --- reflection in Python!

## Java

A lot of why reflection is so simple in Python lies with the fact that both field names and field values are stored in the object itself and not the type --- the `MyStruct` Python class isn't aware of its member fields and doesn't actually do much. (The class type in Python is more important for member functions (methods), but that's a different thing.)  However, this isn't how it works in most of the common programming languages that care at least somewhat about efficiency (e.g. Java, C#, Rust, and C++).

On the surface, we should immediately notice that storing field names in the object itself requires much more memory (unless optimised away somehow), and may reduce execution speed.  However, this is not the full reason --- to properly reap the benefits of static typing, structs are designed to be schemas that instances (objects) of the struct adhere to.  What this means is that in such languages, the struct definition tells us the field names and types (i.e. how the struct is laid out), and any instance created must conform to that layout.  This allows the type checker to, for example, figure out that `obj.a` has type `int` if it already knows that `obj` has type `MyStruct`.

(Note:  Python does have optional type hinting, but this is primarily for development tools and is not enforced by the interpreter.  However, even so, they had to define a syntax to describe the field names and types in the class itself, because this information is so crucial to making type checking work properly.)

Furthermore, because the type of each variable is known at compile time (with some amount of polymorphism, if we want to be pedantic), there is generally no need to iterate fields, because the programmer (and the type checker) will already know the available fields.  I would argue that iterating fields, even if possible, in a statically typed programming language, should be highly discouraged, because (at least in general) the type checker would lose the ability to figure out the type of each field yielded by the iterator.

Well, unless we really have a legitimate reason to iterate fields (like implementing the `object_to_string` function, where we want to implement something that would generally work with objects of any type).  Then, just maybe, we might decide to sacrifice some type checking for the ability to iterate them.

The Java code looks like this:

```java
public class MyStruct {
    public int a;
    public String b;
    public double c;

    public MyStruct(int a, String b, double c) {
        this.a = a;
        this.b = b;
        this.c = c;
    }
}

class Main {
    static <T> String object_to_string(T obj) {
        // How do we implement this?
    }

    public static void main(String[] args) {
        MyStruct obj = new MyStruct(42, "Hello", 3.14);
        
        // This should print something like
        // {a: 42, b: Hello, c: 3.14}
        System.out.println(object_to_string(obj));
    }
}
```

As discussed earlier, to get the field names and field values, one has to first look into the class definition (the schema) to get the field names and figure out where they are located within an instance of this class, and then look at the instance to get those field values.

So how does one get the class definition? 

Java (and so does C#) has a runtime environment, which is a piece of software that runs Java bytecode.  A Java compiler compiles Java source code into Java bytecode, which is a platform-agnostic intermediate representation.  The Java bytecode is then packaged and shipped to users, and users use the Java Runtime Environment (JRE) to run the Java bytecode.  (For efficiency, the runtime environment usually JIT-compiles the bytecode into machine code, but for the purposes of this discussion, it makes no difference whether the bytecode is interpreted or JIT-compiled.)

It turns out that Java bytecode contains class definitions, including all member fields and member functions (methods).  The runtime environment loads the bytecode, and therefore it, too, knows about member fields.  To expose these class definitions to the running program, Java comes with a reflection library to inspect these class definitions at runtime:

```java
static <T> String object_to_string(T obj) {
    // Get the class definition of the given object
    Class<?> class_def = obj.getClass();

    // A temporary container for the stringified field names and values
    ArrayList<String> strings = new ArrayList<>();

    // Iterate the field definitions in the class definition
    for (Field field_def : class_def.getDeclaredFields()) {
        // Ignore the static fields
        if (Modifier.isStatic(field_def.getModifiers())) {
            continue;
        }
        Object value;
        try {
            // Get the field value of the given instance
            value = field_def.get(obj);
        } catch (IllegalAccessException e) {
            // Accessing a private/protected field is not allowed, and we will get this exception if we do so
            continue;
        }
        // Get the field name and the stringified field value
        strings.add(field_def.getName() + ": " + value.toString());
    }

    // Return the formatted string
    return "{" + String.join(", ", strings) + "}";
}
```

The reflection magic that makes this possible consists of three complementary parts:
1. *Entering reflection space* --- Every object comes with a [`getClass()`](https://docs.oracle.com/javase/8/docs/api/java/lang/Object.html#getClass--) method that gets its class definition as a regular object
2. *Traversing reflection space* --- With the class definition, one can extract useful information from it, in this case getting its field definitions with [`getDeclaredFields()`](https://docs.oracle.com/javase/8/docs/api/java/lang/Class.html#getDeclaredFields--)
3. *Leaving reflection space* --- With field definitions, we can get the corresponding field values of the given object with the [`get()`](https://docs.oracle.com/javase/8/docs/api/java/lang/reflect/Field.html#get-java.lang.Object-) method

While parts 2 and 3 may not be necessary for certain use cases, all three parts are fundamental enough that any proper reflection library ought to have some form of each of these parts.

Note:  There is a slight deficiency in the code --- `getDeclaredFields()` returns the member fields in an arbitrary order, which may not necessarily be the declaration order.  It isn't possible to determine the declaration order in Java.  Furthermore, since Java does not allow user code to perform layout-dependent operations (such as the equivalent of `reinterpret_cast` or `offsetof`), it's possible for a conforming Java compiler to reorder member fields, in which case the runtime environment cannot hope to know the order of member fields in the original source code at all.

### Java nitpicks

We can simplify the Java code using streams, which avoids the temporary ArrayList:

```java
static <T> String object_to_string(T obj) {
    return "{"
        + Arrays.stream(obj.getClass().getDeclaredFields()).flatMap(f -> {
            if (Modifier.isStatic(f.getModifiers())) {
                return Stream.empty();
            }
            try {
                return Stream.of(f.getName() + ": " + f.get(obj).toString());
            } catch (IllegalAccessException e) {
                return Stream.empty();
            }
        }).collect(Collectors.joining(", "))
        + "}";
}
```

The generic type parameter (`T`) is also redundant, as `obj.getClass()` gets type information from the runtime type of the object (all classes automatically inherit from `Object` in Java, and all member functions are virtual (in the C++ sense), so the static type of the variable `obj` doesn't matter), so we can simply use an `Object` parameter:

```java
static String object_to_string(Object obj) {
    return "{"
        + Arrays.stream(obj.getClass().getDeclaredFields()).flatMap(f -> {
            if (Modifier.isStatic(f.getModifiers())) {
                return Stream.empty();
            }
            try {
                return Stream.of(f.getName() + ": " + f.get(obj).toString());
            } catch (IllegalAccessException e) {
                return Stream.empty();
            }
        }).collect(Collectors.joining(", "))
        + "}";
}
```

## C#

C#'s reflection is very similar to that of Java, owing to the fact that both are object-oriented programming languages built on classes (i.e. schemas) and use a runtime environment that can be queried using a built-in reflection library.

The C# code looks like this:

```csharp
public struct MyStruct {
    public int a;
    public string b;
    public double c;

    public MyStruct(int a, string b, double c) {
        this.a = a;
        this.b = b;
        this.c = c;
    }
}

public class Program {
    static string object_to_string<T>(T obj) {
        // How do we implement this?
    }

    public static void Main(string[] args) {
        MyStruct obj = new MyStruct(42, "Hello", 3.14);
        // This should print something like
        // {a: 42, b: Hello, c: 3.14}
        Console.WriteLine(object_to_string(obj));
    }
}
```

The implementation of `object_to_string` is really similar to Java and doesn't warrant any additional explanation:

```csharp
static string object_to_string<T>(T obj) {
    // Get the struct definition of the given object
    Type struct_def = obj.GetType();
    
    // A temporary container for the stringified field names and values
    List<string> strings = new();
    
    // Iterate all instance fields (i.e. non-static fields)
    foreach (FieldInfo field_def in struct_def.GetFields(BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance)) {
        strings.Add(field_def.Name + ": " + field_def.GetValue(obj).ToString());
    }
    
    // Return the formatted string
    return "{" + String.Join(", ", strings) + "}";
}
```

### C# nitpicks

Similar to Java, we can simplify the C# code using LINQ and removing the generic type parameter:

```csharp
static string object_to_string(Object obj) {
    return "{"
        + String.Join(", ",
            obj.GetType()
               .GetFields(BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance)
               .Select(f => f.Name + ": " + f.GetValue(obj)))
        + "}";
}
```

However, unlike Java, C# does not type-erase generics.  This allows the following to work in C#:

```csharp
static string object_to_string<T>(T obj) {
    return "{"
        + String.Join(", ",
            typeof(T)
                .GetFields(BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance)
                .Select(f => f.Name + ": " + f.GetValue(obj)))
        + "}";
}
```

## C++

Let's look at the C# implementation again:

```csharp
static string object_to_string<T>(T obj) {
    // Get the struct definition of the given object
    Type struct_def = obj.GetType();
    
    // A temporary container for the stringified field names and values
    List<string> strings = new();
    
    // Iterate all instance fields (i.e. non-static fields)
    foreach (FieldInfo field_def in struct_def.GetFields(BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance)) {
        strings.Add(field_def.Name + ": " + field_def.GetValue(obj).ToString());
    }
    
    // Return the formatted string
    return "{" + String.Join(", ", strings) + "}";
}
```

We used reflection because we wanted to write the function once, but have it work for any given type.  For example, when `T = MyStruct`, we wanted the function to be equivalent to the following function:

```csharp
static string object_to_string_MyStruct(MyStruct obj) {
    return "{a: " + obj.a.ToString() + ", b: " + obj.b.ToString() + ", c: " + obj.c.ToString() + "}";
}
```

However, if we were to profile the two versions, we would find that the generic version is much slower.  This is because, among other things, the generic function had to query the type definition and iterate its fields, while the specialised version did not.  (On [Compiler Explorer](https://csharp.godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIFCJsQAOpJfWQE8Ayo3QBhVLQCuLBnucAZPAZMADkvACNMYhAAZgAOUgtUBUIHBjdPbz1E5PsBAKDQlgiouOtMW1yGIQJzAnSvHy4yitTq2vyQ8MiY%2BIUa4jr3Br0%2B9sDOou64gEprVA9iZHYOD2SjAGohAE8%2BzBYAUmiAIX2NAEFVwOBNnYI9gDp0loEFe4BxRki8ZEOT88uNttdix7gARPBMYAMJL2Ey/U4XNbXIF3EEAJUwVGeDHh5wRFg8YVo33WfWIHjs6wAsltquTKfsAOx/M7rNnrAlEkmBAjrJi41nsznE5CkgjEK7rMIC9kcwki9boeZEzDrH7HBEI2XCkk0ukUggQHl80hiiUbMKmpXy1XIabrJks2WyggIPCvJgO6Kgvky51s13u%2B5hL0%2B6Ua87%2BgNu16iw4%2B9VO9lM0GaxmpvHnHWi5AGBQKdbKYioYDEVgO5la9mjWFmyWoMLaTB2AD6RBbZKuhxcABVDtgID31g3tPbHVX/QB6SfrD6812qskGxWYwIpATDqjrBfrYB4ABujGHjebBAnzp7Wwsi/FBpbWC38eP2nemAIl%2BvEGmfv959l0/WVlUUSMtiC2NVZCYcZiHWfgYJ3TsjDwGhMHQWC8HKNDmDYAtDDQ/cxA8TAFD/dkAj6btEOAfs6yMAsnyCAB3L8f2dUi2QAgBJO4yzuPlaFodZAlGBglnQzCCyNe5MHudZoQYABaGsSRQ2h0AUb9IyjODMCYZAEHWCAADEMLUziGH4cS1PvTEhIYM07wfV8CBMiSICOQJ8CMIyDGAV5lHlEkmRcdYPIYLzgB8yFXmCAQAq5ONGRCsKIqivz7nMkSlmmMdKy0qNq3FK5XjOdB0GoUz0Bsqh7mCVhVX2AAmI4HUaxqQFaxrWpa1SqqcucADVCMwCAR2me4e1QOkri/b8I0FX903Yh18udACMQIBZ7J3OCWCYAg7jQqjluIN8ts68c2u6zYiqMe4AClUECCAmsa01XtNKiNOu16U1%2B%2BbZT%2BzMFrFfaSSo59TzbVAO1u4AWz1W87AgRH6V5MaKyTf1Ts24h7N%2B5kmA617rpHe4mAmqa4a/H62tNMJiaupqWrJsJKemowaeZzrTWQRmuu5snkHZ6mx2ai6lra1igbOc9s1B2t9yetCqSghgIGOgBWE5NZ9cw/NyrHnVR5cR1DOTMEY6laSRw1JDezqAAlyloVAPvWaJ7ikOajf/GcexjUkEHmNSOXNXkFFQNhAw2YkAGtMGWgDHSJ9Z7fpjrnYE1BeY6z2pBlgq4MM408HNjRfiEr0Qs1yvmeZvBDeWwG8pBgrZWqVALEY/b9LFLue4IPun077ve4Qe42gGYJLZYgH2/9CHToUDxaF5J8Ryh9sqNGxsfebqM%2BgH8fJ6ICw599hf1jcBhI/oe4AHUJTuDoXra4BPglUVOuuo%2Bx6Hie2ADAWCUOgSmNRaB1WhEoNA4V6LixJog8Wy9V4EH3qtAqhcr7jgwVfUeg8%2B5/wIQZEeZ9iGn1qDPZi6C25XyXsRVB5tN6tm3nDBGNs0a71HKxK%2B/d/76VPl3C%2BB9/Q3zvjJJ%2BhBMCv1egoa8yBkJBSZuLIhJ8gFMBAahcBYgoFJGbAIdStMBbKJaigteNDeFYMWhmEGMsUwcFmLQTgmteA%2BA4FoUgqBOAuCatEUk8xFgNUatEHgpACCaAcbMOOIBJAAE57ixMSUk5JyT9CcEkK4iJnjOC8AUCADQYSImzDgLAJAmBVDNg8EQMgFAID6wUMoQw5QhDB0Ym40JaAWAWDoGDOKTTaAtNQG0rJnTun0CiEqAgQQCCxI0CYMwlg0CnVzGQUZdBIh1TYDk0gazxkAHkqmDOGe43g5TmxnGIH5bZZzkDVC8ts/gggRBiHYFIGQghFAqHUCc0gugmgGCMCgUw5grDEjCHkyAswu6VDyRwBSCkEDlAsLwVAh5iASiwBCr8pBySCDwGwSa7gsWzEjgsJYIwCBeUaUEAZrT2m8EYmWCwnAeCOOcZkn5XiODYAqcgKpJB1i1WwEOeJGhr4kEwC4PwaJDI%2BOCaaXAhABW%2BK4NMXg4STk5VINEuJCSUn6sSWkjgGTSBuI8Vy3J%2BTCmarZRwRqHLzXbI1VoLVaLkiOEkEAA), it took 2100 ns for the generic version and 1100 ns for the specialised version.)  However, there is nothing about the given task that actually requires us to do the extra work at runtime --- we already know the type of `T` (and therefore all its member fields) at compile time, and so a sufficiently smart compiler (AOT in C++, or JIT in C#) should be theoretically capable of emitting code that is identical to the specialised version.  This is exactly what C++'s new reflection feature bakes into the language --- reflection is performed fully at compile time, and so the compiled code will be effectively identical to the specialised version, making the generic function a zero-cost abstraction!

Leveraging on existing compile time computation support (i.e. `constexpr` and friends), reflection in C++ --- all the three "complementary parts" described above --- is performed at compilation time.  This is not because of a smart optimiser, but because it is specified to require it (much like how a non-type template parameter needs to be instantiated with a template argument that is a constant expression).

This is how we might implement the `object_to_string` function in C++:

```cpp
template <typename T>
std::string object_to_string(const T& obj) {
    // Get the reflection (struct definition) of T
    constexpr std::meta::info struct_def = ^^T;

    // Get the non-static fields at compile time
    // - nonstatic_data_members_of(struct_def, unchecked()) is a consteval function that returns a std::vector<std::meta::info> containing the non-member fields (regardless of access modifier)
    // - define_static_array() is a consteval function converts a constexpr std::vector to a constexpr array of the correct size
    constexpr auto data_members = std::define_static_array(std::meta::nonstatic_data_members_of(struct_def, std::meta::access_context::unchecked()));

    // A temporary container for the stringified field names and values
    std::vector<std::string> strings;

    // Iterate all non-static fields at compile time
    // - "template for" iterates the constexpr array but keeps the loop variable constexpr (i.e. like std::apply)
    // - stringify() is a custom helper function written to convert any object to a string or string_view, because C++ does not standardise such a facility
    template for (constexpr std::meta::info field_def : data_members) {
        strings.push_back(""s + std::meta::identifier_of(field_def) + ": " + stringify(obj.[:field_def:]));
    }

    // Return the formatted string
    return "{" + (strings | std::views::join_with(", "sv) | std::ranges::to<std::string>()) + "}";
}
```

The reflection magic that makes this possible consists of three complementary parts, as with the Java and C# versions:
1. *Entering reflection space* --- `^^T` gets the *reflection* of the type `T` (`^^T` is essentially a handle to the class definition of `T`)
2. *Traversing reflection space* --- With the reflection of `T`, we can get its member fields with `std::meta::nonstatic_data_members_of`
3. *Leaving reflection space* --- From each member field, we can get the corresponding field value for the given object `obj` with `obj.[:field_def:]` (`[:…:]` is known as a *splicer*)

The difference is that all three parts are evaluated at compile time, so after constant evaluation, the code will look something like this:

```cpp
template <>
std::string object_to_string<MyStruct>(const MyStruct& obj) {
    std::vector<std::string> strings;

    strings.push_back(""s + "a"sv + ": " + stringify(obj.a));
    strings.push_back(""s + "b"sv + ": " + stringify(obj.b));
    strings.push_back(""s + "c"sv + ": " + stringify(obj.c));

    return "{" + (strings | std::views::join_with(", "sv) | std::ranges::to<std::string>()) + "}";
}
```

All the reflection code evaporates away after constant evaluation, and therefore will never be part of the final executable binary (regardless of compiler optimisations)!  This is the key difference between reflection in C++ and that in C# and Java --- using reflection in C++ does not slow down your code at runtime at all, as compared to non-reflection code.

### Reflection and constant evaluation

Notice that this "constexpr reflection", as designed for C++, only works because C++ has expressive constant evaluation (via `constexpr` and `consteval`) baked into the language --- it is possible to perform arbitrary computations at compile time, or ensure that a variable must contain a value known at compile time (i.e. a `constexpr` variable).  `std::meta::info` is also specified to be a *consteval-only* type, which means that it is only allowed to exist in compile time code; as a result of this, all the functions that operate on `std::meta::info` (e.g. `std::meta::nonstatic_data_members_of` and `std::meta::identifier_of`) are `consteval` functions.

### Reflection without a runtime environment

C++ does not have a runtime environment like Java and C# does, if C++ were to support runtime reflection, it would need to embed reflection information in the binary itself.  Storing all class definitions in the binary would be inefficient, because most programs that use reflection are likely to only reflect on a few specific types or type hierarchies.  Furthermore, C++ programs generally have more functions and structs than equivalent Java and C# programs, thus increasing the amount of extra metadata required.  These reasons make theoretical runtime reflection support in C++ somewhat problematic, meaning that there perhaps weren't really any good alternatives to compile time reflection in C++.

### C++ nitpicks

The C++ implementation of `object_to_string`, after constant evaluation, still isn't as performant as handwritten code for a specific struct type.

If we only needed to serialise `MyStruct` objects, we could have written this instead, which would be a little faster by eliminating the temporary `std::vector<std::string>`, and pre-concatenating a few strings:

```cpp
template <>
std::string object_to_string<MyStruct>(const MyStruct& obj) {
    return "{a: " + stringify(obj.a) + ", b: " + stringify(obj.b) + ", c: " + stringify(obj.c) + "}";
}
```

To eliminate the temporary `std::vector<std::string>`, we could append directly to the string that will be returned:

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

There's a branch on `is_first` that isn't an `if constexpr`, but once the template for is unrolled, it should be quite easy for a compiler to figure out that the condition is known at compile time, allowing the compiled code to look something like this:

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

This eliminates the temporary `std::vector<std::string>`, resulting in something much closer to the ideal.  It's still not quite exactly equivalent, but *maybe* it won't be too hard for the compiler to figure it out and emit efficient code here.  (It's possible to rewrite the implementation of `object_to_string` to make it exactly equivalent, but that's a discussion for another time.)

## Rust

Well, I lied.  Rust does not have reflection at all.

However, Rust has features that allow it to do many of the things typically done via reflection (and even some things that can't yet be done in C++, Java, or C#), making it look like it has reflection.

This is the equivalent code in Rust:

```rust
struct MyStruct {
    a: i32,
    b: String,
    c: f64,
}

fn object_to_string<T>(obj: &T) -> String {
    // How do we implement this?
}

fn main() {
    let obj = MyStruct{a: 42, b: "Hello".to_string(), c: 3.14};
    
    // This should print something like
    // {a: 42, b: Hello, c: 3.14}
    println!("{}", object_to_string(&obj));
}
```

The standard way to implement `object_to_string` is to use the [Serde](https://serde.rs/) library.  While not part of the Rust standard library, it is so widely used that it has become a de-facto standard.  (The Rust ecosystem makes it really easy to add a library dependency, resulting in a number of third party libraries becoming de-facto standards.)

To use the Serde library, we add a `derive` macro to the struct definition:

```rust
#[derive(Serialize)]
struct MyStruct {
    a: i32,
    b: String,
    c: f64,
}
```

This generates an implementation of the [`serde::Serialize`](https://docs.rs/serde/latest/serde/ser/trait.Serialize.html) trait for our struct.  (A Rust *trait* is approximately a C++ *concept* in that it is a formally specified interface which types may satisfy (with a few differences, but they are not relevant here).)

This gives us the ability to call `serde_json::to_string` on the object, which serialises the object into a string:

```rust
fn object_to_string<T: Serialize>(obj: &T) -> String {
    serde_json::to_string(obj).unwrap()
}
```

The magic happens in the `derive` macro.  The `#[derive(Serialize)]` syntax on the struct definition invokes what is known as a *procedural macro* in Rust (specifically a *derive procedural macro*).  It essentially processes the struct definition it is applied to and emits extra code for it.  The struct definition itself cannot be modified by a derive procedural macro, but it may inspect the struct definition and produce arbitrary code (implemented in the procedural macro).  The `Serialize` procedural macro emits functions that describe the fields of the struct.  These functions make up the implementation of the `serde::Serialize` trait, and they are used by `serde_json::to_string` to produce a string representation of the field names and values of the struct.  The code injection occurs fully at compilation time, making this seem like compile time reflection.

We won't look at exactly how to implement the `Serialize` procedural macro, but at a high level, one would write a function that consumes a token stream (of program tokens after lexing) and produces another token stream:

```rust
#[proc_macro_derive(Serialize)]
pub fn derive_serialize(tokens: TokenStream) -> TokenStream {
    // consume the tokens and produce more tokens
}
```

This function would parse the tokens making up the struct, figure out what the field types and names are, and then emit the appropriate functions.  There are libraries to make this a little easier, but fundamentally, procedural macros transform a bunch of tokens to another.

This works (more or less), and all is good.

Except... how is this different from reflection in other programming languages?

Reflection is about making queries about a given thing in the program.  In C++, we called `std::meta::nonstatic_data_members_of` to ask the compiler to tell us the list of non-static data members of some struct type.  We also did a similar thing in C# and Java, but instead of asking the compiler, we were asking the runtime environment.  There is no such facility in Rust.  This means that what the procedural macro knows about is limited to the tokens in the struct definition.

A simple tweak to our struct makes this limitation obvious:

```rust
#[derive(Serialize)]
struct MyStructV2 {
    a: i32,
    b: String,
    c: f64,
    more: MoreStuff, // an inner struct defined somewhere else, which we should recursively serialise
}
```

How would the procedural macro know how to serialise `MoreStuff`?  It can't, because all it sees is a token containing the identifier `MoreStuff`, but it has no way to look into the definition of `MoreStuff` to obtain its member fields.  (Unless... we also put `#[derive(Serialize)]` on the definition of `MoreStuff`, so that Serde also generates serialisation code for it.)  Rust does not have the ability to do a "jump to definition".

When we say that Rust does not have reflection, we mean that there is no way to make queries about the program, apart from what can be deduced from the code in the procedural macro itself.  In addition to being unable to jump to the definition of a type, we are also unable to figure out the size of each field in the struct, or if the struct has any methods (since methods in Rust are never written "inline" in the struct definition).

### Rust and metaclasses

*Metaclasses*, in C++, refers to the [P0707](https://wg21.link/p0707) proposal by Herb Sutter.

It essentially proposes being able to add an arbitrary "modifier" on a struct definition (in this case `metafunc`):
```cpp
struct(metafunc) MyStruct {
    /* struct definition goes here */
};
```

The "modifier" is a compile-time function that takes in the reflection of the given struct definition (in this case `MyStruct`) as a `std::meta::info`, and returns code that will be injected in place of the "modifed" struct definition --- essentially a transformation of a struct definition into a bunch of new tokens.

This is almost like what Rust can do with *attribute procedural macros*, which, unlike derive procedural macros, allow the entire struct definition to be modified.  In other words, an attribute procedural macro consumes the token stream making up the struct definition, and replaces it with an arbitrary token stream.

The main difference between the C++ proposal and what we have in Rust is that the transformation in the C++ proposal consumes the reflection of the struct definition, on which we can make queries about information not directly in the struct definition; but the transformation in Rust just consumes tokens, so we are limited to the tokens making up the struct definition itself.  However, maybe this limitation isn't really so much of a problem in real-world use cases --- it does look like most, if not all, of the metafunctions implemented in cppfront (referenced by P0707) can be implemented in Rust today as attribute procedural macros.  So perhaps, C++ is effectively playing catch-up with Rust in this space.

## Conclusion

C++26 reflection is often touted as the first compile-time reflection implementation in any major programming language.  I'm not sure if the "first" part is true, but we can definitely say that C# and Java have true reflection (though they are not compile-time), and Rust has compile-time code generation (though it is not reflection).  And Python definitely has runtime reflection, but it's pretty easy when you don't really care about efficiency or type safety.

I'm definitely looking forward to using reflection in C++.
