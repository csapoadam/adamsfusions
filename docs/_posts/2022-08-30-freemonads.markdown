---
layout: post
title: The Free Monad - A Versatile Tool for Mocking Side-Effects and Dynamically Reinterpreting Abstract Code - WIP
date: 2022-08-30 12:20:23 +0900
category: SoftwareDesign
---

In these two previous posts, we went through a number of functional programming concepts, namely functors, monads, contravariant functors and contravariant monads. We saw based on some rudimentary examples that these concepts could be used to write code that is more readable and more elegant in managing both dependencies and side effects. The basic mechanisms by which all of this could be achieved can be summarized as follows:

* Encapsulation of the glue between functions: the wrapping and unwrapping of data structures, and the passing on of data from one function to the next could be encapsulated under the hood by canonical functions such as *map()* and *flatmap()*;
* Lazy (deferred) evaluation: this means that a function, or a composed chain of functions could be stored as data, wrapped in a container, allowing for both the input to and the output from the function(s) to be gradually transformed prior to running the code

Today, we will go one step forward and gently introduce a functional data structure called the **free monad**. The wonder of free monads is that they can be used to model any sequence of operations in such a way that *not only the execution, but also the interpretation of those operations can be deferred to a later time*. Free monads can be used to achieve this by encapsulating both an operation as data and a sequence of transformations on that operation in a canonical way.

# The why and what of free monads

Apparently, many branches of mathematics use the term 'free' to denote a concept that is in some sense the least restricted example in its class. We don't need to go into the details to understand free monads, but maybe this interpretation will help reinforce the idea that free monads are a very minimal, canonical and therefore highly versatile type of monad.

Briefly stated, a **free monad is a composite data type whose instances can either be of type** *Pure* **, or of type** *Suspend*.

The **Pure** variant of a free monad basically holds a value; usually, the result of some computations. It represents the *base case*, or *fixed point* of the free monad (think of the base case of recursion, as in our earlier Trampoline example).

The **Suspend** (also known as **Impure**) variant of a free monad encapsulates some value, together with a deferred operation on that value.

Based on this short description, we can already see that free monads hold the potential to represent more than one kind of data processing pattern. As suggested by the trampoline example, we can implement stack-safe recursive behaviors by having a function that keeps returning a *Suspend* object, as long as further recursion is necessary; and a *Pure* object, once the recursion terminates. If the value part of a *Suspend* object represents a reified operation (as opposed to some literal data in the classical sense), we can reinterpret the semantics of that operation at a later time. In this way, we can create domain-specific languages (DSLs) with multiple alternative interpreters - an approach that can be useful for mocking / testing some behavior without actually running it to get unwanted side effects.

Before going into the examples, let's take a look at a possible implementation of free monads in Javascript:

{% highlight javascript %}
const Pure = x => ({
  type: () => 'free.pure',
  map: f => Pure(f(x)),
  flatMap: g => g(x),
  toString: () => `Pure(${x})`,
  run: () => x
})

const Suspend = (x, f) => ({
  type: () => 'free.suspend',
  map: g => Suspend(x, input => f(input).map(result => g(result))),
  flatMap: g => Suspend(x, input => f(input).flatMap(g)),
  toString: () => `Suspend(${JSON.stringify(x)}, f), where f(${x}) = ${f(x)}`,
  run: () => f(x),
  getX: () => x
})
{% endhighlight %}

First, let's consider *Pure*. Since in native Javascript at least, we don't have the option of creating algebraic data types, we can implement a *type()* function to be able to query the variant we are working with. The canonical functions, *map()* and *flatmap()* operate exactly as one would expect. In the case of *flatmap()*, the flatmapped function g will take care of wrapping the result into a new free monad, so we don't need to do this explictly as in the case of *map()*. *toString()* will be useful for printing the Pure object to the console, and *run()* will be necessary if we want to get back the value *x* that is stored inside the Pure object.

The case of *Suspend* is less trivial. Remember, in this case, we are working not only with some data *x* (whatever that may be), but also with some transformations we already want to carry out (represented by *f*). So, if we wanted to *run()* the Suspend object, we would have to call *f(x)*. If we wanted to map or flatmap a second function, *g*, onto the Suspend object, the way to go would be to return a new Suspend object that still encapsulates the same data, along with a new transformation that somehow represents a composition of the original transformation and the second transformation, *g*. But recall that in the case of *map()*, we do not expect *g* to wrap its result into the same structure that is returned by the original *f*, therefore, we need to map *g* onto *f(x)*. By contrast, in the case of *flatMap()*, we can flatmap *g* onto the output of *f* and be sure that we will obtain an analogous type.

Note that the above implementation entails that function *f* itself will have to return a monad that has both a *map()* and a *flatMap()* operation, otherwise the chain of computations will break down. To make sure that we can always do this, let's create a basic monadic wrapper that can encapsulate any basic value as follows:

{% highlight javascript %}
const Id = x =>
({
  map:f => Id(f(x)),
  flatMap:f => f(x),
  extract: () => x,
  toString: () => `Id(${x})`
})
{% endhighlight %}

Finally, before moving on to some more interesting examples, let's put all of this to the test:

{% highlight javascript %}
let computation = Suspend(5, x => Id(x+1)) // A Suspend that holds value 5 and a function f that increments it to Id(6)
  .flatMap(x => Id(x*2)) // A new Suspend that holds value 5 and a function that doubles the result -> Id(12)
  .map(result => {
    console.log(`\t\tresult of computation 2(x+1) = ${result} (where x = 5)`)
    return result
  }) // a new Suspend that holds 5 and a function that prints something and once mapped will still result in Id(12)
  .flatMap(x => Id(x+1)); // a new Suspend that holds a 5 and a function f that if run, returns Id(13)

console.log('\nAbout to actually run!')
console.log(`finished... ${computation.run().extract()}`) //indeed, we get 13, but 'result of computation ... = 12' is also printed
{% endhighlight %}

## Simulating other monads using the free monad

Since we mentioned that free monads are canonical, let's take a look at an example where we implement another monad using free monads. For this example, we consider the **Maybe monad**, which is often used to represent the results of a computation that can be either successful or fail with no result. Accordingly, an instance of the Maybe monad can be *Just(x)* (where x is the result of a successful operation), or *Nothing()*, which is a primitive representing an unsuccessful computation.

To map the Maybe monad onto the Free monad, so to speak, we first need to *lift* these two objects into the context of Pure and Suspend. Let's take a look at how we might do this:

{% highlight javascript %}
// Simulating other monads with Pure and Suspend...
const just = x => ({x, type: () => 'maybe.just'})
const nothing = () => ({type: () => 'maybe.nothing'})

const liftF = operationObject => Suspend(operationObject, Pure)

// when run, this function creates a Suspend w/ data {x, type: 'maybe.just'}
// and a promise to eventually wrap it into a Pure
const Just = x => liftF(just(x))
// when run, this functioncreates a Suspend w/ data {type: 'maybe.nothing'}
// and a promise to eventually wrap it into a Pure
const Nothing = () => liftF(nothing())
{% endhighlight %}

In our application code, we will (mostly) use the capitalized versions *Just* and *Nothing*. These are both functions that either take a value (*x* in the case of *Just*) or nothing (in the case of *Nothing*) and *lift them into the context of a Suspend object*. This Suspend object promises to wrap the data that represents a *just* or *nothing* object into a *Pure* object at some later time (when the Suspend is fully finalized and run). The data itself that represents the *just* or *nothing* objects either has the form *{x, type: () => 'maybe.just'}* or *{type: () => 'maybe.nothing'}*.

Since a *Just(x)* or a *Nothing()* is now really just a *Suspend* object under the hood, in theory we can now map or flatmap other functions onto them. As expected, any function that we flatmap onto them will have to return either a *Just(x)* or a *Nothing()*. Any function that we flatmap onto them, in turn, will as a general rule have to return a lowercase *just(x)* or a lowercase *nothing()*. That's because, under the hood, the Suspend objects we are operating with actually operate on data of this form (either *{x, type: () => 'maybe.just'}* or *{type: () => 'maybe.nothing'})*.

The big question is, how can we actually run a chain of computations built up in this way? To do this, we need an *interpreter* function, which we will call *runMaybe()* that

{% highlight javascript %}
const runMaybe = justOrNothing => {
  if (justOrNothing.type() === 'free.pure') {
    return justOrNothing.run()
  } else if (justOrNothing.type() === 'free.suspend') {
    if (justOrNothing.getX().type() === 'maybe.nothing') {
      return Nothing()
    }
    return runMaybe(justOrNothing.run())
  }
}

const maybeExample1 = Just(4)
  .map(value => just(value.x + 1))
  .flatMap(input => Just(2*input.x))
console.log(`result of maybeExample1 is: ${JSON.stringify(runMaybe(maybeExample1))}\n`)

const maybeExample2 = Just(4)
  .flatMap(input => Just(2*input.x))
  .flatMap(input => Nothing())
  .flatMap(input => Just(input + 1))
console.log(`result of maybeExample2 is: ${JSON.stringify(runMaybe(maybeExample2))}\n`)
{% endhighlight %}

...