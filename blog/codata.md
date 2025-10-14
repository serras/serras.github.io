# The marriage of FP and OOP (WIP)

_by Alejandro Serrano Mena ([website](https://serranofp.com/), [Twitter](https://twitter.com/trupill), [Bluesky](https://bsky.app/profile/serranofp.com))_

In the programming world, we often speak about the different approaches taken by _functional_ and _object-oriented_ programming. Most mainstream languages nowadays feature a blend of both styles. This blend oftentimes seems ad hoc, just two things put together because they go well together. But if we look a bit further, we discover an intriguing duality between both approaches, which helps us to understand both sides of the coin more uniformly. This journey is going to take a while, but it will definitely give you a different view of programming.

> This (long) post is inspired by the ideas in [_Deriving Dependently-Typed OOP_](https://arxiv.org/ftp/arxiv/papers/2403/2403.06707.pdf), [_Codata in Action_](https://pauldownen.com/publications/esop2019.pdf) and [_Compiling without Continuations_](https://pauldownen.com/publications/pldi17.pdf).

* [Data and interfaces (= codata)](#data-and-interfaces--codata)
* [The same thing, in two ways](#the-same-thing-in-two-ways)
    * [The expression problem](#the-expression-problem)
    * [Where is the duality?](#where-is-the-duality)
* [Functions are not built-in](#functions-are-not-built-in)
* [Inheritance](#inheritance)
    * [Taking out the sugar](#taking-out-the-sugar)
* [Generalized (Co)data Types](#generalized-codata-types)
* [Join points](#join-points)
    * [Other uses of join points](#other-uses-of-join-points)
* [Conclusion](#conclusion)

## Data and interfaces (= codata)

Let us begin our journey by introducing our protagonists: the two different ways to define _types_ (which we could also see as _schemas_ defining some shape), and _values_ following those types, also known as _instances_ in some circles.

_Data_ types are defined by their _constructors_, each of them storing a sequence of values in _fields_. A fancier name for them is [algebraic data types](https://en.wikipedia.org/wiki/Algebraic_data_type), but they are also known as [enumerations](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/enumerations/) in Swift, or can be emulated using [data classes](https://kotlinlang.org/docs/data-classes.html) and [sealed interfaces](https://kotlinlang.org/docs/sealed-classes.html) in Kotlin.

```kotlin
data User {
    Person(name: String, age: Integer, address: Address)
    Company(legalName: String, vatNumber: String, country: Country)
}

data Country {
    Albania()
    Andorra()
    ...
}
```

A value of such data type is constructed by giving the constructor we use, and values for each of the fields.

```kotlin
def user = User.Company("Alex Inc.", "1234", User.Netherlands)
```

At this point, we want to keep the data type itself separate from the constructors. Later in this article, we shall look at a way to reconcile them, leading to the approach in which subtyping emulates constructors seen in languages like Kotlin or Scala.

_Interface_ types, in contrast, are defined by a set of _methods_, each of them taking arguments to some result. In academic circles you sometimes see the word _codata_ instead of "interface", and _destructor_ instead of "method". This hints already towards the dual relation between the two concepts. Interfaces are a well-known constructor, here we have an example of a generic one.

```kotlin
interface Collector<A, R> {
    initialValue(): R
    accumulate(old: R, new: A): R
}
```

A value of such data type is constructed by defining how each of the methods should behave. Later in the article, we shall see where the notion of _class_ enters the game. For the time being, instances are always defined "on the spot".

```kotlin
def summer = object : Collector<Integer, Integer> {
    initialValue() = 0
    accumulate(old: Integer, new: Integer) = old + new
}
```

Functional programming tends to favor data-oriented programming. Pattern matching, for example, is a powerful way to inspect values, which assumes that every piece of data is built by piling up calls to constructors. On the other hand, object-oriented programming favors interface-oriented programming, in which values describe some behavior instead of storing data.

## The same thing, in two ways

> The examples in this section are taken from [_Deriving Dependently-Typed OOP_](https://arxiv.org/ftp/arxiv/papers/2403/2403.06707.pdf).

The description above seems to divide the reality (or at least the concepts we want to model in programming) into two distinct sorts. Those "aggregating information" are defined by data, those "specifying behavior" by interfaces. However, the cut is not as clear as it may seem. Let us consider Booleans, for example; the usual representation is an enumeration of two entries, which using our lenses becomes a data type with two constructors.

```kotlin
data Boolean {
    True()
    False()
}
```

To define an _operation_ over such type we match on the potential constructors. Note that we have been careful to not mention the word "function", using instead `def` to stand for "definition". If you are a functional programmer, try to leave aside all the things you know you can do with functions; our definitions define simple substitution for all their arguments, but are not values themselves, they cannot be partially applied, ...

> For a complete treatment of definitions, and their distinct role in a programming language, we redirect the reader towards the book [_Type Theory and Formal Proof_](https://www.cambridge.org/core/books/type-theory-and-formal-proof/0472640AAD34E045C7F140B46A57A67C).

We assume here that our language allows definitions using [extension syntax](https://kotlinlang.org/docs/extensions.html) as found in Kotlin.

```kotlin
def Boolean.not(): Boolean = when (this) {
    True()  -> False()
    False() -> True()
}
```

How do we define Booleans using interfaces instead? In this case, the behavior we are interested in is negation, so this is the method to include in the interface:

```kotlin
interface Boolean {
    not(): Boolean
}
```

Each of the two Boolean values becomes now definitions instead, whose methods define the `not` behavior.

```kotlin
def True = object : Boolean {
    not() = False
}

def False = object : Boolean {
    not() = True
}
```

In this case, we have focused on a _single_ operation on Booleans, but in many cases, there is a set of behaviors "generic" enough to define exactly what the type represents. _Pairs_ are an example, which can be defined as a data type that holds two values, or by an interface defining both projections.

```kotlin
data PairD<A, B> {
    Pair(x: A, y: B)
}

interface PairI<A, B> {
    first(): A
    second(): B
}
```

As an example, we can go from the interface to the data one, and vice versa. Note that even though `PairD` has a single constructor, we still use pattern matching to access its fields; the goal here is to use the least amount of "syntactic sugar", in order to understand everything going on under the surface.

```kotlin
def PairI<A, B>pairIToD(): PairD<A, B> = Pair(this.first(), this.second())

def PairD<A, B>.pairDToI(): PairI<A, B> = when (this) {
    Pair(x, y) -> object : PairD<A, B> {
        first = x
        second = y
    }
}
```

### The expression problem

Our two ways to define Booleans above showcase an interesting challenge in programming known as the [expression problem](https://en.wikipedia.org/wiki/Expression_problem). In a nutshell, data and codata have opposite extensibility properties. In the case of data, it is very easy to define a new operation,

```kotlin
def and(x: Boolean, y: Boolean) = when (x) {
    True() -> y
    False() -> False()
}
```

Conversely, in the case of interfaces, it is very easy to define a new value. For example, if we want to include the possibility of an unknown value.

```kotlin
def Unknown = object : Boolean {
    not() = Unknown
}
```

On the other hand, adding such `Unknown` value to the data-based scenario means that we need to change the _original_ definition. This has a potential huge impact on the rest of the codebase. A similar problem arises if we want to include the `and` operation as one of our methods in the interface-based scenario: we need to change the _original_ definition, which means that every value creation now must define the new method.

### Where is the duality?

Hopefully, at this point you are somehow convinced that there is a relation between both approaches to programming. But why do we speak of _duality_? To understand it, it is useful to consider the types we would assign to constructor and destructors. The duality is a bit more obvious if we consider those types [_curried_](https://www.baeldung.com/scala/currying), that is, taking one argument at a time.

```kotlin
Person  : String -> Integer -> Address -> User
Company : String -> String  -> Country -> User

initialValue : Collector<A, R> -> R
accumulate   : Collector<A, R> -> R -> A -> R
```

What you can see is that in constructors the types of the fields appear _first_, and the data type itself appears _at the end_. Interfaces are exactly the opposite, with the interface _at the beginning_, and the arguments and return type _afterwards_. That's it!

In the rest of this post, we are going to look at what are the consequences of this duality, and also how programming languages sometimes hide these two subtle flavors behind too much sugar.

## Functions are not built-in

Functional programmers may have noticed an omission in our set of basic types: we have no function type! Or better said, we have no _single_ function type, but rather we can define as many as we want using interfaces.

```kotlin
interface Predicate<R> {
    test(value: R): Boolean
}
```

There is one mainstream language that follows this paradigm, namely Java. There every [interface with a single abstract method](https://www.baeldung.com/java-8-functional-interfaces) gets access to functional goodies, like shorter lambda syntax.

```kotlin
def isZero: Predicate<Integer> = (n) => n == 0

// is just shorthand syntax for

def isZero: Predicate<Integer> = object : Predicate<Integer> {
    test(value: Integer) = value == 0
}
```

If you are used to having a single function type in your language, this may seem quite odd. However, compare it with pairs, which are the most generic way of storing two values. Do you use `Pair<String, Int>` to save the name and age of a person? No, you often define a data type for that particular matter.

```kotlin
data Person {
    Person(name: String, age: Integer)
}
```

Using a different type for each concept in your domain is a [useful design guideline](https://arrow-kt.io/learn/design/domain-modeling/). Why should we apply that rule of thumb only for data, but not for codata? In many languages using the function type is the only way to get nice syntax, but as Java shows us, we can achieve an even more useful result by extending that syntax to every single-method interface.

## Inheritance

Until now we have considered only data and interfaces with no relationship between them. Object-oriented programming brings an important concept to interfaces. At any point, we can _extend_ an interface by adding additional methods, and the result is a new interface that _inherits_ the methods of the former.

```kotlin
interface Figure {
    draw(context: DrawingContext, lineColor: Color): Unit
}

interface ColoredFigure : Figure {
    fill(context: DrawingContext, lineColor: Color, fillColor: Color): Unit
}
```

To define a value for the new `ColoredFigure`, we must define both methods, `draw` and `fill`. The extension relationship entails that we can use a `ColoredFigure` everywhere we need a `Figure`, the so-called [Liskov Substitution Principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle).

Inheritance is a well-known concept in object-oriented programming; what happens when we bring it to the data world using our duality lenses? If we think of _extending_ as "adding more stuff", the notion translates to adding more constructors. Here we have a way to define the days of the week, where the bigger `WeekDay` contains all 7 constructors for both kinds of days.

```kotlin
data WorkingDay {
    Monday()
    ...
    Friday()
}

data WeekendDay {
    Saturday()
    Sunday()
}

data WeekDay : WorkingDay, WeekendDay { }
```

Languages that emulate algebraic data types using classes, like Kotlin or Scala, seem to do it "the other way around", though. The design you arrive following those rules is more similar to the following:

```kotlin
sealed interface WeekDay
sealed interface WorkingDay : WeekDay
sealed interface WeekendDay : WeekDay

data object Monday : WorkingDay
...
data object Friday : WorkingDay
data object Saturday : WeekendDay
data object Sunday : WeekendDay
```

We argue in this post that, even though it is correct from a typing perspective, this usual way to emulate data types is conceptually _wrong_. We are conflating the (distinct) concepts of subtyping and extension into a single one. Note that in the emulation the fact that the interfaces are `sealed` plays a big role: this puts us in a "closed world", and henceforth we know the exact set of constructors for the type. However, in the definition of extension given in this post -- where `WeekDay` extends `WeekendDay` -- these acrobatics are no longer needed.

What about typing, then? Remember that when we say that "`T` is a subtype of `S`", we mean that "`T` can be used everywhere `S` can", which more formally means "a value of type `T` can be used as an argument in a position where a parameter of type `S` is required". From that perspective, we have that "a data type `T` extends `S`" means that "`S` is a subtype of `T`" -- like above, whenever we require a `WeekDay`, we can give a `WeekendDay` value. One way we can further strengthen our intuition for that rule is by the fact that, as discussed above, constructors and destructors put the type we are defining in very different positions, which in particular have opposite _variances_. As a result, it is natural that constructors and destructors behave in opposite ways with respect to the subtyping relation.

### Taking out the sugar

The language we have defined so far is comprised of five different elements:

- Data types and constructors,
- Interface (codata) types and methods (destructors),
- Definitions.

Actual programming languages usually introduce a few more. In this section we describe how the other elements can be translated back into the smaller set, a process oftentimes called _desugaring_ in the compiler world.

If you ask any practitioner of object-oriented programming what is the main element in such a language, _classes_ are usually the favorite. A class bundles together two separate concepts: the interface (or API) used to access the class, and a way to construct new instances of the class. We can untangle both parts, turning the former into an interface, and the latter into different definitions.

```kotlin
class Moo : Foo {
    constructor(/* constructor args */)
    // interface
    moo(): Milk = /* moo code */
}

// translates to

interface Moo : Foo {
    moo(): Milk
}

def Moo(/* constructor args */) = object : Moo {
    moo(): Milk = /* moo code */
}
```

This pattern of the definition having the same name as the interface, and acting as a constructor, is used sometimes the mimic the usual object-oriented notion of class. For example, in Kotlin this allows the programmer to work around some of the restrictions for class constructors.

Orienting ourselves now to data, let us consider the usual way of emulating them in languages like Kotlin or Scala.

```kotlin
sealed interface User {
    data class Person(val name: String, val age: Integer): User
    data class Company(val legalName: String, val vatNumber: String): User
}
```

We have already discussed how the inheritance relation is the opposite of the extension relation. However, another difference is that `Person` and `Company` are assigned their own _types_, so we can say that "`Person` is a subtype of `User`". As discussed above, constructors and types are distinct in our formalization, though. The trick to bridge both worlds is that we can always create a type with a single constructor, and then make the bigger type extend all those single-constructor data types.

```kotlin
data Person {
    Person(name: String, age: Integer)
}

data Company {
    Company(legalName: String, vatNumber: String)
}

data User : Person, Company { }
```

One interesting fact regarding single-constructor data types is that we can define nicer syntax to access each of their fields, `person.name`. As in the case of lambda syntax, it seems that having a single way to construct or destruct values lends itself to providing nicer syntax.

## Generalized (Co)data Types

[_Generalized algebraic data types_](https://en.wikipedia.org/wiki/Generalized_algebraic_data_type) (GADTs) form a very useful extension to regular data types. The core idea is the constructors may _refine_ some of the type variables in the type. When we later match on a value of the type, knowing that a certain constructor is used brings additional typing constraints.

The usual example of GADT is that of _well-typed expressions_. Suppose we define a data type to represent expressions in a small language with integers and Booleans, and three operations, namely addition of numbers, negation, and comparison.

```kotlin
data Expression {
    IntLiteral(value: Integer)
    BoolLiteral(value: Boolean)
    Add(a: Expression, b: Expression)
    Not(a: Expression)
    Equals(a: Expression, b: Expression)
}
```

Alas, with this data type definition we can build wrong-typed expressions. `Not(IntLiteral(5))` is an example where a Boolean operation is applied to a number. This may be exactly what you want if you are building a compiler and implementing type-checking yourself, but if you are building a Domain Specific Language (DSL), you may prefer the compiler to take care of this for you.

You can provide a solution by extending the type with a variable that holds its type and making constructors refine the type. Following the usual convention, we write the more refined type as the return type of each constructor.

```kotlin
data Expression<E> {
    IntLiteral(value: Integer) : Expression<Integer>
    BoolLiteral(value: Boolean) : Expression<Boolean>
    Add(a: Expression<Integer>, b: Expression<Integer>) : Expression<Integer>
    Not(a: Expression<Boolean>) : Expression<Boolean>
    Equals<E>(a: Expression<E>, b: Expression<E>) : Expression<Boolean>
}
```

The new duality lenses allow us to define a similar notion of _Generalized Codata Type_. In the same way that in constructors the type you are defining goes at the end, in the case of interfaces it goes at the very beginning. In the case of pairs, we can be more explicit by writing,

```kotlin
interface Pair<A, B> {
    Pair<A, B>.first(): A
    Pair<A, B>.second(): B
}
```

This shows where the refinement should come: at the beginning. As an example, we can extend the notion of `Pair` with a notion that "joins" both components into a single one; but this may only be done when those components have the same type.

```kotlin
interface Pair<A, B> {
    Pair<A, B>.first(): A
    Pair<A, B>.second(): B
    Pair<E, E>.join(): A
}
```

This feature may seem far-fetched, why would you like to restrict the type of a method? One example is [error accumulation](https://arrow-kt.io/learn/typed-errors/working-with-typed-errors/#accumulating-errors) in Arrow's [typed errors](https://arrow-kt.io/learn/typed-errors/) module. The core type in that module is an interface that allows raising an error.

```kotlin
interface Raise<E> {
    raise(error: E): Unit
}
```

The accumulation interface, however, is not available for any error type `E`, but specifically for non-empty lists, where we place each of the errors. In an ideal world, this method would be part of `Raise`, but with a refined type, `Raise<NonEmptyList<E>>.accumulate(...)`. Alas, in the current implementation, we need to define an additional interface.

## Join points

Looking at the duality between data and interfaces as simply "moving the arrow" is a bit handwavy, especially the fact that methods have a result type that data constructors do not. There is a way to make methods only _consume_ values, to do so we need to introduce the concept of _join points_, as discussed in [this paper](https://pauldownen.com/publications/pldi17.pdf).

The inspiration comes from [continuation-passing style](https://en.wikipedia.org/wiki/Continuation-passing_style), a technique where functions do not _return_, but rather take an additional argument describing the _next_ step. Imagine we have our regular `increment` function.

```kotlin
fun increment(x: Int) = x + 1
```

Its CPS version is extended with an additional argument, which is called at the end of execution.

```kotlin
fun incrementCPS(x: Int, next: (Int) -> Unit) = next(x + 1)
```

We compose larger behaviors by building larger continuations. For example, this is how we write a function to add two based on `increment`.

```kotlin
fun addTwoCPS(x: Int, next: (Int) -> Unit) =
  incrementCPS(x) { xPlusOne ->
    incrementCPS(xPlusOne) { xPlusTwo ->
      next(xPlusTwo)
    }
  }
```

Those examples, however, use a function type, which we are trying to avoid. Instead, we introduce a new basic element in our formalization, that of a _join point_. You can think of them as places where we can continue execution. The only thing we can do with such a join point is _jumping_ (or returning) to them, which immediately moves execution to that point. Much like `goto`s in older languages, but with one important difference: join points have a _type_ that describes the arguments they expect.

Since there is no agreed syntax for join points, we are going to use the keyword `join` to describe them, and `return @p` for a jump to join point `p`. Given their similarity with continuations, we are going to use lambda syntax (but with `=>` instead of `->`) to construct new join points.

```kotlin
def increment(x: Int, next: join Int) = return @next (x + 1)

def addTwo(x: Int, next: join Int) =
  increment(x) { xPlusOne =>
    increment(xPlusOne) { xPlusTwo =>
      return @next xPlusTwo
    }
  }
```

Finally, our methods only _consume_ values. What before was the return type becomes now a join point.

```kotlin
interface Collector<A, R> {
    initialValue(next: join R)
    accumulate(old: R, new: A, next: join R)
}
```

The intuition here is that to call a function you actually are providing _two_ pieces of data. The first one is the set of arguments to the function. The second one is the place where the function ought to return once the job is done. This is quite similar to how function calls work at the assembly level, where the stack includes both arguments and the return point.

### Other uses of join points

Join points are only featured in the [Core language of the Haskell GHC compiler](https://pauldownen.com/publications/pldi17.pdf). Their initial goal is to fix the problem of several branches in a pattern match sharing their body. Join points provide a (type-safe) way to jump to the shared code.

Although their application has not been explored, one potential usage is describing how exceptions operate in most languages. Instead of a single join point representing normal return, functions would include join points for exceptional return.

```kotlin
interface UserService {
  userById(id: UserId, next: join User?, problem: join Throwable)
} 
```

Throwing an exception translates to jumping to the `problem` join point, whereas protecting some block of code with a `try` / `catch` translates to using the exception handler as the argument for `problem`.

## Conclusion

The aim of this post was always to open your mind, diving into new ideas coming from academia over the relation between FP and OOP. If you want to go further than GADTs, [_Deriving Dependently-Typed OOP_](https://arxiv.org/ftp/arxiv/papers/2403/2403.06707.pdf) moves in the direction of dependently-typed programming. In this post, we have tried to notation based on `object` to build values of interfaces, but [copattern matching](https://www2.tcs.ifi.lmu.de/~abel/popl13.pdf) provide an interesting twist on pattern matching to describe them.