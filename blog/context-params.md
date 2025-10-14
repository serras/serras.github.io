# Context parameters and API design

_by Alejandro Serrano Mena ([website](https://serranofp.com/), [Twitter](https://twitter.com/trupill), [Bluesky](https://bsky.app/profile/serranofp.com))_

**Context parameters** are one of the big features introduced in [Kotlin 2.2](https://kotlinlang.org/docs/whatsnew22.html#preview-of-context-parameters) (the [KEEP proposal](https://github.com/Kotlin/KEEP/blob/main/proposals/KEEP-0367-context-parameters.md) provides all details, but is not required reading for this post). Although the technical part is quite clear, the community is still debating how to better utilize context parameters, and how to design APIs that use them. This blog post describes the _conceptual_ model that I find more useful to guide such decisions, and _examples_ on how to apply that guidance.

> This blog post discusses my opinions about API design after the advent of context parameters into the Kotlin language, in whose development I took part. This should not be taken as the official opinion of the Kotlin Team.

Let us quickly review the main ingredients of context parameters. At first sight, context parameters are just like value parameters, except you define them at the beginning of the signature:

```kotlin
context(users: UserService) fun User.getFriends() { ... }
```

By doing so you no longer need to explicitly pass the `users` argument if there's some value in the context (usually referred to as _implicit value_) of the same type. That context is comprised of context parameters to the function, and also every receiver in scope (this last part is important, as we shall see later).

```kotlin
context(users: UserService) fun User.summarize(): String {
    // ...
    val friends = getFriends()  // 'users' is "contextually resolved"
    // ...
}
```

One of the main goals of the context parameters design is to _minimize noise_. In the example above, you may only call methods on `users` deep in the call chain, but you still need to carry it around on every other function using it. Without context parameters you'd need to manually pass the argument, shadowing the real logic.

As hinted above, the interaction between _receivers_ and context parameters is quite important in this discussion. Before the advent of context parameters, there was an established pattern of using extension receivers to obtain the same implicit passing behavior.

```kotlin
fun UserService.getFriends(user: User) = ...
```

It is no accident, thus, that the first iterations of this feature talked about multiple _receivers_. Still nowadays, most introductions to context parameters draw analogies with receivers — but this also becomes the source of problems. It seems that Kotlin has _two_ ways to do the same thing, so what should be chosen in each case?

## The spotlight principles

Let me tell you, dear reader, how I imagine a piece of Kotlin source code (especially when talking about object-oriented languages). It is not a boring mathematical function, nor a bunch of electrons running inside a chip: it is a **play**, a wonderful theatrical performance. At each given moment, just a handful of characters have the **spotlight**: they correspond to every _receiver_ in scope. They should be well-known to us at that point in the code, this is why the language allows us to access their members without any qualification.

Any other implicit value is just a **secondary** character in the scene. Think of any TV series, and how every time a secondary character enters the scene the main ones (with the spotlight) always something like "remember John, my cousin with two kids?". In Kotlin terms, you need to _call_ the implicit value you want to enter the scene; context parameters have a _name_, after all. The **first spotlight principle** distills this idea: it is fine for a value in the spotlight to function as a secondary character implicitly, but you need to be _explicit_ whenever you want a secondary to enter the spotlight. Going back to Kotlin, this is the explanation behind the fact that the compiler may use receivers to "fill" context arguments, but not the other way around.

The **second spotlight principle** subscribes the idea that we, as humans, can only keep track of a handful of characters at a time. Kotlin follows that rule by providing a very limited number of "spotlight arguments": _one_ if using extension receivers, that may be extended to _two_ if the function is defined inside a class, that acts as dispatch receiver.

## Ok, but when to use context parameters?

My point with that rant is that _receivers_ and _context parameters_ serve two very different purposes in an API, and this difference should be accounted for. Let's begin by discussing a few scenarios that are very well served by context parameters.

The first group are **invisible contexts**, those that never get the spotlight. A prime example is a logger, that (almost) never acts as the main character in your little function/scene. In Kotlin terms, the most common pattern is an interface with a single method exposed via a bridge as a contextual function. On top of that single function you often build a larger API.

```kotlin
interface Logger {
    internal fun log(level: LogLevel, group: String?, message: String)
}

// bridge function
context(logger: Logger) fun log(level: LogLevel, group: String?, message: String) =
  logger.log(level, group, message)

// rest of the API
context(logger: Logger) fun log(message: String) = 
  log(LogLevel.MEDIUM, null, message)
```

The [`Raise`](https://arrow-kt.io/learn/typed-errors/working-with-typed-errors/) interface from the [Arrow project](https://arrow-kt.io/) provides another example of those invisible contexts (full disclaimer: I took part on the development of `Raise`). The library provides a [fully contextual interface](https://github.com/arrow-kt/arrow/blob/main/arrow-libs/core/arrow-core/src/commonMain/kotlin/arrow/core/raise/context/RaiseContext.kt), in addition to the [older one using extensions](https://github.com/arrow-kt/arrow/blob/main/arrow-libs/core/arrow-core/src/commonMain/kotlin/arrow/core/raise/Raise.kt). `Raise` should never be the spotlight, it just provides additional convenience for describing non-happy paths. On a separate note, this group is for me the only one in which using _unnamed_ context parameters (`_`) is advisable.

```kotlin
context(_: Raise<Error>) fun validatePerson(name: String, age: Int): Person
```

The second group served by context parameters are **leaf-only contexts**. Those cases are all quite similar to the `UserService` discussed above: you only really access the context parameters at the very last point of the call chain (the "leaves"), and in the rest of the functions you only carry it around. In this scenario context parameters work as poor man's (although type safe) _dependency injection_.

Another example can be found in the Kotlin compiler, in particular in the [checkers module](https://github.com/JetBrains/kotlin/tree/master/compiler/fir/checkers/src/org/jetbrains/kotlin/fir/analysis/checkers). The `check` function that must be implemented for each checker has two context parameters, and in most cases you only "touch" `reporter` if you need to report an error, but otherwise the contexts are just threaded to other functions.

```kotlin
object FirDataObjectContentChecker : FirSimpleFunctionChecker(MppCheckerKind.Common) {
    context(context: CheckerContext, reporter: DiagnosticReporter)
    override fun check(declaration: FirSimpleFunction) { ... }
}
```

## Building things

In the two categories above we use the spotlight rules to "hide" the values as context parameters. **Builders** are an example in which following the rules leads to the opposite conclusion, namely that we should use a receiver. Most Kotliners know about the `buildList` function,

```kotlin
fun <T> buildList(block: MutableList<T>.() -> Unit): List<T>

// to be used as follows
buildList {
    add(1)
    if (somethingHappens) add(2)
}
```

It is tempting to turn the receiver into a context parameter, but I argue one should _not_ take that step. The `buildList` call is making very clear that we are, well, building a list in the lambda afterwards. This means that the `MutableList` is really in the spotlight, and should remain a receiver.

Even if you accept this reasoning, things become a bit harder when we talk about **serialization** or **document conversion**. As an example, suppose we want to describe how classes should be converted to HTML using the [nice DSL from `kotlinx.html`](https://kotlinlang.org/docs/typesafe-html-dsl.html). Most of the functions in that library are defined as extensions of `FlowContent`,

```kotlin
interface RenderAsHtml {
    fun FlowContent.renderHtml()
}

data class Person(val name: String, val age: Int) : RenderAsHtml {
    override fun FlowContent.renderHtml() {
        div { +"Person: $name" }
        div { +"Age: $age" }
    }
}
```

The question here is whether this is the right API, or `FlowContent` should rather be a context parameter,

```kotlin
interface RenderAsHtml {
    context(content: FlowContent) fun renderHtml()
}
```

According to the spotlight principles, this is not the right move. The purpose of `render` is to build a HTML document, so it makes sense for `FlowContent` to get the spotlight, and should remain a receiver.

This problem is by no means artificial, they are described for real APIs (for example, in [KtMongo by Ivan "CLOVIS" Canet](https://ivan.canet.dev/blog/2025/10/13/kotlin-context-parameters.html#introducing-context-parameters)). If we try to keep builders as context parameters _and_ at the same time keep the same syntax as we have for the receiver version, we easily end up in **bridge function hell**. That is, we end up duplicating the API as both members or extensions of the writer type (`MutableList` for `buildList`, `FlowContent` for `renderHtml`).

One final remark here is that using a _value_ parameter would not solve the problem as neatly as a receiver does. The `kotlinx.html` library uses receivers to define the nesting structure of the HTML document at hand,

```kotlin
inline fun FlowContent.div(crossinline block : DIV.() -> Unit = {}) : Unit
```

and this idea breaks if we defined `renderHtml` as taking a parameter. This example also provides us very concrete advice for library authors: if your API uses nesting or uses [`DslMarker` for scope control](https://kotlinlang.org/docs/type-safe-builders.html#scope-control-dslmarker), and has a larger API surface, think twice before using it as a context parameter. Otherwise define the whole API using context parameters, as done with `Raise` in Arrow.

## The dance of the receivers

Unfortunately, using a receiver in this position comes with an issue: switching the "order" of receivers is sometimes required by the compiler ([the aforementioned blog post on KtMongo](https://ivan.canet.dev/blog/2025/10/13/kotlin-context-parameters.html#what-if-we-used-regular-extension-receivers) explains this problem quite well). The problem is often visible when using interfaces such as `RenderAsHtml` recursively,

```kotlin
data class Person(val name: String, val age: Age) : RenderAsHtml {
    override fun FlowContent.renderAsHtml() {
        div { +"Person: $name" }
        age.renderAsHtml() /* RED: Unresolved reference */
    }
}

@JvmInline value class Age(val value: Int) : RenderAsHtml { ... }
```

The reason for this problem is that `renderAsHtml` expects the `FlowContent` to be given as receiver, but we are using `age` instead. The direct solution is quite convoluted: `with(age) { renderAsHtml() }` is the code required to "flip" the order of the receivers.

Instead of taking this as a reason for dropping receivers in favor for context parameters for `FlowContent`, I advocate for introducing variations of the `renderAsHtml` function that perform this dance for us. One such variation that makes the code above compile is,

```kotlin
context(flow: FlowContent)
fun RenderAsHtml.renderAsHtml() = flow.renderAsHtml()
```

## Summary

In this blog post I advocate for a clearer separation between use cases for _receivers_ and _context parameters_. Not every use case of extension receivers pre-context-parameters need to be transformed into context parameters right away.

| Kind | Usual naming | Context or receiver? | Exposed as |
|--|--|--|--|
| Invisible | `BlahScope` | Context | Core contextual function + contextual API |
| Leaf-only | `BlahService` | Context | Plain interface |
| Builder | `BlahBuilder`, `BlahWriter` | Receiver | Plain interface + "dance" function |
