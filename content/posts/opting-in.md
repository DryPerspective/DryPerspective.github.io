---
date: '2026-10-10T15:06:55+01:00'
title: 'Opting in, and back out again'
ShowToc: true
TocOpen: false
summary: "The ways to manually opt your code into a function through the history of C++, and the black magic required to opt it back out again."
tags: ["c++", "c++26", "reflection", "concepts", "one-definition-rule", "stateful-metaprogramming", "wg21", "annotations"]
---

You have just written a new library. It's an incredible new library which will revolutionise the way that we handle widgets forever. But there is just one problem - the user has to opt into it themselves. For one reason or another you can't assume whether a class is compatible from the code itself, you need the user to make an explicit check, and you need to provide the API for them to do so.

```cpp
template<opted_in T>
constexpr Widget widgetiser(T&& user_provided_class);
```

So, how do you go about it? How do you make the `opted_in` concept? There are quite a few ways in C++ which have varying degrees of success, and a small minefield of things which can go wrong. Let's examine some of the options, considering how convenient they are, how easy to understand they are, and how easy it is to customise them based on the user class.

## Class typedefs and named tags

The classic case which standard library iterators have been using since the standard library came into being. As part of your public API you can expose a name in the form of a typedef for your library to latch onto. Consider:

```cpp
//Library code
template<typename T>
concept opted_in = requires{
    typename T::is_widget;
};


//User code
struct user_class{
    using is_widget = void;

    //...
};
```

Sorted. Except, how do we make this conditional? If the user class is a template, how do we opt in for only a certain subset of types? Well, we can encode a special type, and require that `T::is_widget` is the same type as `yes_i_definitely_opted_in_t`; but this is hardly ergonomic, particularly for the general case where you want to unconditionally opt in. You could try to invert it so that exclusively in the template case, `T::is_widget` is `actually_i_opt_out_t` for the cases where your class is not a suitable widget; but then you are stacking up more and more negative conditions to try to describe your code. You could make it a `constexpr bool` but then you require an explicit opt-in in the unconditional case; and if you're using multiple opt-in libraries you tightly couple those to the definition of your class by introducing reachable, readable names whose meanings are largely irrelevant to what the class is actually supposed to be.

And even then - a user who has not read your library's documentation will see a class with `is_widget` present and might be quite surprised when the library summarily rejects it. All of the ways around this ultimately result in your needing to make up a pseudo-language of conditions encoded in things the user doesn't know about and at scale can give the user a 2000 word essay on all the mechanics to read before they feel capable of using your code. Equally, not everything can have member typedefs, only classes. If you want to make a particular enum, or function or universal constant valid for widgetisation then you're out of luck.

And there is a simple question here - how do you opt third party classes into your library for use in your code? You might not be able to reasonably edit the definition and maintain a fork from its original source for the sake of easier widgeting.

## An opt-in template boolean

This is the more modern approach which ranges-v3 and `std::ranges` tend to follow (e.g. `std::ranges::enable_borrowed_range`). Rather than read something tightly coupled inside the class definition, you provide a template boolean, say `enable_widgetising`, which you then specialise to `true` for all the classes which follow it. So:

```cpp
//Library code
template<typename T>
constexpr inline bool enable_widgetising = false;

template<typename T>
concept opted_in = enable_widgetising<T>; //Arguable about whether a concept is necessary


//User code
struct user_class{
    //...
};

template<>
constexpr inline bool enable_widgetising<user_class> = true;
```

There we go. Now your user classes have an entirely bespoke and entirely unique-to-your-library name they can customise to `true`. You can also selectively apply this boolean to class templates in user code by constraining the boolean, so

```cpp
//Everyone knows that integer widgets are the only true widgets
template<typename T> requires std::integral<T>
constexpr inline bool enable_widgetising<user_class<T>> = true;
```

And this will compile and behave as you reasonably expect. Since we have decoupled the opt-in mechanism from the class definition, it is also trivial to enable your library for third party code. There is a catch here, however. Every time the compiler attempts to evaluate `enable_widgetising<user_class>`, it must come to the same result otherwise your program is ill-formed, no diagnostic required (IFNDR) - a very special kind of wrong where the compiler is no longer required to produce results which are logical or even deterministic through successive builds of the program. To give an example of this, consider two TUs:

```cpp
//TU A
#include "user_class.hpp"

//Everyone knows that user_class is not a widget...
static_assert(not enable_widgetising<user_class>);
```
```cpp
//TU B
#include "user_class.hpp"
#include "widgetiser.hpp"

//...But I can shortcut the logic in my current ticket by making it behave like one
template<>
constexpr inline bool enable_widgetising<user_class> = true;
```

[[temp.expl.spec]](https://eel.is/c++draft/temp.expl.spec#7) requires that the specialisation we provide must be reachable from every use in every TU. Neither TU can see the other, so each will compile happily with its own answer. At link time, the linker will pick one answer and discard the other, and the behaviour of the rest of the program will then depend on link order and optimisation level; which in turn will kick up bugs which appear in one file but are actually the result of compiling another, potentially at any time in the past.

This can be avoided by still coupling the boolean specialisation to your definition, ensuring that every TU which can see `user_class` can also see that `enable_widgetising` is set, if indeed it is.

If this seems a little abstract and unlikely to present a problem to you, there is a much more pressing ergonomic concern which makes this kind of boolean unideal - where you place it. Up to this point, to simplify examples, I have not used namespaces. But let's fix that:

```cpp
namespace library{
    //...
    template<typename T>
    constexpr inline bool enable_widgetising = false;
}

namespace user{
    class user_class {};
    //We can't specialise enable_widgetising here - we're in the wrong namespace
}
//So we have to reopen our library namespace.
namespace library{
    template<>
    constexpr inline bool enable_widgetising<user::user_class> = true;
}
```

But here is the problem. It's not uncommon for a C++ code file to contain multiple classes. It is unthinkably silly to close whatever namespace they're in after each definition, open up the library namespace to customise, close that namespace, and then open up the userspace namespace for the next class definition. So what often ends up happening is that these template booleans drift their way down to the bottom of the file after everything else. Aside from their being out of sight (and therefore out of mind) opening the door to forgetting to update them when you play with the class, you get yourself into a bit of a pickle if any of the classes want to be able to widgetise before you reach the boolean. Be it an inline function body, or use of a previously-declared class, or even just the fact you want to widgetise a plain old class template, you are left in the unfortunate position which might mean that the opt-in to enable widgetising hasn't yet been reached, and you are stuffed.

## A consteval function opt-in reached via ADL

So, you think, the problem with a boolean is that it's an object in a different namespace from your own. But, being a good developer, you recall that argument-dependent lookup has long been a basis for customisation points in classes in order to avoid this very problem. And since you are defining a function to determine opt-in eligibility you can have an easy time writing freeform code rather than trying to round-trip it through a concept. So, let's take a look at what we can do:

```cpp
//Library code
consteval bool enable_widgetising(auto) = delete; //More on this later

template<typename T>
concept opted_in = requires(T){
    { enable_widgetising(std::type_identity<T>{}) } -> std::same_as<bool>;
} && enable_widgetising(std::type_identity<T>{});


//User code
class user_class{};
consteval bool enable_widgetising(std::type_identity<user_class>){
    return true;
}
```

In short, we require that it is possible to call some function `enable_widgetising` with some tag type which represents our desired type, and then we require that when we do call it the result is true. In this case we use `std::type_identity`, but so long as it is a type which will cause ADL to search in the namespace of your `user_class` (this is guaranteed for `type_identity<user_class>` via [[basic.lookup.argdep]](https://eel.is/c++draft/basic.lookup.argdep#3.3) as it is for other templates with `user_class` as a template argument) then there is no messing with namespaces needed. It's trivial to customise the behaviour based on your class, if it is a template, within the function and in a way which makes it as unobscured from the user as any approach discussed so far. You get full flexibility. You can even place it inside your class definition as a hidden friend and use your function inside the class itself, across namespaces, seamlessly:

```cpp
namespace library{

consteval bool enable_widgetising(auto) = delete;

template<typename T>
concept opted_in = requires {
    { enable_widgetising(std::type_identity<T>{}) }
        -> std::same_as<bool>;
} && enable_widgetising(std::type_identity<T>{});

template<opted_in T>
void widgetise(T) {}

}


namespace user{
template<typename T>
class user_class{
    public:
    friend consteval bool enable_widgetising(std::type_identity<user_class>){
        return true;
    }

    void some_function(){
        library::widgetise(*this);
    }
};

//And of course the conditional case is simple
template<typename I>
class user_class_v2{
    public:
    friend consteval bool enable_widgetising(std::type_identity<user_class_v2>){
        return std::integral<I>;
    }
};
}
```
Godbolt [here](https://godbolt.org/z/87Whjz33c)

The deleted `enable_widgetising(auto)` function is a poison pill to contain the lookup to just relevant places when checking the concept. When the concept check tries to require that `enable_widgetising(std::type_identity<T>)` is true, it checks `namespace std` (because that's where `type_identity` lives), and the namespace in which `T` is declared. If it doesn't find anything named `enable_widgetising`, it then "steps outwards" and checks the library namespace, where it will encounter the deleted function and, since it can't be called, will mean that the concept check fails.[^ADL] The fact that we use `auto` is an overload resolution trick - if more than one match is found in the library namespace, then a function which is explicitly declared as taking an argument of some `std::type_identity` for some type will rank higher than an unspecialised template and be chosen first; such that a class in the library's own namespace being opted in will still be opted in, but the compiler will never step further out and consider other namespaces or the global namespace when looking for a match for `enable_widgetising`.

[^ADL]: Technically ADL doesn't step out of anywhere. It composes a list of the associated namespaces and searches them in one pass, while normal lookup runs from where the concept is defined, and the results of the two are merged. What the deleted function buys you is name hiding in that normal lookup: the presence of a particular name in an "inner" scope stops the lookup from reaching that name in "outer" scopes. But looking in one place and then stepping outwards echoes normal name lookup and is in many ways easier to understand.

There are a few caveats to note - this change is not immune to the same IFNDR concern mentioned with a template boolean. If you opt to place the opt-in function outside of the class definition, then the result when checking if a class has opted in must be the same at all places it is checked in all TUs. This is also a minimal example and you may need judicious use of `remove_cvref_t` and similar in your library definitions if your function accepts cv-qualified or reference types.

This seems to have answered some of the problems with the other methods - we are free to place the opt-in mechanism in a few different places without it interfering with our work, but for the case of third party types we still need to open up their namespaces to place the declaration for ADL to work. But it can still be improved. It's more lines of code both at the library and user level and starts to require understanding of slightly more niche language mechanisms to fully utilise. Perhaps there's another way.

## An annotation on the class definition

Among the reflection tools added in C++26, we received annotations. For those unfamiliar, an annotation is syntactically similar to an attribute, except that it is prefixed with `=` and can be any structural type constant expression. With reflection we can reflect on the resulting values of that expression. So, let's construct a basic example:

```cpp
//Library code
struct enable_widgetising_t {};

constexpr inline enable_widgetising_t enable_widgetising {};

template<typename T>
concept opted_in = not annotations_of_with_type(^^T, ^^enable_widgetising_t).empty();


//User code
class [[=enable_widgetising]] user_class{

};
```

Nice and simple. The annotation `[[=enable_widgetising]]` is an expression whose underlying type is `enable_widgetising_t`. We use the C++26 reflection metafunction `std::meta::annotations_of_with_type` (here called via ADL) to see if `T` has any annotations of type `enable_widgetising_t`, and our concept works.

But, we ask the same question as on everything else - how do we customise this, for example if our class is a template of which only certain specialisations should widgetise? Let's start by considering what we want our final syntax to be. I think that the code in the above example is ideal for the non-conditional variant, and to add a boolean condition, we should follow the way of the C++ keywords so that `[[=enable_widgetising(some_condition)]]` will enable widgetising if and only if `some_condition` is met.

Fortunately, this is fairly simple. We can add a function call operator to our tag type so that both `enable_widgetising` and `enable_widgetising(condition)` are valid expressions. But there is one catch - we can't just check if the type of the annotation is `enable_widgetising_t`, since we can't return types as types; so instead we attach our boolean state so that we can pass out something which is true or false, and which is the same type as the unconditional case.

```cpp
//Library code
struct enable_widgetising_t{
    bool enabled = true;
    consteval enable_widgetising_t operator()(bool b) const{
        return {b};
    }
};

constexpr inline enable_widgetising_t enable_widgetising{};

template<typename T>
consteval bool opted_in_via_annotation(){
    for(auto annotation : annotations_of_with_type(^^T, ^^enable_widgetising_t)){
        if(extract<enable_widgetising_t>(annotation).enabled){
            return true;
        }
    }
    return false;
};

template<typename T>
concept opted_in = opted_in_via_annotation<T>(); //Again arguable that a concept is needed


//User code
template<typename I>
class [[=enable_widgetising(std::integral<I>)]] user_class{

};
```

In short, we carry a boolean which will default to true unless otherwise perturbed, and so we are enabled by default. When we pass in a condition which is false, we return a tag with the boolean also set to false and when we examine annotation we can query whether it is enabled or disabled. Simple.

This approach has returned us back to where we started - if the annotation is tightly coupled to the class definition, then third party classes cannot pick it up. Except, they can if you're willing to get your hands dirty. The definition of your class can only exist in your immutable third-party headers, but *declarations* of it can exist anywhere. Declarations can carry annotations, and when the annotations on an object are queried then the result is the union of the annotations on all declarations and definitions the compiler can see. gcc seems to counter such hackery by ignoring annotations on a class declaration which appears after its definition. But it doesn't ignore declarations before the definition. So if you're willing to hold your nose and do something like this:

```cpp
#include "widgetiser.hpp"

class [[=enable_widgetising]] user_class;

#include "user_class.hpp"
```

Now, I am definitely not recommending that you actually do this in production code. This is a quirk to hack around a very reasonable restriction and if you do this in your code then there's a good chance it'll break, either from the IFNDR reasons discussed previously or the fact that `user_class` might not be a class but instead a type alias or some other construct.

What's more interesting about the annotation approach is that now we have the annotation looking up named types, it's easy to generalise the machinery:

```cpp
//Shared opt-in machinery
template<typename Unique>
struct opt_in_t {
    bool enabled = true;
    consteval opt_in_t operator()(bool b) const { return {b}; }
};

template<auto& Key>
consteval bool opted_in_via_annotation(std::meta::info cls) {
    using K = std::remove_cvref_t<decltype(Key)>;
    for (auto annotation : annotations_of(cls))
        if (is_same_type(remove_cvref(type_of(annotation)), ^^K)){
            return extract<K>(annotation).enabled;
        }
    return false;
}

//What we need to define for widgets
struct widgetiser_tag;
constexpr inline opt_in_t<widgetiser_tag> enable_widgetiser{};

template<typename T>
concept opted_in = opted_in_via_annotation<enable_widgetiser>(^^T);


//And user code remains
class [[=enable_widgetiser(some-condition)]] user_class {};
```

Where we define a `widgetiser_tag` unique type to make sure that our specialisation of `opt_in_t` is itself a unique type, distinct from other specialisations; or rather, so that opting in to one thing does not opt you into everything by mistake. An alternative route is to use a reflection of the thing you are opting into:

```cpp
template<std::meta::info Unique>
struct opt_in_t {
    bool enabled = true;
    consteval opt_in_t operator()(bool b) const { return {b}; }
};

//Opt-in function is unchanged from previous example

namespace widgetiser_lib {
    constexpr inline opt_in_t<^^widgetiser_lib> enable_widgetiser{};

    template<typename T>
    concept opted_in = opted_in_via_annotation<enable_widgetiser>(^^T);
}

//And user code is unchanged
class [[=widgetiser_lib::enable_widgetiser(some-condition)]] user_class {};
```

Note that we can't reflect on our opt-in function directly, as this will create a circular dependency - its constraint needs `enable_widgetiser` to exist, and `enable_widgetiser` would need the constraint to exist to reflect on a proper declaration. But, as you can reflect on pretty much anything in C++26, a sensible alternative might be the library namespace you are opting into.

This is probably as far as it is sensible to take this, but what if we could take it further? What if it's just too onerous to define a unique new type when opting in, or deciding which entity to reflect on, for the sake of satisfying some template pedantry. What we'd need is a way for the compiler to mint an entirely unique type on-demand with some kind of unutterable name. Well, fortunately C++ has just the feature. What if instead, we wrote:

```cpp
constexpr inline opt_in_t<decltype([]{})> enable_widgetiser{};
```

And relied on the fact that a lambda, even one which is token-identical to another lambda, will always generate an entirely bespoke type? Well, this would probably *work* in isolated test cases, but lambdas are surrounded by a cloud of uncertainty in the standard when it comes to exactly whether a lambda definition which is included in multiple TUs refers to the same entity.

The important question to ask here is simple - is this `enable_widgetiser` the same type in every TU it's included into, or is it a different one? As far as I can tell, the standard's answer is a somewhat sheepish maybe. [[basic.def.odr]](https://eel.is/c++draft/basic.def.odr#18) seems to suggest that such lambdas in the definition of an inline entity have the same closure type, which would appear to cover us. However, a note in that clause explicitly tells us that the entity is still declared in multiple TUs, that [[basic.link]](https://eel.is/c++draft/basic.link) applies to those declarations, and that lambdas appearing *in the type* of an entity can result in each declaration having a different type. [basic.link] requires that every declaration of a variable gives it the same type, otherwise the program is IFNDR. If the type of the lambda matches everywhere then our trick works; but if not then our program is ill-formed, no diagnostic required and those last three words mean that we might not find out until the least opportune time.

Entertainingly, if we define our constant instead like this:

```cpp
constexpr inline auto enable_widgetiser { opt_in_t<decltype([]{})> {}};
```

Then the standard is surprisingly clear that, due to the lambda being in the initializer of an inline variable, this is unambiguously well-defined everywhere, per a specific paragraph in [[basic.def.odr]](https://eel.is/c++draft/basic.def.odr#18).

Equally, you may have the bright idea that typing out `decltype([]{})` every time is too much like hard work and that you'd instead rather just use it as the default template argument for `opt_in_t`. Here the standard is much clearer. The same [[basic.def.odr]](https://eel.is/c++draft/basic.def.odr#16.11) which might save us for the type definition case and which does save us in the initializer case treats a default template argument as if it were written inside the definition, so the created closure is required to have the same type in every TU. But [a later point](https://eel.is/c++draft/basic.def.odr#18) explicitly excludes lambdas in default template arguments from being treated as the same entity across TUs. This seems to reach a confusing conclusion that they are required to match while also guaranteeing that they don't, and that your program is IFNDR.

However, the initializer case being well-defined does not inherently mean your program will work. Compilers quite notoriously have problems maintaining the exact letter of the standard in fringe lambda edge-cases such as this one, and may silently break your program anyway. Indeed we can see a conformance issue just by throwing this example into [godbolt](https://godbolt.org/z/jfqhxb7K6). gcc correctly gives the initializer case the same definition across TUs, but Clang silently gives the closure internal linkage, meaning each type is unique even if they report the same `typeid` name. It's most likely worth the strain of defining a new type or deciding on something to reflect on when defining the opt-in annotation - those at least are always guaranteed to be consistent in every TU and implementations consistently get it right.

Overall, the annotation idea solves many of our problems with just how flexible annotations can be, and how easy they are to append onto almost anything in the language. But if you go down the library route in your own code, it's far more sensible to define a tag type or use a reflection as the unique key than it is to try to solve that problem with lambda hijinks.

## Opting back out again

The dark side of template metaprogramming is a pathway to many abilities the Committee consider to be unnatural. I refer of course to the black magic known as stateful metaprogramming - there are ways and means to inject state into the compiler as it is parsing and processing your code, and ways to exploit that state to effect some program logic. But in case it's not clear everything in this section comes with a huge disclaimer - do not do this in any real production code. A lot of it is entirely IFNDR and even if parts are somehow well-defined by the letter of the standard, the committee have [gone on the record](https://cplusplus.github.io/CWG/issues/2118.html) to say that they don't like that this is possible and want to find a way to forbid it. Also your compiler may spuriously decide to break it if it caches a definition differently between builds. While there are many ways to go about this, the classic method is:

### Friend Injection

The idea behind friend injection is simple. You declare, but don't define, a friend function inside of some template. Then, later on, you can provide a definition of that function in another template. Instantiating the template will define the friend function, and the definition outlives the instantiation. Consider:

```cpp
template<int N>
struct counter_slot {
    //Declare our friend
    friend constexpr auto is_taken(counter_slot<N>);
};

template<int N>
struct take_slot {
    //And define it here
    friend constexpr auto is_taken(counter_slot<N>) { return true; }
};

//Each call needs a fresh specialisation, or the compiler can cache previous results
//So we use a lambda for uniqueness
template<int N = 0, auto = []{}>
consteval int next_id() {
    if constexpr (requires { is_taken(counter_slot<N>{}); }) {
        return next_id<N + 1>();
    } else {
        static_cast<void>(take_slot<N>{});
        return N;
    }
}
```

The instantiation of `counter_slot<0>` quietly declares a unique `is_taken` function at namespace scope, invisible to normal lookup. What's more, it has an `auto` return type so it cannot be called until the compiler sees its definition, and `take_slot` will provide that definition only after `take_slot<0>` is instantiated. Then ADL lets the call `is_taken(counter_slot<N>{})` find the declaration. So when the function checks `requires { is_taken(counter_slot<N>{}); }`, if `take_slot<N>` has already been instantiated elsewhere then `is_taken(counter_slot<N>{})` is callable and the requires expression comes up true. If `take_slot<N>` has not been instantiated anywhere, the requires expression attempts to call an uncallable function, comes out as false, and the function then instantiates `take_slot<N>`. As such, every time you call `next_id`, the compiler will have to instantiate a new `take_slot` specialisation, which in turn gives you an entirely new `N`.

But how does this relate to opting out? Well, it's not a large logical leap to consider that maybe if the latest N is even then we can widgetise and if it is odd then we can't; we can wrap the counter up in functions named `opt_in` and `opt_out`, and go from there. And it's not that much harder to write the code, but the requirement of uniqueness does start to go viral. The compiler is perfectly within its rights to cache the result of multiple templates with the same arguments, as the intent of the standard is that these should universally provide the same answer every time. So, we need to push lambdas everywhere to enforce a unique evaluation. But, it can be done:

```cpp
#include <print>
#include <type_traits>

//Same as before, except we introduce a typename T for the user class
template<typename T, int N>
struct toggle_slot {
    friend constexpr auto is_taken(toggle_slot<T, N>);
};

template<typename T, int N>
struct take_toggle {
    friend constexpr auto is_taken(toggle_slot<T, N>) { return true; }
};

template<typename T, int N = 0, auto = []{}>
consteval int toggles_so_far() {
    if constexpr (requires { is_taken(toggle_slot<T, N>{}); }) {
        return toggles_so_far<T, N + 1>();
    } else {
        //Except we are currently just querying so we don't instantiate a new specialisation yet
        return N;
    }
}

//An even-numbered slot opts in, an odd-numbered slot opts back out
template<typename T, auto = []{}>
consteval bool widgetising_enabled() {
    return toggles_so_far<T>() % 2 == 1;
}

//So our opt-in/opt-out simply handles the instantiation for us to make sure the number is suitably
//even or odd after calling, with a static_assert to catch out-of-order calls.
template<typename T, auto = []{}>
consteval bool opt_in_to_widgetising() {
    constexpr int next_slot = toggles_so_far<T>();
    static_assert(next_slot % 2 == 0, "already opted in");
    static_cast<void>(take_toggle<T, next_slot>{});
    return true;
}

template<typename T, auto = []{}>
consteval bool opt_out_of_widgetising() {
    constexpr int next_slot = toggles_so_far<T>();
    static_assert(next_slot % 2 == 1, "not opted in");
    static_cast<void>(take_toggle<T, next_slot>{});
    return true;
}

struct Widget {};

//The concept takes a fresh lambda too, so every check is a new atomic constraint
template<typename T, auto Fresh>
concept opted_in = widgetising_enabled<std::remove_cvref_t<T>, Fresh>();

//As does your library function
template<typename T, auto Fresh = []{}>
    requires opted_in<T, Fresh>
constexpr Widget widgetiser(T&&) { return {}; }

template<typename T, auto = []{}>
constexpr bool can_widgetise = requires { widgetiser(T{}); };

struct user_class {};

constexpr bool before_opting_in = can_widgetise<user_class>;

static_assert(opt_in_to_widgetising<user_class>());
constexpr bool after_opting_in = can_widgetise<user_class>;

static_assert(opt_out_of_widgetising<user_class>());
constexpr bool after_opting_out = can_widgetise<user_class>;

static_assert(opt_in_to_widgetising<user_class>());
constexpr bool after_opting_back_in = can_widgetise<user_class>;

int main() {
    std::println("Before opting in:      {}", before_opting_in);
    std::println("After opting in:       {}", after_opting_in);
    std::println("After opting out:      {}", after_opting_out);
    std::println("After opting back in:  {}", after_opting_back_in);
}
```
Godbolt [here](https://godbolt.org/z/7eW4rax3x).

Ten years ago this was the hot trick to talk about at C++ cocktail parties, but times have moved on, C++26 is here, and there's a whole new language waiting for ways to break things:

### Reflection

On the face of it, it's trivial to get inconsistent answers out of C++26 reflection - it comes with a function to count how many entities can be seen at that point in the TU, and while it doesn't have a full suite of generative facilities it also comes with a function to inject new definitions in the form of `define_aggregate`. Indeed we can knock out a rhyme of our friend injection technique in a few metafunctions:

```cpp
//Declare but don't define a class this time
template<int N>
struct history_slot;

//Get a (still undefined) reflection of history_slot<index>
consteval std::meta::info slot_for(int index) {
    return substitute(^^history_slot, {std::meta::reflect_constant(index)});
}

//And iterate up the values of index until we find the next undefined value
consteval int next_id() {
    int index = 0;
    while (is_complete_type(slot_for(index))) {
        ++index;
    }
    return index;
}

//And separate defining the next slot
//We must define the injection in a consteval block
consteval void take_slot() {
    define_aggregate(slot_for(next_id()), {});
}

static_assert(next_id() == 0);

consteval { take_slot(); }

static_assert(next_id() == 1);
```
Godbolt [here](https://godbolt.org/z/Yr8ov6Mv6)

This has a few advantages over friend injection - the code is much more transparent about what it's doing and, minus the consteval block requirement for `define_aggregate` calls, isn't relying on implicit quirks of the standard which require multiple citations to follow. And it is of course possible to reengineer the opt-in/opt-out mechanism with this functionality. But reflection lets us take this a step further - because you can now wire up almost anything to anything else, what if we wanted to add metadata about the opt-in and opt-out at the point where it happens?

```cpp
//Create the toggle with a reflectable reason
//why we toggled it
struct opt_in_t {
    bool enabled = true;
    const char* reason = "";
};

namespace widgetising {
    //Same mechanism as before with a type parameter for our user class
    template<typename T, int N>
    struct history_slot;

    consteval std::meta::info slot_for(std::meta::info cls, int index) {
        return substitute(^^history_slot, {cls, std::meta::reflect_constant(index)});
    }

    consteval int entries(std::meta::info cls) {
        int index = 0;
        while (is_complete_type(slot_for(cls, index))) {
            ++index;
        }
        return index;
    }

    consteval void record(std::meta::info cls, opt_in_t opt_in) {
        auto annotation = std::meta::reflect_constant(opt_in);
        define_aggregate(slot_for(cls, entries(cls)), {data_member_spec(^^int, {.name = "entry", .annotations = {annotation}})});
    }

    //Each slot holds a single member, annotated with the opt_in_t it records
    consteval opt_in_t entry(std::meta::info cls, int index) {
        auto member = nonstatic_data_members_of(slot_for(cls, index), std::meta::access_context::unchecked())[0];
        return extract<opt_in_t>(annotations_of(member)[0]);
    }

    //The most recent entry wins, and no entries means not opted in
    consteval bool enabled(std::meta::info cls) {
        int count = entries(cls);
        return count > 0 && entry(cls, count - 1).enabled;
    }
}

struct user_class {};

constexpr bool before_opting_in = widgetising::enabled(^^user_class);

consteval { widgetising::record(^^user_class, {true, std::define_static_string("initial support")}); }
constexpr bool after_opting_in = widgetising::enabled(^^user_class);

consteval { widgetising::record(^^user_class, {false, std::define_static_string("broken by v2")}); }
constexpr bool after_opting_out = widgetising::enabled(^^user_class);

//Works just as well on a class we could never edit
consteval { widgetising::record(^^std::string, {true, std::define_static_string("third party")}); }

int main() {
    std::println("Before opting in:  {}", before_opting_in);
    std::println("After opting in:   {}", after_opting_in);
    std::println("After opting out:  {}", after_opting_out);
    std::println("std::string:       {}\n", widgetising::enabled(^^std::string));

    std::println("History of user_class:");
    template for (constexpr int index : std::define_static_array(std::views::iota(0, widgetising::entries(^^user_class)))) {
        constexpr opt_in_t opt_in = widgetising::entry(^^user_class, index);
        std::println("  {}: {:<5}  ({})", index, opt_in.enabled, opt_in.reason);
    }
}
```
Godbolt [here](https://godbolt.org/z/1j3odheva).

And here we are - you don't need to thread lambdas throughout the code to trick the compiler into giving a fresh specialisation, you can opt any class in or out, regardless of whether you own it. More interestingly, we can persuade the compiler to embed comments on why we're doing what we're doing in the code as we are doing it, and query them later when we try to figure out the history. Indeed, this is an interesting mechanism quite apart from the breaking of opt-ins and opt-outs and horrible IFNDR tricks the rest of this section gives.

## Conclusion

So where does this leave us? I chose the topic of this post not just because it's mildly interesting code, but because in many ways it's a microcosm of the fashions of template programming over the years. C++ has gone from needing to inject an arbitrary name into your class and hope for the best, to methods where you can inject an arbitrary class under a name and see what happens. But in more practical terms:

* The chief contention in deciding on a good opt-in mechanism is between tightly coupling the opt-in to the class definition (and therefore blocking third party code), and the potential IFNDR issues which come from detaching the opt-in. This is a solvable problem, but requires you to be very careful about how such code enters all TUs in your program.
* C++26 annotations are probably the best answer for the tightly coupled approach - it's trivial to make their names clear, make their effect conditional, and even generalise and reuse the code to do so as library code.
* Stateful metaprogramming exists. It allows for all sorts of wizardry which the officially-supported language is still trying to design; but you shouldn't do it in your own code because there's a very good chance that if it isn't broken now, it will be broken soon.