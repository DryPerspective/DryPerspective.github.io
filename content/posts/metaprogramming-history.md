---
date: '2026-11-07T17:24:01Z'
draft: false
title: 'A Brief History of Metaprogramming'
summary: "Five examples of how template metaprogramming has evolved from its discovery in (pre-)C++98 to C++26"
tags: ["c++", "c++26", "templates", "metaprogramming", "sfinae", "constexpr", "reflection", "wg21"]
---

At a previous job of mine, our entire codebase and toolchain was exclusively written in C++98 using the classic, late-90s Borland compiler. And it's possible the company had good reason for this. In any case, this meant that I got fairly familiar with the shape of C++ template metaprogramming in its basest form, before the committee added features to take the edge off it. Since you need to be that much more familiar with the intricacies of how the template engine works in order to engineer the solution, it has always been an interesting walk through the black magic on which so much of the world is built.

So let's take that walk - five examples of metaprogramming from the days of C++98 through to our modern world of C++26.

## Setting the Scene: C++98

Template metaprogramming is a field which was discovered rather than designed; and this has several corollaries about the state of metaprogramming in the C++98 days. Templates were still a new feature with a fairly broad implementation divergence and one or two features which left most implementers stumped (see also: why `export` is a full keyword but `import` is a contextual one); the standard itself left much of this area unspecified and underspecified as it had primarily been written with traditional programming in mind; and there was a lot of tribal knowledge floating around where everyone had their own ideas and techniques to achieve the same goals. For the purposes of this post, I'll lean more towards what eventually became the standard tools and traits if they are implementable in plain C++98, but know that wider techniques did exist.

### The Tools You Had

Making metaprogramming work in C++98 wasn't always easy. A (by no means exhaustive) list of the primary tools you were able to use at compile time is:

* Full and partial template specialisations.
* The preprocessor.
* `sizeof()`.
* Integer constant expressions.
* Overload resolution ranking.

And so, the game was typically to coerce a result out of templates' inherent pattern-matching abilities, or to have templates recursively instantiate other templates until a base case could be established. You could have some integer or boolean constant readable and reachable at compile time but its value could only come from the limited set of possible operations. Contrast this with our modern world of C++26 where we have multiple full keywords and language features to express compile-time logic.

### The Tools You Didn't Have

No `constexpr` and friends, no concepts and constraints, not even `decltype`, and a much more limited standard library. Again, this is very much an incomplete list and you didn't have to go far to come across minor and pedantic features to get in your way. For example, consider the case where you want to pass some one-off type to a standard library algorithm (as there were no lambdas in C++98 and the commonly taught best practice was to favour a functor over a function pointer):

```cpp
void partition_enabled_widgets(std::vector<Widget>& vec){
    //Define our oracle for whether a widget is enabled
    struct is_enabled{
        bool operator()(Widget const& widget) const{
            return widget.enabled;
        }
    };

    //And put all the enabled widgets at the front of our list
    std::vector<Widget>::iterator enabled_end =
        std::partition(vec.begin(), vec.end(), is_enabled()); //ERROR
}
```

This error may be surprising to some. Because `is_enabled` is a local type, it has no linkage. And any type without linkage cannot be a template argument. As such, when deduction tries to instantiate `std::partition<std::vector<Widget>::iterator, is_enabled>` the compiler issues an error. You have to move `is_enabled` out of your function (and thus greatly expand the scope of places in code which can reach it) in order for your code to compile. Non-type template parameters were similarly restrictive: a pointer or reference argument had to refer to a function or object with external linkage.

If placing these types and variables outside functions and ensuring that they were externally linked causes concern about cross-TU visibility, C++98 had a solution. In C++98 an anonymous namespace did not grant internal linkage as it does today, but instead granted its members external linkage in an unnameable namespace. As such, types within it could be used as template arguments but were not nameable in other TUs. This was changed in C++11 to grant internal linkage ([CWG1113](https://cplusplus.github.io/CWG/issues/1113.html)), and extend templates to be able to use internally linked variables and local types ([N2657](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2008/n2657.htm)) as template arguments.

This is just one hazard among many, but consider how common it is in code to pass a bespoke callable to an algorithm and you will realise how frequently you could trip up on even the smaller wording restrictions.

But the headline limitation for C++98 metaprogramming is its much more limited SFINAE. The C++98 specification gave a small and discrete list of possible substitution failures which were not an error. If a substitution failure occurred outside of these situations, your compiler was within its rights to issue a hard error:

> * Attempting to create an array with an element type that is `void`, a function type, or a reference type, or attempting to create an array with a size that is zero or negative.
> * Attempting to use a type that is not a class type in a qualified name.
> * Attempting to use a type in the qualifier portion of a qualified name that names a type when that type does not contain the specified member, or if the specified member is not a type where a type is required.
> * Attempting to create a pointer to reference type.
> * Attempting to create a reference to a reference type or a reference to `void`.
> * Attempting to create "pointer to member of T" when T is not a class type.
> * Attempting to perform an invalid conversion in either a template argument expression, or an expression used in the function declaration.
> * Attempting to create a function type in which a parameter has a type of `void`.
> * Attempting to create a cv-qualified function type.

As you can see, this is a list which seems more interested in guarding against accidental cases where a type wouldn't cleanly produce a valid template rather than a tool to manipulate the compiler into branching your code at compile time.

Implementations varied on how doggedly they stuck to this list; but formally these were the only guarantees before [CWG339](https://cplusplus.github.io/CWG/issues/339.html), resolved by [N2634](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2008/n2634.html), expanded it to clarify that any expression in which a substitution failure occurs in the immediate context is not inherently an error. This happened in the C++11 timeframe and is what makes "Substitution Failure Is Not An Error" a broadly accurate term in C++ today.

## 1. A Static Assertion

In C++98, the simplest (and naive) way to engineer a static assertion was to make the `false` case an uninstantiable template:

```cpp
template<bool b>
struct static_assertion; //Declared, not defined

template<>
struct static_assertion<true> {}; //Fully defined

#define STATIC_ASSERT(cond) sizeof(static_assertion<(cond)>)
```

In this case, if the condition is true, then `static_assertion<true>` is chosen, it has a known size, and the macro expands to a statement with no effect. If the condition is false, then you attempt to take the size of an incomplete type, which gives you an error pointing to that line and which likely contains the term `static_assertion`.

This is good, but it can be better. It still litters your code with meaningless statements which don't do anything, and you can only place your static assertions inside functions. So what happens if you want to check, for example, some property of the template arguments on a class template? You *could* put this macro in the function body of the constructor, but the function bodies of member functions of class templates are not instantiated until they're called. This means that you could have an inherently invalid class which you can name in your code and use as a template argument and you would only notice when you attempt to construct one.

Instead, we can push the check into a declaration so that it is checked when the specialisation is instantiated. Let's define the macro as:

```cpp
#define STATIC_ASSERT(cond) typedef char _name_[sizeof(static_assertion<(cond)>)]
```

Where `_name_` is a preprocessor-friendly unique name generator. Usually your preferred way of mashing some prefix onto `__LINE__` and/or the nonstandard extension `__COUNTER__`. We use an array to turn a size into a type, and now we have a static assertion which can exist as a *declaration* rather than just a statement.

Another possibility is to define your macro as:

```cpp
#define STATIC_ASSERT(cond) typedef char _name_[ (cond) ? 1 : -1 ]
```

Where we dispense with the struct and just guarantee that if the condition is false, we attempt to create an array of negative length, which is inherently ill-formed. We avoid using 0 for the false case as some implementations offered zero-length arrays as an extension. This has the advantage of not depending on any template or types, and the disadvantage that the error you get no longer refers to a name `static_assertion`.

In C++11, `static_assert` was added as a keyword, with a mandatory condition and string literal reason which would be added to the diagnostic if the assertion failed. In C++17, the string literal reason was made optional ([N3928](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n3928.pdf)). C++17 also added `if constexpr` which brought in a new, potentially surprising behaviour:

```cpp
template<typename T>
auto widgetise(T in){
    //Ints go one way
    if constexpr(std::is_integral_v<T>){
        return widgetise_the_integrals(in);
    }
    //Floats go another
    else if constexpr(std::is_floating_point_v<T>){
        return widgetise_the_floats(in);
    }
    //Otherwise you used an invalid type
    else{
        static_assert(false, "Only integral and floating point types are permitted");
    }
}
```

One might expect that last `else` block to act as a gutter, only instantiated when `widgetise` is called with a type which is neither integral nor floating point. Unfortunately, this was not the case - the literal `false` does not depend on any template parameter and so when the compiler encounters `static_assert(false)` it eagerly renders the entire program ill-formed regardless of what type `T` is. This was fixed in [CWG2518](https://cplusplus.github.io/CWG/issues/2518.html) for C++23 (as a DR so retroactively a part of older standards), but until then the conventional trick was to get a "dependent false" to delay evaluating the `static_assert` until the template is instantiated:

```cpp
//Elsewhere, in some header
template<typename T>
constexpr inline bool dependent_false = false;

//And your function is now:
template<typename T>
auto widgetise(T in){
    if constexpr(std::is_integral_v<T>){
        return widgetise_the_integrals(in);
    }
    else if constexpr(std::is_floating_point_v<T>){
        return widgetise_the_floats(in);
    }
    else{
        static_assert(dependent_false<T>, "Only integral and floating point types are permitted");
    }
}
```

In C++26, `static_assert` got another update thanks to [P2741](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2741r3.pdf) - the message string is no longer required to be a string literal and instead can be a compile-time-friendly stringlike type. So if you have a `constexpr` string formatter you can now spin up bespoke `static_assert` messages, for example:

```cpp
template<std::size_t N, typename T>
class packed_storage{
    static_assert(sizeof(T) % N == 0,
        my::fmt("Error, type {} (size {}) must have a size divisible by {}", display_string_of(^^T), sizeof(T), N));

    //...
};
```

## 2. Integer Mathematics

Consider a very basic numerical operation, all of the information for which exists in your source code. One case might be finding the greatest common divisor of two numbers to build a value which must be known at compile time (for example, for strongly typed fractions in the vein of `std::ratio`). Were we in runtime code, a simple function to do this might look like:

```cpp
int gcd(int a, int b){
    while(b != 0){
        a = a % b;
        swap(a, b);
    }
    return abs(a);
}
```

We iteratively reduce `b` to the remainder of division between `a` and `b` until we hit our "base case" of no remainder after division, at which point we know we have our greatest common divisor.

In C++98 there are no `constexpr` functions, so we need to encode the value into a type and use the template engine to write a metafunction to get the value. Let's start with a rough outline of what the answer must look like:

```cpp
template<int a, int b>
struct gcd{
    static const int value = ???; //We'll get to what goes here later
};
```

With this structure, the result of the gcd calculation can be `gcd<a, b>::value`; which, due to being an integer constant expression, is accessible at compile time.

But how do we implement this? We can't iteratively expand out templates to match the machinery of the plain function, so we instead need to transform the operation into a recursive one. On each "iteration" all we do is map `a` to `b`, and `b` to `a % b`. The workhorse of our calculation therefore looks like:

```cpp
template<int a, int b>
struct gcd{
    static const int value = gcd<b, a%b>::value;
};
```

This is equivalent to the body of our loop, but we also need to make sure that we stop when `b == 0` to set the base case. This is simple enough - a full or partial class template specialisation will always be chosen over an unspecialised one, so we add the base case with an additional (and trivial) `abs` metafunction:

```cpp
template<int a>
struct gcd<a, 0>{
    static const int value = abs<a>::value;
};
```

And putting it together gives us:

```cpp
template<int a>
struct abs{
    static const int value = a < 0 ? -a : a;
};

template<int a, int b>
struct gcd{
    static const int value = gcd<b, a%b>::value;
};

template<int a>
struct gcd<a, 0>{
    static const int value = abs<a>::value;
};
```

This is likely not production-ready, as `abs<INT_MIN>::value` will overflow and `int` might not be the optimal type, but it serves as a good demonstration.

From here, implementing something resembling `std::ratio` falls out easily. Using an additional `sign` metafunction (which is trivial to implement so left as an exercise to the reader), the template simply becomes:

```cpp
template<int Num, int Denom = 1>
struct ratio{
    STATIC_ASSERT(Denom != 0);
    static const int num = sign<Denom>::value * Num / gcd<Num,Denom>::value;
    static const int den = abs<Denom / gcd<Num, Denom>::value>::value;
    typedef ratio<num, den> type;
};
```

Of note, some code would use `enum { value = ??? };` instead of a `static const` integer variable. This was known as the "enum hack", and had some benefits and drawbacks. It was more portable for a time, as not all compilers supported in-class initialisation of these `static const` members. Enumerators cannot have their addresses taken and do not occupy storage, which means that you never bind a reference to a real object which you then need to link to later (and a plain `static const int` could fail to link without an out-of-class definition). The argument against the enum hack was type safety - enums promote to `int`, so `f(trait::value)` would pick `f(int)` over `f(bool)`; and as the enum is its own type `std::min(0, trait::value)` would fail deduction entirely. For the purposes of consistency with modern-day `static constexpr` variables used by the type traits, this post will stick with `static const` variables.

C++11 gave us `constexpr`, which drastically simplified this code by allowing us to write functions as functions rather than recursively generated types. But there were still limitations in C++11: a `constexpr` function could only contain a single return statement - it could not contain local variables, loops, conditionals, or more complex code. Fortunately, you could still use the ternary operator as a conditional inside that return statement, and you could still recursively call other `constexpr` functions inside the ternary. So, we follow the same recursive pattern we did in C++98 and produce this code:

```cpp
namespace detail{
    constexpr int abs(int x){
        return x < 0 ? -x : x;
    }
}

constexpr int gcd(int a, int b){
    return b == 0 ? detail::abs(a) : gcd(b, a % b);
}
```

C++14 loosened `constexpr` restrictions further ([N3652](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3652.html)) and allowed local variables, loops, and conditionals in `constexpr` functions. As such, our original Euclidean algorithm can now be a plain `constexpr` function:

```cpp
constexpr int gcd(int a, int b){
    while(b != 0){
        a = a % b;
        swap(a, b);
    }
    return abs(a);
}
```

In the case of greatest common divisor specifically, C++17 included a `std::gcd` function in the `<numeric>` header which you should use over handspun approaches.

What is permitted in `constexpr` has only grown with each new standard. C++17 allowed for `constexpr` lambdas, expanded `constexpr` across the standard library, and added language support for compile-time branching in `if constexpr`. C++20 allowed allocation and dynamic polymorphism during constant evaluation (with some restrictions) as well as expanding `constexpr` support to the standard library algorithms. C++23 added more language support in `if consteval` to determine whether execution was presently happening at compile time; and finally C++26 gave us `constexpr` exception throwing, made all the standard library containers `constexpr` friendly (except `std::hive`), allowed placement new and casting from `void*`, not to mention the headline feature of static reflection and introspection entirely at compile time.

Which is to say that `constexpr` machinery has come a long way, but it all started with complex expansions of templates and using types to generate values.

### Aside: Array Sizes

The machinery above is just one technique to do this kind of processing at compile time. It cannot be applied to all operations in the way that you can funnel (almost) everything into a `constexpr` function in more modern C++. Consider the case of finding the size of a C-array from a variable of array type. A metafunction of the style above doesn't help you:

```cpp
template<typename>
struct array_size{};

template<typename T, std::size_t N>
struct array_size<T[N]>{
    static const std::size_t value = N;
};
```

Without `decltype` there is no way to convert a variable into something which can go in between the angle brackets of `array_size<???>::value`. Instead we need to take a different approach to template pattern matching and use an array to find the size of our array:

```cpp
template<typename T, std::size_t N>
char (&array_size_helper(T (&)[N]))[N]; //Declared but not defined

#define ARRAY_SIZE(a) (sizeof(array_size_helper(a)))
```

To decode the syntax, `array_size_helper` is a function which accepts a reference to an array `T[N]` and returns a reference to an array of `char[N]`. It's not possible to simply write it as `char(&)[N] array_size_helper(T(&)[N])` for the same reason you need to parenthesise the reference when passing an array by reference - parsing requirements. Since `sizeof(char)` is required to be `1`, this means that the size of the return type we declare is guaranteed to be `N`.

This approach has limitations - as previously discussed, local types cannot be used as template arguments, so the size of an array of some local type cannot be retrieved by this trick. Equally, in my own experience, not all compilers were able to reflect on the size of the return type of a function as chosen by overload resolution, and so this trick would produce a somewhat apologetic error promising to implement the feature in the future.

By contrast, in C++11 all of this can be replaced by a single function for arrays whether they are of local types or not, and which doesn't need an explainer to understand its syntax:

```cpp
template<typename T, std::size_t N>
constexpr std::size_t size(T(&)[N]){
    return N;
}
```

Which is pretty much what C++17's `std::size` does for arrays. Since we are now dealing with plain functions we can of course overload them so that other containers can have their sizes retrieved just through generic calls to `std::size`.

It's also obligatory to shout out that there is a "simpler" trick to get array size:

```cpp
#define ARRAY_SIZE(a) sizeof(a)/sizeof(a[0])
```

This is not type safe and will compile and produce the wrong answer for a decayed array. And for some baffling reason it's still occasionally taught to beginners even when a type-safe equivalent exists in every C++ standard. Suffice it to say, don't use it.

/Rant

## 3. Function Constraints and Traits

Despite the limitations it remained very possible to implement a lot of what ultimately became the C++ type traits in C++98 without reaching for compiler intrinsics. For example, the implementation of `is_same` remains trivial:

```cpp
template<typename T, typename U>
struct is_same : false_type{};

template<typename T>
struct is_same<T, T> : true_type{};
```

And if we wanted to constrain which types could apply to which functions, we could force a substitution failure and remove them from the overload set with `enable_if`:

```cpp
template<bool b, typename T>
struct enable_if {};

template<typename T>
struct enable_if<true, T>{
    typedef T type;
};
```

But how do we apply this to our function? There is one small hitch. In C++98 function templates could not have default template arguments. It was not possible to add a dummy template argument to the function, default it to a value so the user never has to supply it, and then let SFINAE do its thing. So instead, you could typically make it the return type:

```cpp
template<typename T>
typename enable_if<is_integer<T>::value, int>::type integers_only(T in);
```

Note the requirement to use `typename` on `enable_if<...>::type`, which is a qualified dependent name, but not on `is_integer<T>::value`, which names a value rather than a type.

And for the most part this could work, even if it did make your signatures harder to read and made your function's mangled names much more complex. But what about functions which don't declare a return type, like constructors? You may try to reach for something like this:

```cpp
template<typename T, std::size_t Extent>
class span{
    public:
    //Default ctor only exists if we have a static extent of 0 or a dynamic extent
    span(typename enable_if<(Extent == 0 || Extent == dynamic_extent), bool>::type = true);
    //...
};
```

This may look reasonable - you have a defaulted function parameter which will either substitute or not depending on your `Extent`. But it's not that easy. When you instantiate a class template, the declarations (but not definitions) of all of its non-template member functions are also instantiated. In this case, the type of your `enable_if<...>::type` parameter is determined by eagerly substituting `Extent` in, and if this results in a substitution failure then it is outside of a SFINAE-able context and so results in a hard error.

Unfortunately, the only real way around this was to use a static assertion in the constructor body. This would successfully prevent the function from being called and you lose little compared to traditional SFINAE as you are unlikely to be trying to choose a different, valid constructor via overload resolution in this case.

When C++11 came along with defaulted template arguments for function templates, we got the simple answer to tidy up our constraining code:

```cpp
template<typename T, typename std::enable_if<is_integer<T>::value, bool>::type = true>
int integers_only(T in);

template<typename T, std::size_t Extent>
class span{
    public:
    template<std::size_t E = Extent, typename std::enable_if<(E == 0 || E == dynamic_extent), bool>::type = true>
    span();
};
```

And again, note the requirement to use `typename`. We are not introducing a type template parameter, but a `bool` template parameter which happens to be spelled as a dependent type.

In C++14, the standard library started adding `_t` type aliases such that our SFINAE types could be named in a type-only context where qualification with `typename` is unnecessary:

```cpp
template<typename T, std::enable_if_t<is_integer<T>::value, bool> = true>
int integers_only(T in);
```

No `typename` qualification needed any more.

Until C++20, there was an additional problem to consider when handling multiple constrained overloads of a function. Absent any other factor to affect overload resolution ranking, all constrained overloads had the same priority. This means that in order to prevent an ambiguous call, every overload's constraints had to be made mutually exclusive with every other overload's constraints:

```cpp
//Just integers
template<typename T, std::enable_if_t<is_integer<T>::value && (sizeof(T) != 4), bool> = true>
int f();

//Integers of size 4
template<typename T, std::enable_if_t<is_integer<T>::value && (sizeof(T) == 4), bool> = true>
int f();
```

Which scaled about as well as you'd expect when conditions started stacking up.

All of these were still ways to coerce the compiler into causing a substitution failure in the error case, and in C++20 template constraints were formalised so that you could finally work with the language rather than against it:

```cpp
template<typename T> requires std::integral<T>
int integers_only(T in);

template<typename T, std::size_t Extent>
class span{
    public:
    span() requires (Extent == 0 || Extent == dynamic_extent);
};
```

C++20 concepts also gave us subsumption which, while not without some traps ([shameless plug](https://sub.fail/posts/what-is-a-concept/)), allowed the compiler to automatically order more constrained overloads higher than less constrained ones:

```cpp
//Just integers
template<typename T> requires std::integral<T>
int f();

//Integers of size 4
template<typename T> requires std::integral<T> && (sizeof(T) == 4)
int f();
```

Which is a far cry from needing a 50-character constraint syntax to force the compiler to do your bidding.

### Traits with Expression SFINAE

While it is very possible to engineer a subset of the type traits with plain C++98 template pattern matching, if you were on an implementation which would SFINAE on any expression, a whole new world of traits opened up to you. Let's consider checking if two types, `T` and `U`, are swappable.

In C++ when we say swappable, what we really mean is whether the expression `swap(t, u)` is well-formed. Whatever actually happens when you call that function is its business. To start we need a way to get handles on `t` and `u` in an unevaluated context to attempt substitution. We shouldn't just use `swap(T(), U())` as this imposes a requirement of default constructibility (and results in an rvalue which won't bind to a swap function's parameter type). Instead we want something which resembles modern C++'s `std::declval`. And indeed this is very simple to implement:

```cpp
template<typename T>
T& declval();
```

From here, the next step is to put `swap(declval<T>(), declval<U>())` into an unevaluated context and see if substitution succeeds. We recall from writing our static assertion that our workhorse for this in C++ is the `sizeof()` operator, since it doesn't evaluate its argument but still produces a value. Let's start down that path and consider putting `sizeof(swap(declval<T>(), declval<U>()))` into our code.

But here we have already introduced a problem - what is the return type of your average `swap()` function? It's `void`. `sizeof(void)` is invalid C++ and so will always fail. So we use the comma operator to check our swap expression first, and then some other value which is guaranteed to work second. `sizeof((swap(declval<T>(), declval<U>())), 0)` will still force a substitution of `swap`, but the result of the overall expression is just `sizeof(0)`. But this introduces a second trap - `operator,` is overloadable. In order to make sure we don't trigger a user-defined overload from a custom return type of `swap()`, we must cast the expression to `void`, so our final expression is `sizeof(void(swap(declval<T>(), declval<U>())), 0)`.

Another requirement we have is that we need a clean and well-defined way to tell whether the expression was well-formed or whether substitution failed. The style at the time was to send this out to `sizeof()` too. If you could guarantee that an expression which generated your "Yes" case would give a different result of `sizeof()` from your fallback "No" case, then you could just have your trait's value be a `static const bool value = (sizeof(???) == sizeof(Yes));`. And we do need a "No" answer because we need whatever `???` ends up being to exist and be callable whatever the answer - we are examining a `true`/`false` value, not an "exists"/"doesn't exist" value.

This is solved by letting overload resolution pick a function and examining its return type. We need two overloads - one which is well-formed if and only if our swap expression is, and another which is always well-formed but always lower in overload resolution ranking. So if our swap expression is well-formed, the first gets picked and if not the second does. A C-vararg function is always last in overload resolution, so `f(...)` will never be picked if any other overload of `f()` is valid.

Let's put all this together:

```cpp
namespace detail{
    //We need this for builtins and user classes which use std::swap
    //But we obviously don't want it unscoped in a header.
    using std::swap;

    template<typename T, typename U>
    struct is_swappable_impl{
        //Define our yes/no as guaranteed different sizes
        typedef char No;
        typedef char Yes[2];

        //Recall from our span example we need to lift these to templates
        //to prevent an eager substitution failure giving us a hard error instead
        template<typename V, typename W>
        static Yes& test(int (*)[sizeof(void(swap(declval<V>(), declval<W>())), 0)]);
        template<typename V, typename W>
        static No test(...);

        static const bool value = sizeof(test<T,U>(NULL)) == sizeof(Yes);
    };
}
template<typename T, typename U>
struct is_swappable : integral_constant<bool, detail::is_swappable_impl<T, U>::value> {};
```

If you were fortunate enough to be on an implementation which allowed full expression SFINAE then a surprisingly large number of type traits and constraints became available to you through tricks like this one. Of course if you have C++20 you can replace this whole 20-line example with `requires(T t, U u){ swap(t, u); }` and call it a day.

## 4. Empty Bases and Compressed Pairs

Let's say you want to write your own `unique_ptr`. This was often a sensible idea in the C++98 days since the standard's own `std::auto_ptr` had a few surprising quirks; and making your own smart pointer was the pet exercise in every C++ book released between 1990 and 2011. But this is not an exercise in ownership semantics. Instead, let's talk about the deleter. In short, `std::unique_ptr<T, D>` will hold a pointer to a `T` and will use an object of type `D` to delete that pointer on destruction; and if no `D` is specified by the user it defaults to `std::default_delete` which just calls `delete` on the pointer (or `delete[]` in the array case). This is not too hard to engineer, but there is one expectation of `std::unique_ptr` which does become trickier to handle - if the deleter is stateless then `sizeof(std::unique_ptr<T, D>)` should be equal to `sizeof(T*)` ([N1856](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2005/n1856.html)).

According to the C++98 standard, there is no guaranteed and portable way to do this. Every object in C++ needs to be uniquely addressable and so always has a size of at least 1. Even an empty class would have a size of 1. If you inherited from an empty class, the final object would have a size of 1 plus the derived class' size. There was a common optimisation to work around this, which was permitted but not required by the standard: the Empty Base Optimisation. If you inherited from an empty class, the compiler was permitted to give that empty class an effective size of 0 within the derived object:

```cpp
template<typename T, typename D>
class unique_ptr : D {
    T* ptr;

    //...
};
```

Now if we had an empty deleter, and EBO applies, then the sizes will match, and if it's stateful then it's trivial enough for the class to cast up to its own base to retrieve it.

Except there is one hitch here. In addition to functors, function pointers can also be used as deleters. A function pointer cannot be a base class, so the above solution would be ill-formed. As usual, our solution is an extra layer of indirection and template pattern matching:

```cpp
//FPTR case
template<typename T, typename D, bool = is_pointer<D>::value>
class deleter_holder{
    D deleter_;
    //...
};

//FUNCTOR case
template<typename T, typename D>
class deleter_holder<T, D, false> : D { ... };

//Main class
template<typename T, typename D>
class unique_ptr : deleter_holder<T, D> { ... };
```

And now our inheritance chain is 3 classes deep, but if EBO applies then we guarantee that neither stateless functor nor our `deleter_holder` intermediate class will contribute to the size of the final `unique_ptr`. If it doesn't, then our class will be quite a bit bigger.

But for completeness we can make the function pointer case stateless again by adding *another* link in the inheritance chain:

```cpp
template<typename T, void(*F)(T*)>
struct fptr_deleter{
    void operator()(T* ptr) const{
        F(ptr);
    }
};
```

This works, but inheritance seems like a strange tool to store some functor for the class to call when, if not for the size restriction, we would naturally reach for composition. It's also entirely bespoke to our `unique_ptr`, when EBO can also be used for many standard library classes, such as the comparators on ordered containers. So let's generalise this trick into something reusable. First, we can make a trait to determine whether a type is empty:

```cpp
namespace detail{
    template<typename T>
    struct empty_impl : T {
        char member;
    };

    //Typically implemented by probing T::*
    //Which will portably SFINAE in C++98 without expression SFINAE.
    template<typename T>
    struct is_builtin_type{ ... };

    template<typename T, bool = is_builtin_type<T>::value>
    struct is_empty_impl : integral_constant<bool, (sizeof(empty_impl<T>) == 1)> {};

    template<typename T>
    struct is_empty_impl<T, true> : integral_constant<bool, false> {};
}

template<typename T>
struct is_empty : detail::is_empty_impl<T> {};
```

Now we can walk towards the most basic kind of storage which exploits this trick: a "compressed pair", or rather a pair which doesn't waste storage if one of its elements is empty. A pair consists of two elements, each of which has an index, so let's make our efficient element type:

```cpp
//Nonempty types
template<typename T, std::size_t Idx, bool = is_empty<T>::value>
class element{
    T elem;

    public:
    element() : elem() {}
    explicit element(const T& value) : elem(value) {}
    T& get(){
        return elem;
    }
    T const& get() const{
        return elem;
    }
};

//Empty types
template<typename T, std::size_t Idx>
class element<T, Idx, true> : T{
    public:
    element() : T() {}
    explicit element(const T& value) : T(value) {}
    T& get(){
        return *this;
    }
    T const& get() const{
        return *this;
    }
};
```

Now we have a way to efficiently store an object with an index, and we make sure to match the interface on both cases so that the detail is hidden from the caller - it just sees a thing it can construct a `T` into and `get()` a `T` out of. After that, we just make our pair:

```cpp
template<typename T, typename U>
struct compressed_pair : element<T, 0>, element<U, 1>{

    compressed_pair(const T& t, const U& u) : element<T,0>(t), element<U,1>(u) {}

    T& first(){
        return element<T,0>::get();
    }
    U& second(){
        return element<U, 1>::get();
    }
    //Same for const overloads
};
```

The index on our `element` type guarantees that each base of our compressed pair is a unique type. This prevents `compressed_pair<T, T>` from inheriting the same class twice, which would be an error.

From here, we can make our `unique_ptr` use composition over inheritance, as is the more idiomatic way:

```cpp
template<typename T, typename D>
class unique_ptr{
    compressed_pair<T*, D> data;

    public:
    explicit unique_ptr(T* ptr) : data(ptr, D()) {}
    unique_ptr(T* ptr, D deleter) : data(ptr, deleter) {}

    ~unique_ptr(){
        if(data.first()){
            data.second()(data.first());
        }
    }

    //And the rest of the interface...
};
```

This was a common and powerful technique in the C++98 days, but it does come with some caveats worth knowing:

* Distinct objects of the same type must have different addresses, even with EBO. So `compressed_pair<T, T>` still has a size of (at least) 2.
* Unions are a hard error: Unions cannot be base classes, and there is no way to tell them apart from classes in C++98 without a compiler intrinsic. As such, there is no way to make a portable `compressed_pair` which can gracefully handle a union.

C++11 helped us greatly by making EBO mandatory for standard-layout types, which meant that all of these space-saving techniques became guaranteed behaviour instead of a common optimisation. It also hindered us by adding the `final` specifier which made many of the techniques here ill-formed, and only made it possible to reflect on whether a class was final with `std::is_final` in C++14.

C++20 added the `[[no_unique_address]]` attribute ([P0840](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0840r2.html)) which was at least intended to dispense with needing to use EBO tricks altogether. A member with this attribute is permitted to overlap with other members of a class (and in doing so contribute nothing to the class' overall size) if it is empty. So our `unique_ptr` hypothetically can now look like:

```cpp
template<typename T, typename D>
class unique_ptr{
    T* ptr;
    [[no_unique_address]] D deleter;

    //...
};
```

Unfortunately, reality is a bit more complicated. Like C++98 EBO, this is not required to be respected; and indeed MSVC ignores the attribute entirely. Using the attribute on a member permits the compiler to store a neighbouring member in the `[[no_unique_address]]` member's padding, which means that a `memcpy` over that member could overwrite a stateful data member of the class. It remains an imperfect solution.

## 5. Tuples and Variadics

Our `compressed_pair` is very useful for `unique_ptr` and turning inheritance into space-efficient composition, but it's really a special case of a more general type - a tuple. C++98 did not include variadic templates and so there is no way to have a template of arbitrary arity. Whatever you did, things would still come down to a template with a fixed number of template arguments. However, the interface layer for these could often use preprocessor tricks to give the illusion of arbitrary arity, but even here this was still over a hidden maximum preprocessor arity as `__VA_ARGS__` was not officially in C++ until C++11. For now, though, we will focus on the actual class underneath that interface.

Let's generalise our `compressed_pair`. For the sake of typing we'll just use an arity of 3, but it can be as large as you are willing to type out:

```cpp
template<int N>
class null_type{};

template<typename T0, typename T1 = null_type<1>, typename T2 = null_type<2> >
struct tuple : element<T0, 0>, element<T1, 1>, element<T2, 2> { ... };
```

This is nothing new - we use a defaulted class template argument to hide the unused template parameters, and our EBO trick to ensure that they consume no effective storage unless they are used. Our `null_type` is a template to guarantee that all our unused arguments are different types and so can all be stored efficiently with EBO.

This trick solves storage, but it doesn't solve how we would actually retrieve an element by index. The first thing we need to be able to do is name the type we are retrieving - there was no `auto` for type deduction in C++98 and every type had to be spelled out directly. Fortunately, this can be done by applying the same pattern matching to a trait-style struct that we've already seen:

```cpp
template<std::size_t Idx, typename T0, typename T1, typename T2>
struct element_type;

template<typename T0, typename T1, typename T2>
struct element_type<0, T0, T1, T2> { typedef T0 type; };

template<typename T0, typename T1, typename T2>
struct element_type<1, T0, T1, T2> { typedef T1 type; };

template<typename T0, typename T1, typename T2>
struct element_type<2, T0, T1, T2> { typedef T2 type; };
```

And now given a list of types and an index, we can name the type of element `Idx` of that list. Our indexed getter function is simplicity itself:

```cpp
template<std::size_t Idx, typename T0, typename T1, typename T2>
typename element_type<Idx, T0, T1, T2>::type& get(tuple<T0, T1, T2>& t){
    typedef typename element_type<Idx, T0, T1, T2>::type type;
    return static_cast<element<type, Idx>&>(t).get();
}
```

This is a lot of typing, and is still just pretending that we're doing something "variadic" while hiding that what we really have is flat storage with a fixed size, only some of which may be filled with data you care about. So let's fix that. Let's take our model for a pair, and ask ourselves what if the second template argument was itself a pair of two types. And that deeper pair could itself be a pair of an element and another pair, and so on. In theory we can nest these pairs arbitrarily deep[^recurse-depth] and deal exclusively in pairs consisting of an element and the rest of the list. Lisp calls this a cons list, and anything which walks this list only needs two cases - the head, and everything else.

[^recurse-depth]: The standard does recommend a minimum depth on supported nested template instantiations. In C++98 this was 17, but implementations frequently supported deeper.

What would this look like? The magic lies in the constructors. We want the ability to push an element onto the front of an existing list, and the ability to construct a fresh list from scratch:

```cpp
struct null_type{};

template<typename Head, typename Tail = null_type>
struct cons{
    typedef Head head_type;
    typedef Tail tail_type;

    Head head;
    Tail tail;

    //Push a new Head onto an existing list
    cons(const Head& h, const Tail& tail) : head(h), tail(tail) {}

    //And construct a new one from a set of elements
    template<typename T0, typename T1, typename T2>
    cons(const T0& t0, const T1& t1, const T2& t2):
        head(t0), tail(t1, t2, null_type()) {}

};

//And our base case
template<typename Head>
struct cons<Head, null_type>{
    typedef Head head_type;

    Head head;

    cons(const Head& h, null_type) : head(h) {}

    template<typename T0, typename T1, typename T2>
    cons(const T0& t0, const T1& t1, const T2& t2) : head(t0) {}
};
```

Again we need a way to name the type of the element at index `Idx` of our list. Fortunately, the recursive route means that we only need to write out two class templates rather than one per arity:

```cpp
template<std::size_t Idx, typename List>
struct element_type{
    typedef typename element_type<Idx - 1, typename List::tail_type>::type type;
};
template<typename List>
struct element_type<0, List>{
    typedef typename List::head_type type;
};
```

And our indexed getter function can just recurse down the list `Idx` times and then return the head of whatever `cons` it finds there.

```cpp
namespace detail{
    template<std::size_t Idx>
    struct get_impl{
        template<typename List>
        static typename element_type<Idx, List>::type& get(List& l) {
            return get_impl<Idx - 1>::get(l.tail);
        }
    };
    template<>
    struct get_impl<0>{
        template<typename List>
        static typename List::head_type& get(List& l){
            return l.head;
        }
    };
}

template<std::size_t Idx, typename Head, typename Tail>
typename element_type<Idx, cons<Head, Tail> >::type& get(cons<Head, Tail>& c){
    return detail::get_impl<Idx>::get(c);
}
```

All that remains before we can define our tuple is a means to convert a flat template argument list `<T0, T1, T2>` into a cons list `cons<T0, cons<T1, cons<T2> > >` (note the mandatory spacing on the closing angle brackets - this was needed as C++98 always interpreted `>>` as the right shift operator). Unfortunately, this is where our ability to avoid writing one overload per arity ends. As soon as we are actually naming the final types we will put into this class we can't keep pretending it's truly variadic. But the approach here does still follow the previous recursive pattern:

```cpp
template<typename T0, typename T1, typename T2>
struct to_cons{
    typedef cons<T0, typename to_cons<T1, T2, null_type>::type> type;
};
template<>
struct to_cons<null_type, null_type, null_type>{
    typedef null_type type;
};
```

We just keep packing with `null_type` until we hit our base case and stop.

Now we can define our tuple:

```cpp
template<typename T0, typename T1 = null_type, typename T2 = null_type>
struct tuple : to_cons<T0, T1, T2>::type {
    typedef typename to_cons<T0, T1, T2>::type base;

    explicit tuple(const T0& t0 = T0(), const T1& t1 = T1(), const T2& t2 = T2()) : base(t0, t1, t2) {}
};
```

So, in this example, `tuple<int, float>` derives from `cons<int, cons<float> >`. Even though we name `null_type` and default our unused template arguments to it, it never actually occupies storage because our template instantiation stops at the first `null_type` it encounters. Our `get<1>(t)` function compiles down to `t.tail.head` and extracts the value stored there.

The cons tuple and the flat tuple are each a good solution to around half of the problem. The flat tuple guarantees that element lookup is a direct operation, but this comes at the cost that all surrounding functions and metafunctions must have an overload or specialisation per arity. The cons tuple saves us from most (but not all) of those duplicates, but it means that each saved function must perform a recursive template instantiation which grows with your index. In C++98, your "flat" constructors in `cons` and `tuple` had to spell out every parameter and any `make_tuple` function you wanted to write needed one overload per arity, and there was nothing you could do about it.

C++11 added variadic templates, which had the immediate benefit of solving the arity problem - now your arity was genuinely uncapped.[^uncapped] It also allowed the two strategies discussed here to be combined. We can keep our `element<T, Idx>` storage type as before (but with forwarding and `std::is_empty`), and build a recursively stored set of tuple elements:

[^uncapped]: At least up to the limit supported by your implementation. The standard recommended a minimum of 1024.

```cpp
//The end of the list - an empty type
template<std::size_t Idx, typename... Ts>
struct tuple_impl {};

//And a cons-list recursive case for everything else
template<std::size_t Idx, typename Head, typename... Tail>
struct tuple_impl<Idx, Head, Tail...> : element<Head, Idx>, tuple_impl<Idx + 1, Tail...>{
    tuple_impl() = default;

    template<typename T, typename... Ts>
    explicit tuple_impl(T&& t, Ts&&... ts)
        : element<Head, Idx>(std::forward<T>(t)),
          tuple_impl<Idx + 1, Tail...>(std::forward<Ts>(ts)...) {}
};
```

And now we can make our tuple:

```cpp
//We need to constrain that each constructor arg is convertible to its corresponding tuple element
//So we need these helpers
template<bool... > struct bools;
template<bool... Bs>
using all_true = std::is_same<bools<true, Bs...>, bools<Bs..., true>>;

template<typename... Ts>
struct tuple : tuple_impl<0, Ts...> {
    using base = tuple_impl<0, Ts...>;

    tuple() = default;

    //Now we can have a defaulted template parameter on a function so we can constrain this ctor
    template<typename... Us, typename std::enable_if<
        all_true<std::is_convertible<Us, Ts>::value...>::value,
    bool>::type = true>
    explicit tuple(Us&&... us) : base(std::forward<Us>(us)...) {}
};
```

Our `tuple_impl` recursively expands out our bases to match those of the flat tuple, i.e. pairs of `element<T, Idx>`. As we now have `decltype` we don't need a lengthy metafunction to name the type our `get()` function returns:

```cpp
namespace detail{
    template<std::size_t Idx, typename T, bool E>
    T& get_elem(element<T, Idx, E>& e){
        return e.get();
    }
}

template<std::size_t I, typename... Ts>
auto get(tuple<Ts...>& t) -> decltype(detail::get_elem<I>(t)) {
    return detail::get_elem<I>(t);
}

//Plus const& and && overloads all the way down
```

As before in the flat case, because our tuple publicly inherits from a matching `element<T, Idx>` for each type, it can be implicitly converted to each one as needed.

This is a big improvement on the C++98 case, but it can be better. In addition to a real `is_final` trait and the `_t` type trait aliases removing the need for `typename` qualifiers everywhere, C++14 allowed us to drop the recursive expansion of our `tuple_impl` completely. We got `std::index_sequence` ([N3658](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3658.html)) to expand out a variadic list of indices next to our `Ts...` when forming the tuple base. Our `tuple_impl` class simplifies to:

```cpp
template<typename, typename...> struct tuple_impl;

template<bool... > struct bools;
template<bool... Bs>
static constexpr bool all_true = std::is_same<bools<true, Bs...>, bools<Bs..., true>>::value;

template<std::size_t... Idxs, typename... Ts>
struct tuple_impl<std::index_sequence<Idxs...>, Ts...> : element<Ts, Idxs>... {
    tuple_impl() = default;

    template<typename... Us>
    explicit tuple_impl(Us&&... us) : element<Ts, Idxs>(std::forward<Us>(us))... {}
};

template<typename... Ts>
struct tuple : tuple_impl<std::index_sequence_for<Ts...>, Ts...>{
    using base = tuple_impl<std::index_sequence_for<Ts...>, Ts...>;

    tuple() = default;

    template<typename... Us, std::enable_if_t<
        all_true<std::is_convertible<Us, Ts>::value...>,
    bool> = true>
    explicit tuple(Us&&... us) : base(std::forward<Us>(us)...) {}
};
```

Our `get()` function remains largely unchanged, except with `auto` for type deduction we can avoid naming the return type of the function altogether:

```cpp
template<std::size_t I, typename... Ts>
auto& get(tuple<Ts...>& t) {
    return detail::get_elem<I>(t); //get_elem unchanged from before
}
```

The standard library tool for this, `std::get`, was also made `constexpr` in C++14.

C++17 gave us fold expressions, so our constraint on the tuple constructor no longer needs its `all_true` and `bools` helpers:

```cpp
template<typename... Ts>
struct tuple : tuple_impl<std::index_sequence_for<Ts...>, Ts...>{
    using base = tuple_impl<std::index_sequence_for<Ts...>, Ts...>;

    tuple() = default;

    template<typename... Us, std::enable_if_t<
        (std::is_convertible_v<Us, Ts> && ...), // <-- No helpers here
    bool> = true>
    explicit tuple(Us&&... us) : base(std::forward<Us>(us)...) {}
};
```

C++20 gave us proper constraints so we could drop the `enable_if_t` trick, and for variadic templates allowed pack expansion in lambda-init-capture ([P0780](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0780r2.html)). C++23 added deducing this ([P0847](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p0847r7.html)) to cut down on overloads, which applies to our tuple's `element::get()` functions.

C++26 brings static reflection, and while it can't do arbitrary code generation it does allow us to cast aside the entire bag of template tricks we have been relying on so far in this section and implement a tuple in the model that you would naively expect it to follow (adapted from [P2996](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p2996r13.html)):

```cpp
template<typename... Ts>
struct tuple {
  struct storage;
  consteval {
    define_aggregate(^^storage, {data_member_spec(^^Ts, {.name = "_", .no_unique_address = true})...});
  }
  storage data;

  tuple(): data{} {}
  explicit tuple(Ts const& ...vs): data{ vs... } {}
};
```

Now our tuple simply houses an aggregate which contains a data member for each type. This seems to close the loop. We have gone from scraping together a collection of template black magic tricks with more `...` expansions than any person wants to see in a single file to a single, ten-line class which just tells the compiler to put the members you want in the right places.

## Conclusion

I hope you have enjoyed these five[^five] examples of template metaprogramming through the years. They show the journey that this field of C++ has been on, from the dark days where you needed to trick the template engine into doing your bidding and substitution failure was indeed an error, to our modern world where you effectively just tell the compiler what you want in plain English (or as close to it as C++ will ever get) and your templates do exactly what you intend. Reflection promises to reshape template metaprogramming again, and `std::meta::info` as an opaque type means that many constraints and functions don't even need to be templates any more.


[^five]: For sufficiently large values of five.