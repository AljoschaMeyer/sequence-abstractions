# On Sequence Abstractions

Some of the most ubiquitous abstractions in programming are interfaces for lazy manipulation of sequences of data. When programming in the small, they enable convenient computations on data structures. When programming in the large, they form an important mechanism for decoupling pieces of program logic. Sequences can be produced or consumed, they can be finite or infinite, buffered or unbuffered, fallible or infallible, synchronous or asynchronous. Perhaps due to this variety of options, many API designs appear to be ad-hoc, guided more by experience and intuition rather than principled design. Common names for such abstractions are *iterator*, *stream*, *source*, or *sink*.

In this text, we want to suggest some principles that can lead to more consistent and expressive sequence abstractions. These principles are as follows:

- To define APIs for working with sequences, one must first define sequences in a way that characterizes them independently of how they are to be manipulated. This conceptual model is never implemented, so it is fine for its instances to be of infinite size.
- An API for consuming sequences should be chosen such that there is a one-to-one correspondence between abstract sequences and sequence consumers.
- An API for producing sequences should be chosen such that there is a one-to-one correspondence between abstract sequences and sequence producers.
- Corollary: There should be a one-to-one correspondence between sequence consumers and sequence producers.
- These bare-bones APIs can then be extended with effects such as error handling or asynchronicity. This should be done without breaking the one-to-one correspondence between them.

The remainder of the text will derive and justify these principles, as well as provide concrete examples of such APIs. First, we introduce the notation for the types of the functions that make up the APIs (types aid our reasoning, but the API designs work just as well in dynamically typed languages). Next, we search for suitable abstract characterization of sequences. Then, we derive basic consumer and producer APIs. We then extend them with buffering, asynchronicity and error handling. Finally, we discuss some questions of resource control that arise when working with consumers and producers.

## Notation For Types

In order to define APIs, we need notation for expressing function signatures. We can get away with a fairly simple type system. We will use uppercase letters to denote type variables, e.g. "a stream emitting items of type `S`". Such a type can have any number or even infinitely many inhabiting values. We only require that all such values have finite size, i.e. a computer can represent them in memory.

A function that takes an argument of type `T` to an argument of type `U` is written as `T => U`. We assume all function values to be of finite size.

A pair of types `T` and `U` is written as `(T, U)`. A value of such a type consists of a value of type `T` and a value of type `U`. We also allow arbitrary tuples such as `(T, U, V)`. The empty tuple type `()` denotes absence of information, its sole inhabiting value is also written `()`.

Sum types are written `T + U`. Such a value is either a value of type `T` or a value of type `U`. The empty sum type is written as `0`. There is no value inhabiting the type `0`. As such, a sum `T + 0` is effectively equivalent (isomorphic) to just `T`, since the right alternative can never be taken.

With these building blocks, we can define purely functional interfaces. For example, the iterators producing finitely many items of type `T` can be described as a pair of a current `state` of type `S` and a `next` function of type `S => (T, S) + ()`. Calling `next` on a `state` value either returns `()` (the final value has been yielded), or it returns the current item (of type `T`) and a new state (of type `S`) to be queried later. An example of a concrete instance would use strings as the `state`, and the `next` function would return `()` when given the empty string, or it would return the argument's initial character as the yielded item and the remaining characters as the new state.

Such a purely functional approach is rather unusual in imperative programming languages. Rather than returning new state values, the `next` function would take a mutable reference to the state. We will write this as `&w S => T + ()`. Imperative manipulation of the state can be more efficient than the functional solution. The functional API however is more precise: When the iterator yields its last item, no new state is returned. In the imperative world, the types don't prevent the programmer from calling `next` on an iterator which has already finished. When reasoning about APIs, we will thus use the functional approach, and then convert the results to the equivalent imperative APIs.

A final note on these general prerequisites: We assume a basic, strictly evaluated programming language. Some languages provide features that would influence the APIs proposed in this text. The most common of these features are exceptions (which allow for more idiomatic error handling), lazy evaluation (which mitigates the need for indirection via function calls in order to handle infinite sequences), linear types (which allow to combine the efficiency of imperative APIs with the precision of functional APIs), and coroutines (which provide an alternative way of handling state). This text assumes that none of these features are available.

## Abstract Sequence Model

Before defining APIs for working with sequences, it makes sense to first define what precisely is meant by "sequence". We could define sequences as consisting of *finitely* many values of the same type (the above iterator API consumes such sequences), or *infinitely* many values of the same type (a state `S` and a function `S => (T, S)` would consume such a sequence), or perhaps allow both. Even more ambitiously, one could allow multiple item types: Intuitively, a sequence of finitelymany values of type `T` followed by finitely many values of type `U` should again be a sequence. Such composability might be useful when programming.

Fortunately, *finite* composable sequences are well-studied in theoretical computer science, they are exactly the [regular languages](https://en.wikipedia.org/wiki/Regular_language). We can easily define [regular expressions](https://en.wikipedia.org/wiki/Regular_expression) over types:

- The unit type `()` is a regular expression. The corresponding value is `()`.
- Any nonempty type `T` is a regular expression. The corresponding values are the values of type `T`.
- Let `R` and `S` be regular expressions whose values are the values of the types `T` and `U`:
  - `R + S` is a regular expression. The corresponding values are the values of the sum type `T + U` (note that these `+` operators are two distinct operators: One operates on regular expressions, the other operates on types).
  - `(R, S)` is a regular expression. The corresponding values are the values of the product type `(T, U)`.
  - `R*` is a regular expression. The corresponding values are the arrays containing values of type `T`.

The unit type is the neutral element of the product operator (concatenation). Interestingly enough, this definition excludes the empty type, even though it would be the neutral element of the `+` operator. Why is that? There is no straightforward corresponding principle in formal languages. Languages contain only finite strings, and we can interpret the neutral element of the `+` operator as introducing infinite sequences:

Consider regular expressions augmented by `0` as the neutral element of `+`. Now consider the expression `(T*, 0)`. As a type, it is not inhabited by any value. But things change if we view it as a sequence being consumed one item at a time. At each step, the next item could either be enough the value of type `T`, or a value inhabiting the empty type. Since no such value exists, the next item always has to be a `T` - the sequence extends to infinity.

Thus, regular expressions with `0` can serve as an abstract model for composable sequences of finitely or infinitely many values. There is however no straightforward translation to types, because infinite sequences cannot inhabit types (values are of finite size by definition). More practically speaking, infinitely large objects don't fit into memory of a computer. So instead, we need to work with the sequences lazily, one item at a time.

## Basic Consumer API

As already sketched above, a sequence can be *consumed* piece by piece. First off, it should be noted that laziness is only necessary when the `*` operator is involved. Sequences without this operator have a fixed number of items and can be represented as regular product types. In particular, a sequence of exactly one item of type `T` can be represented by simply the type `T`, and the `0` sequence is represented by the empty type. Sums of sequences translate directly into sum types.

The `*` operator is handled similarly to the above iterator example. But instead of returning `()` when the last item of type `T` has been emitted, we return a value of a generic type that represents the continuation of the sequence. A consumer producing many items of type `T` followed by another consumer of type `U` thus consists of a `state` of type `S`, and a `next` function of type `S => (S, T) + U`. A finite iterator (corresponding to the regular expression `T*`) uses `()` as `U`. An infinite iterator (corresponding to the regular expression `(T*, 0)`) uses the empty type as `U`. But consumers of more complex sequences can also be expressed, `U` could itself be a pair of a new `state` and a `next` function.

There is a one-to-one correspondence between such consumers and augmented regular expressions. Given one, we can construct the other that describes the same sequence. Assuming that augmented regular expressions provide a useful model of sequences, this gives us confidence in the choice of consumer API. While the API can and should be augmented for external effects such as asynchronicity, it is clearly minimal: removing any feature would break the one-to-one correspondence.

Despite the direct correspondence, augmented regular expressions and consumers are not identical. It is instructive to look at the differences. While the expressions form a very abstract description, the consumer API adds opinionated details. An obvious one is the indirection through function calls that introduces laziness and thus allows to work with infinite structures in a finite amount of memory. Also, it restricts the sequence to be accessed from left to right. One could just as well define a construction that accesses sequences from right to left.

There is another, less obvious detail that the API fixes in a principally arbitrary way. The caller of the `next` function has no control over which of the two  return alternatives it gets. The implementation of the function determines how many repetitions a `*` creates. In that sense, we can call this API a *passive consumer*.

It is also possible to define an *active consumer* which controls whether at a given state of type `S` the next item of a `*` operator should be yielded, or whether the sequence should progress to the next part of the augmented regular expression. The API would consist of a state of type `S` and two functions: `next` of type `S => (S, T)` and `continue` of type `S => U`. There is again a one-to-one correspondence between these types and augmented regular expressions. Thus, neither active nor passive consumers the better API, they are merely useful in different circumstances. It just so happens that passive consumers tend to pop up a lot more often in practice, so we will focus on them.

## Basic Producer API

Aside from consuming sequence item by item, another natural action is to *produce* it item by item. Again, the goal is to find a producer API with a one-to-one correspondence to augmented regular expressions. With such a correspondence, that would then also be a one-to-one correspondence with consumers. In fact, producing a sequence should be the exact opposite of consuming a sequence. It would be very elegant to automatically derive the produce API from the consumer API by applying some sort of "opposite" operation. This would make for the most consistent API design.

As a first attempt, we can take the `next` function and reverse the direction of the arrow (there's a category theory joke somewhere in there), or equivalently swap the argument and the return type. This yields the function signature `(S, T) + U => S`. What does this mean? First off, a function whose argument is a sum type can be split up into two functions taking one of the choices each as the argument (take a moment to think through why this is true). We thus get two functions: `push` of type `(S, T) => S` and `close` of type `U => S`.

The `push` function does exactly what we need. It takes a state and an item to insert into the sequence, and then returns the new state which allows us to insert at the next position. A specific example: `S` could be the type of strings, and `T` could be the type of characters. `push` would then take a string and a character, and return the string obtained by appending the character to the input string.

Unfortunately, the `close` function doesn't quite work. It traverses the sequence from right to left. One would require a state of type `U`, obtained from producing items on the right side of the sequence, before being able to obtain the state of type `S` to produce further to the left. Especially if we think about sequences as existing in time - the consumer API implies time progressing from left to right through the sequence - this mismatch in traversal direction is an issue, since the producer cannot progress back lots in time. This issue is easy to fix though, we can simply flip argument and return type of the `close` function: `S => U`.

Now, one can push items into the producer until one calls close, which then progresses to the next part of the sequence (which is merely `()` in case of the sequence `T*`). Note that this API defines *active* producers, the caller of the functions controls when to progress to the next part of the sequence.

While many libraries or programming languages provide synchronous consumers (iterators), few bother with synchronous producers. We consider this both a missed opportunity and a violation of the principle of least surprise, in particular if both consumers and producers are supported in the asynchronous case.

---

This concludes the first part of the text. We derived a basic consumer and producer API by first defining an abstract sequence model, and making sure to maintain symmetry between the two APIs and a one-to-one correspondence with the abstract model. While we like the evidence and expressivity of augmented regular expressions as a sequence model, other models might be fine as well. For example, a model that can only express sequences of the form `T*` or the form `T*0` can be fine and plenty of situations and leads to less complicated APIs. The important part in our opinion is to clearly define the model to guide the API design process, and to maintain symmetry between producers and consumers.

The APIs corresponding to augmented regular expressions:

```pseudocode
interface Consumer<S, T, U> {
  next: S => (S, T) + U
}

interface ImperativeConsumer<S, T, U> {
  next: &w S => T + U
}

interface Producer<S, T, U> {
  push: (S, T) => S,
  close: S => U
}

interface ImperativeProducer<S, T, U> {
  push: (&w S, T) => (),
  close: &w S => U
```

## Buffering

While these APIs are nice from a purely mathematical view, they still need to be extended to be more useful in the real world. One such extension is the introduction of *buffering*. Imagine a producer that appends a single byte on each push operation to a file on disk. Every such filesystem write is fairly expensive. It would be much more efficient to first buffer many bytes in memory and write them to disk all at once. More generally speaking, we want to allow producers to delay performing work by buffering items, occasionally *flushing* the buffered items and performing the work on a larger batch.

Producers can do most of this internally, calling `push` might sometimes buffer and sometimes trigger a flush - the caller of the function need not and cannot observe this. The implementation of `close` must also implicitly flush any buffered items before closing the producer. It must however also be possible for the calling code to manually trigger a flush to prevent items from sitting in the buffer too long. To that end, we extend the producer API with a `flush` function that takes a producer state and returns the corresponding flushed state: `S => S`.

What would a buffered consumer look like? Consider a consumer that emits arrays of bytes which it receives from a network connection. If it receives them one at a time and the `next` function is always called before the next byte arrives, it always emits arrays containing one byte each. A buffered consumer would be allowed to block on a `next` call and wait for more bytes to arrive. This would happen internally, the caller would be unable to detect such behavior.

There is an obvious problem though, is no more bytes arrive while the consumer is blocking, the buffered items are effectively lost. So we extend the API by adding a `force` function: `S => (S, T) + U`. While `next` allows buffering of items rather than directly returning them, `force` tells the consumer to immediately process and return all buffered items, or the next available item if none are currently buffered.

An unbuffered producer can be trivially turned into a buffered producer by using the identity function as the implementation of `flush`. An unbuffered consumer can be trivially turned into a buffered consumer by using the `next` function as the implementation of `force`. It is thus possible to only provide APIs for buffered consumers and producers. Alternatively, one could provide APIs for unbuffered ones, and then extend them to buffered versions by adding `flush` and `force` respectively. Either design is fine and preserves symmetry between consumers and producers. An API design however that would e.g. have buffered consumers but unbuffered producers by default would break symmetry and seems less desirable.

```pseudocode
interface Consumer<S, T, U> {
  next: S => (S, T) + U,
  force: S => (S, T) + U
}

interface ImperativeConsumer<S, T, U> {
  next: &w S => T + U,
  force: &w S => T + U
}

interface Producer<S, T, U> {
  push: (S, T) => S,
  flush: S => S,
  close: S => U
}

interface ImperativeProducer<S, T, U> {
  push: (&w S, T) => (),
  flush: &w S => (),
  close: &w S => U
```

## Effects

The above APIs are quite useful for interacting with in-memory data structures. They are however insufficient to model effectful interactions with the real world. For example, a producer for writing bytes over a TCP connection needs to be able to deal with the fact that the connection might fail at any time (error handling), or that writing might take a long time due to backpressure (asynchronicity). This section shows how to add such effects to the APIs in a principled manner.

### Error Handling

Error handling is necessary when function calls are fallible, i.e. when the caller has no control over whether a regular result is returned or an error. Such functions can be modeled by having them return `RegularReturnType + ErrorType`. For our APIs, we add the type variable `E` as the type of errors.

We will only consider unrecoverable errors. For consumers, recoverable errors can be modeled by making the item type a sum type of the actual item type and the error type. And for producers, unrecoverable errors seem to arise much more often in practice than recoverable ones.

When looking at consumer APIs, it is unclear whether it is actually necessary to add error handling at all. Changing the type of `next` to `S => ((S, T) + U) + E` gives an API where error are unrecoverable. But this is not more expressive than simply choosing `U` as `ActualU + E`. To get a clearer picture, we can look at producers first and then adjust the consumer API to maintain symmetry.

On the producer side, things are more clear-cut. While the `close` function signature `S => U + E` could also be emulated by choosing `U` as `ActualU + E`, this is not possible for `push`. `(S, T) => S + E` is strictly more expressive than merely `(S, T) => S`. To maintain consistency, `close` should also handle the error explicitly, as should `next`.

Two short notes on these APIs: First, infallible consumers and producers can still be modeled by using the empty type as `E`. Second, the imperative versions of these APIs do not express in the types whether errors are recoverable or unrecoverable. When defining APIs for imperative language, the documentation should thus clearly state which kind of errors it deals with.

While these APIs are quite useful, the process of deriving them felt very ad hoc. They also break the idea of being able to convert between producer and consumer API by merely switching the direction of the function arrow (and adjusting the direction in which the sequence is processed). But there is an interpretation which restores this property: We can think of the functions themselves being fallible.

A fallible function of error type `E` from argument type `A` to return type `B` is simply a regular function of type `A => B + E`. Converting all functions of our APIs to fallible functions trivially yields the APIs for producers and consumers with unrecoverable errors. Additionally, some programming languages have built-in support for fallible functions: exceptions. In a language with exceptions it would be more idiomatic to use those rather than sum types to signal errors.

### Asynchronicity

Just like we can generalize functions to fallible functions, we can also generalize them to asynchronous functions, written as `A ~> B`. Again, that might be different ways of expressing these in a particular programming language. Some languages might have built in support for asynchronous functions, others might use callbacks, another common choice of values that represent that the result will be available at a later point (often called *futures* or *promises*). Obtaining asynchronous producer and consumer APIs thus becomes trivial, one merely needs to change all functions to asynchronous functions.

Many languages only provide very basic synchronous consumer APIs (iterators) and often no producer APIs at all. Designers of asynchronous APIs for processing sequences are thus often not only dealing with asynchronicity, but also with defining an appropriately expressive sequence model, reinventing the concept of a producer, error handling, buffering etc. This usually makes the issue of asynchronous sequence abstractions look much more daunting than it needs to be. Asynchronous sequence abstractions that differ from their synchronous counterparts in more than merely the introduction of asynchronicity could be considered flawed designs - and quite often the issue lies with the overly simplistic synchronous APIs.

When combining asynchronicity and error handling, the order of the effects matters (`Future<T + E>` as opposed to `Future<T> + E`). This usually means that exceptions cannot be used for error handling in asynchronous APIs - unless of course the language explicitly supports asynchronous exceptions.

```pseudocode
interface Consumer<S, T, E, U> {
  next: S ~> ((S, T) + U) + E,
  force: S ~> ((S, T) + U) + E
}

interface ImperativeConsumer<S, T, E, U> {
  next: &w S ~> (T + U) + E,
  force: &w S ~> (T + U) + E
}

interface Producer<S, T, E, U> {
  push: (S, T) ~> S + E,
  flush: S ~> S + E,
  close: S ~> U + E
}

interface ImperativeProducer<S, T, E, U> {
  push: (&w S, T) ~> E,
  flush: &w S ~> E,
  close: &w S ~> U + E
```

---

The notion of effectful functions can be used to introduce other effects as well, error handling and asynchronicity merely happen to be the most common ones. In languages with higher-kinded types, the APIs can abstract over effects, but we will not further consider this option in this text. Those readers who enjoy their type system shenanigans can think about other useful effects, what kind of type classes effects need to implement in order to be able to implement useful type classes for the sequence abstractions as well, or perhaps why some effects (for example non-determinism) don't seem to lend themselves towards imperative APIs.

## Resource Ownership



## Conclusion

## Acknowledgments

Many thanks to Erick Lavoie and Andreas Dzialocha for listening to my ramblings, asking clever questions and providing valuable feedback.

## Bonus Problem

Using similar methodology, define principled APIs for working with trees.
