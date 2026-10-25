---
date: '2026-10-25T17:16:18+00:00'
draft: false
title: 'What is an evaluated concept?'
summary: "What you get when you evaluate a C++ concept, what you get when you don't, and why it matters for reflection and subsumption."
tags: ["c++", "c++26", "reflection", "concepts", "constraints", "subsumption", "wg21"]
---

Like many people, I've been tinkering with C++26 reflection lately, and noticed something quite interesting in a mistake I made. Let's say that you are trying to compare whether two arbitrary types support postfix increment.

```cpp
template<typename C>
concept has_postfix = requires(C c){
    c++;
};

static_assert(substitute(^^has_postfix, {^^A}) == substitute(^^has_postfix, {^^B}));
```

Do you see the mistake in this code?

For those unfamiliar, `std::meta::substitute` (here using ADL to avoid `std::meta`) takes the reflection of a template and a range of reflections of some arguments, and returns a reflection of a template specialised with those arguments. So `substitute(^^has_postfix, {^^A})` returns a reflection of `has_postfix<A>`. But note that it returns a *reflection* - just some `std::meta::info` which is likely implemented as a pointer to whatever AST node holds `has_postfix<A>`. My mistake is that I forgot to splice the result of the function to get a program-level handle on the actual concept itself.

The surprising thing is that the code worked anyway.

But let's expand this further and make sure I didn't just hit on some fluke case:

```cpp
//Convenience wrapper
consteval bool postfix_matches(std::meta::info lhs, std::meta::info rhs){
    //Splicer no splicing
    return substitute(^^has_postfix, {lhs}) == substitute(^^has_postfix, {rhs});
}

static_assert(postfix_matches(^^int, ^^float));

//Also test the case where a concept is unsatisfied
static_assert(not postfix_matches(^^int, ^^std::string));

struct A {};

//And user-defined types
static_assert(not postfix_matches(^^int, ^^A));

struct B {
    B& operator++(int);
};

static_assert(postfix_matches(^^int, ^^B));
```

Both gcc and the experimental P2996 Clang fork accept this code ([Godbolt](https://godbolt.org/z/Y5Go3j9br)).

So what's going on here? Well, [[expr.eq]](https://eel.is/c++draft/expr.eq#7) tells us that two objects of type `std::meta::info` compare equal if and only if they:

> * are null reflection values,
> * represent values that are template-argument-equivalent ([temp.type]),
> * represent the same object,
> * represent the same entity,
> * represent the same annotation ([dcl.attr.annotation]),
> * represent the same direct base class relationship, or
> * represent equal data member descriptions ([class.mem.general]),

None of these stand out as the obvious candidate (to me, anyway). We are not dealing with null reflections, annotations, base class relationships, or data member descriptions. We don't expect `has_postfix<int>` and `has_postfix<float>` to refer to the same object or entity because that's not how templates work; and being template-argument-equivalent typically means making sure that `Widget<4>` and `Widget<2 + 2>` refer to the same class.

So let's step back. Perhaps this is some hitherto unknown quirk of SFINAE coming to show itself. Let's compare the result with a more traditional non-concept trait to test if we get the same result:

```cpp
template<typename T, typename = void>
struct old_has_postfix : std::false_type {};

template<typename T>
struct old_has_postfix<T, std::void_t<decltype(std::declval<T&>()++)>> : std::true_type {};
//Bonus points for why std::declval<T&> over just std::declval<T>

template<typename T>
constexpr inline bool old_has_postfix_v = old_has_postfix<T>::value;

static_assert(substitute(^^old_has_postfix_v, {^^int}) == substitute(^^old_has_postfix_v, {^^float})); //FAILS
```

The assertion fails, so evidently not. And if we dig a little deeper, we find another difference - the result of `substitute` on a concept seems to misplace the arguments which got there:

```cpp
//The "base case". old_has_postfix_v<int> obviously has a template argument - int
static_assert(has_template_arguments(substitute(^^old_has_postfix_v, { ^^int }))); //PASSES

//And we expect that has_postfix<int> should have one too
static_assert(has_template_arguments(substitute(^^has_postfix, { ^^int }))); //FAILS
```

This is surprising, but narrows it down cleanly for us - the issue seems to come from the fact we're reflecting on concepts.

## What is a concept, anyway?

It's easy to think of a concept like any other template, because syntactically it strongly resembles one. The very first point in [[temp.concept]](https://eel.is/c++draft/temp.concept) explicitly states that a concept is a template. But then most of the rest of the section is about all the ways a concept is different from a template - it cannot be instantiated, explicitly or partially specialised, and it cannot itself be constrained.

But if a concept cannot be instantiated or specialised, what are we actually naming when we type `has_postfix<int>` in code? [[temp.names]](https://eel.is/c++draft/temp.names#15) is very clear on this - it names a prvalue of type `bool`, whose value is whether the concept's constraint is satisfied. This seems to clear up half of the reason why we get the result we do when reflecting - `has_postfix<int>` and `has_postfix<float>` are not separate, bespoke entities in the way that specialisations of other templates would be, but just `bool` values.

So returning to the list of reasons for `meta::info` to compare equal, perhaps `substitute` doesn't just give us *a* `bool` value, but instead gives us *the same* internal `bool` object for all satisfied or unsatisfied concepts. Let's investigate:

```cpp
//Theory holds when we compare different concepts (in this case of different arities)
static_assert(substitute(^^has_postfix, {^^int})
            == substitute(^^std::convertible_to, {^^int, ^^float}));

//But is weakened by checking against other bool-producing tools
static_assert(substitute(^^has_postfix, {^^int})
            == std::meta::reflect_constant(true));

static_assert(substitute(^^has_postfix, {^^int})
            == std::meta::reflect_constant(my_true_returning_function()));
```

But is this really the case, and is it required?

The standard doesn't seem to have a clear answer here. If we look at the specification for `substitute`, [[meta.reflection.substitute]](https://eel.is/c++draft/meta.reflection.substitute#7) states that the function returns `^^Z<TARG-SPLICE(Args)...>` for template `Z`, where `TARG-SPLICE` is a mechanism to splice each template argument into position in the template argument list. So, `substitute(^^has_postfix, { ^^int })` is specified to return `^^has_postfix<int>`. There is no special case or carve-out for concepts. Indeed, the `can_substitute` metafunction to determine valid substitutability only requires that `Z<TARG-SPLICE(Args)...>` is a valid *template-id*, and `has_postfix<int>` is.

But if we actually try to type `^^has_postfix<int>` in code, we will find that our code is ill-formed. gcc will even spell out the problem in its diagnostic: "error: '^^' cannot be applied to a concept check" ([Godbolt](https://godbolt.org/z/s8z6nTs81)). Looking at the spec, [[expr.reflect]](https://eel.is/c++draft/expr.reflect#7) provides a list of valid things which can be reflected on in the form `^^id-expression`; and none of them are a *concept-id*, so the program is ill-formed. The result is that `substitute` can be used to return a reflection which the core wording says cannot be formed.

There is some hint about the intention here. The revision history on [P2996](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p2996r13.html#pnum_841) highlights the issue of reflecting on concepts, and even mentions that `substitute` can be used to evaluate whether a concept is satisfied, which helps us part of the way but still says little about exactly what the function must return. The authors' intentions are there but we never received normative wording in the standard to clarify or enforce this.

But let's say that we accept that the result of `substitute` is the reflection of some `bool`. We have actually already covered why we see the phenomenon that we do. Recall that two `meta::info` compare equal if they:

> * are null reflection values,
> * represent values that are template-argument-equivalent ([temp.type]),
> * represent the same object,
> * represent the same entity,
> * represent the same annotation ([dcl.attr.annotation]),
> * represent the same direct base class relationship, or
> * represent equal data member descriptions ([class.mem.general]),

And let's hone back in on template-argument-equivalent. [[temp.type]](https://eel.is/c++draft/temp.type#2) says that two values are template-argument-equivalent if, among other possibilities, they are of integral type and their values are the same. `bool` is an integral type. If `substitute` has to return a reflection of a `bool` value then there's nothing more to it: `has_postfix<int>` and `has_postfix<float>` both reduce to a `bool` prvalue, two `true`s are equivalent and so their reflections must compare equal. Whether this is powered by some internal shared constant is irrelevant and unobservable, because a `bool` value is just a value - it doesn't have identity of its own.

This might seem to answer the question - naming a concept gives us a `bool`, and `substitute` gives us a reflection of a `bool`. But there's a reason the standard doesn't just accept this and move on. Concepts are not that simple.

## What about constraints?

A *concept-id* can appear in two contexts in your code - it can be evaluated as an expression or appear as a constraint. So far we have been evaluating it as an expression, and we have seen the specification dictate that in this case it is a prvalue of type `bool`. But what about in a constraint? Given this code:

```cpp
template<typename T> requires has_postfix<T>
void foo(){ ... }
```

Do we eagerly substitute our type into `T` and get a `bool` prvalue, as we would for a *concept-id* used in an expression? Does this just reduce the constraint to `requires true` or `requires false`? The answer is that it's not quite that simple. While we do need to check whether a constraint is satisfied to determine if the function is viable to call, the compiler doesn't stop there. Unlike traditional SFINAE-based ways to constrain a template to particular specialisations, constraints participate in subsumption. If the compiler detects that one candidate is "more specialised" than the others, it chooses that one for you without the need to engineer mutual exclusivity of other candidates yourself. To demonstrate this, let's add another constrained overload of `foo()`.

```cpp
template<typename T> requires has_postfix<T>
void foo(){ ... }       //1

template<typename T> requires has_postfix<T> && std::integral<T>
void foo(){ ... }       //2
```

In this case, we expect that `foo<int>` would call overload `2` and `foo<float>` would call overload `1`, thanks to subsumption. But in order for this to work there has to be more going on than just replacing a satisfied *concept-id* with `true`.

Instead, constraints are normalised according to [[temp.constr.normal]](https://eel.is/c++draft/temp.constr.normal). Compound constraints are recursively split out, and normalised, so the normal form of `has_postfix<T> && std::integral<T>` is the conjunction of the normal forms of `has_postfix<T>` and `std::integral<T>`. Each individual *concept-id* is then recursively expanded, so `has_postfix<T>` is replaced by the normal form of `requires(T c){ c++; }` and `std::integral<T>` is normalised to whatever its definition is in your `<concepts>` header. This continues until every remaining piece is an *atomic constraint*, which can't be subdivided further. For the case of `foo()`, let's call the result of these A and B respectively, such that the normal form of the constraint on function `1` is A, and the normal form of the constraint on function `2` is A ∧ B. It's important to note that at no point so far have we substituted a real type like `int` for `T` or produced a `bool`; we have only rewritten the constraint as a logical formula over atomic constraints.

These constraints are then checked when we attempt to call the function. [[over.match.viable]](https://eel.is/c++draft/over.match.viable#3) tells us that, unsurprisingly, for a function to be viable all of its associated constraints must be met. For `foo<int>()` both A and B are satisfied, so both overloads are viable. For `foo<float>()`, only A is satisfied so only overload `1` is viable. Because we have reduced the viable overload set to 1 candidate, this candidate is chosen.

In the `foo<int>()` case, [[temp.constr.order]](https://eel.is/c++draft/temp.constr.order) determines which overload is the preferable match. It asks two questions:

* Does the constraint on `2` subsume the constraint on `1`? In this case, the clause `A ∧ B` contains `A`, so yes.
* Does the constraint on `1` subsume the constraint on `2`? In this case, `B` appears in `A ∧ B` but not in `A`, so no.

As such, `2` is considered more constrained and is picked.

For completeness, a disjunction works the other way around: adding an alternative widens the set of types the constraint accepts, so the overload with the extra alternative is the less constrained one:

```cpp
template<typename T>
concept is_enumeration = std::is_enum_v<T>;

template<typename T> requires has_postfix<T>
void bar(){ ... } //3

template<typename T> requires has_postfix<T> || is_enumeration<T>
void bar(){ ... } //4
```

Normalisation proceeds as before. Function `3` has normalised constraint `A`, and function `4` has normalised constraint `A ∨ E`. When calling `bar<int>()`, both overloads are viable. We ask these two questions:

* Does the constraint on `3` subsume the constraint on `4`? In this case, that would require that `A` guarantees `A ∨ E`, so yes.
* Does the constraint on `4` subsume the constraint on `3`? In this case, that means asking if each alternative of `A ∨ E` guarantees `A`. We know this is not the case, as enumeration types do not guarantee postfix increment operators, so no.

As such, `3` is considered more constrained and is what the compiler chooses.

## How the compiler sees atoms

But how does the compiler know this? How does it form an idea of A ∧ B? For example, if we had instead reversed the concepts in code and written `requires std::integral<T> && has_postfix<T>`, how would the compiler determine that this is equivalent to the same A ∧ B above?

The standard gives an answer which, as is traditional, tries to pretend that compilers and files don't exist. [[temp.constr.atomic]](https://eel.is/c++draft/temp.constr.atomic#2) says that two atomic constraints are identical "if they are formed from the same appearance of the same expression" and their parameter mappings match. The words "same appearance" strongly suggest that this is an artifact of position in the source code. This makes sense, considering that comparing two token sequences to determine if they are different ways to spell the same concept is non-trivial, but it does mean that token-identical concepts which appear in multiple places do not subsume each other, or rather:

```cpp
//Token-identical definitions
template<typename T> concept PF1 = requires (T c){ c++; };
template<typename T> concept PF2 = requires (T c){ c++; };

template<typename T> requires PF1<T>
int f(T);
template<typename T> requires PF2<T> && std::integral<T>
int f(T);

f(0); //AMBIGUOUS
```

`PF1` and `PF2` are identical when evaluated as an expression to determine whether a type is postfix incrementable, and will both reduce to a prvalue of type `bool`; but when used as a constraint they are entirely distinct and unrelated atoms in the constraint list.

This is generally sensible, as duplicating your concepts should be vanishingly rare; but the traps come when you try to use conditions which aren't represented as concepts:

```cpp
template<typename T> requires (sizeof(T) == 4)
int g(T);
template<typename T> requires (sizeof(T) == 4) && std::integral<T>
int g(T);

g(0); //AMBIGUOUS
```

Despite the second condition being trivially more constrained than the first, the fact that the `sizeof(T) == 4` check does not come from the same appearance of the same expression means the compiler treats them as entirely separate atoms. If we were to pull them out into a named concept all their own then our code would work:

```cpp
template<typename T>
concept size_four = (sizeof(T) == 4);

template<typename T> requires size_four<T>
int g(T);
template<typename T> requires size_four<T> && std::integral<T>
int g(T);

g(0); //UNAMBIGUOUS
```

Recall that when a constraint is normalised a concept is recursively expanded into its base conditions. The above example could be equivalently written:

```cpp
template<typename T>
concept size_four = (sizeof(T) == 4);
template<typename T>
concept size_four_and_integral = size_four<T> && std::integral<T>;

template<typename T> requires size_four<T>
int g(T);
template<typename T> requires size_four_and_integral<T>
int g(T);

g(0); //UNAMBIGUOUS
```

As such, concepts can be used to construct the building blocks of constraint logic in your program. They are intended to be combined both in the concept definition and the constraint list in order for you to express the constraints in a simple and predictable way.

Duplicates of expressions in different concept definitions are *different appearances* and do not subsume. To take the above example further:

```cpp
template<typename T>
concept size_four = (sizeof(T) == 4);
template<typename T>
concept size_four_and_integral = (sizeof(T) == 4) && std::integral<T>;

template<typename T> requires size_four<T>
int g(T);
template<typename T> requires size_four_and_integral<T>
int g(T);

g(0); //AMBIGUOUS
```

Despite the expression to define the size of four being identical, and therefore the normalised form of each constraint being token-identical to that of the previous, working example, the fact that our `(sizeof(T) == 4)` expression exists in multiple places in the source code means that it cannot participate in subsumption. As such, when designing with subsumption in mind, you should wrap your base condition expressions in a concept and then combine *concepts* rather than combining expressions.

It is worth noting here that expressions which contain a concept are still expressions. If `g()` had instead required `!size_four<T>` on both overloads then we would be back to the original problem of two different atoms from two different appearances in the code, and an ambiguous call.

This isn't exclusive to expressions using operators though. Let's bring this all the way back to reflection:

```cpp
template<typename T> requires [: substitute(^^has_postfix, {^^T}) :] //We need exactly bool so we need to splice
int h(T);
template<typename T> requires [: substitute(^^has_postfix, {^^T}) :] && [: substitute(^^std::integral, { ^^T }) :]
int h(T);

h(0); //AMBIGUOUS
```

`std::meta::substitute` is just a `consteval` function which in this case returns a `std::meta::info` representing a `bool`. The compiler is unable to determine subsumption because each appearance of `substitute` in code is unique. But, as before, if we were to wrap them in a concept:

```cpp
template<typename T>
concept silly_postfix = [: substitute(^^has_postfix, { ^^T }) :];
template<typename T>
concept silly_integral = [: substitute(^^std::integral, { ^^T }) :];

template<typename T> requires silly_postfix<T>
int h(T);
template<typename T> requires silly_postfix<T> && silly_integral<T>
int h(T);

h(0); //SILLY BUT UNAMBIGUOUS
```

This brings us to the advice in our new world of reflection and `consteval` functions - if they are intended as constraints or check properties of things which could be template arguments, wrap them in a concept. You should make one concept per check and combine them by combining concepts, not functions.

This comes with some caveats which you should know, however:

* Inside the constraint, the type of the condition must be exactly `bool`. A `consteval` function which returns `int` gives a hard error rather than converting its return value to `bool`.
* Failures which aren't substitution failures will cause a hard error - if an exception escapes the `consteval` function then your program is ill-formed.
* If your reflection is stateful, and the result of the function is different in different parts of the program, then any concept which depends on the result renders that program ill-formed, no diagnostic required. This might include `is_complete_type(^^T)` or `members_of(^^NS)` whose results change as definitions and members are added.
* *Primary-expression*s (e.g. naming `true` or `false`, splices, or *concept-id*s) can be unparenthesised, but all other expressions must be parenthesised.
* Individual constraints are separated by `&&` and `||` in the constraint on the declaration, and any other operator reduces the constraint to a single atom. So `requires C<T>` can be subsumed by other constraints, but `requires (C<T> == true)` cannot.

It is also important to note that this may not matter in the trivial cases resembling one function with one constraint. The concern arises when subsumption must be applied - the situation where there exists more than one overload whose constraints are viable and the compiler must pick which of them is the more constrained.

Equally, partial mixing works. For a concept `C`:

```cpp
template<typename T> requires C<T>
void j(T);
template<typename T> requires C<T> && (f(^^T))
void j(T);

j(0); //UNAMBIGUOUS
```

The compiler is still able to reduce this to one function with constraint `C` and another with constraint `C ∧ f` and determine subsumption. If, however, you'd used `f(^^T)` as part of the constraint of another overload of `j()` then you would likely have an ambiguous call.

## Conclusion

Concepts are weird. They behave very differently from the other kinds of template in C++, because they represent a logical building block rather than something which exists in your program. It's unlikely that you're going to be reflecting on them any time soon, not least because there is an intentional gap in the specification of what reflecting on a *concept-id* means, and there is no way to reflect on a template's constraints.[^displaystring] However, you can use `substitute()` to determine whether a concept would be satisfied for a set of template arguments, and this follows the pattern of using a concept in an expression and simply gives you a `bool`, losing all information about where that `bool` came from.

[^displaystring]: Technically `std::meta::display_string_of(^^function)` will show the constraints in its returned string in gcc 16; however, the contents of this string are entirely implementation defined and there are no guarantees that it will continue to do so. It is intended that you should *not* base any logic on the result of this function in the same way you shouldn't rely on the result of `typeid(T).name()`.

But in more practical terms:

* Concepts are intended to be able to represent a single constraint or a conjunction or disjunction of constraints, and are uniquely able to have the atoms of that set of constraints expanded and considered individually.
* Expressions other than *concept-id*s written directly in a *requires-clause* are atoms of that one constraint only. They cannot be shared with another declaration and so cannot participate in subsumption, even if they are identical.
* If you are writing some non-concept check on a type or value which you intend to be used to constrain a template, you should wrap it in a concept and combine it with other checks by combining concepts, not unwrapped checks.
* Concepts should be positive: `requires C<T>` can subsume and be subsumed. `requires (!C<T>)` cannot. `requires not_C<T>` can again. Checks which require negation should go into their own concept.