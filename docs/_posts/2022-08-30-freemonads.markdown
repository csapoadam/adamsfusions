---
layout: post
title: The Free Monad - A Versatile Tool for Creating Test Doubles
date: 2022-08-30 12:20:23 +0900
category: SoftwareDesign
---

As we know from the vast literature on software testing, there are many practical approaches for substituting as yet not fully functional software components with so-called **test doubles** -- including [stubs, fakes, and mocks](https://blog.pragmatists.com/test-doubles-fakes-mocks-and-stubs-1a7491dfa3da). In this post, we will take a look at a functional programming structure called the **Free Monad**, and consider how it offers an elegant method for creating test doubles.


In these two previous posts, we went through a number of functional programming concepts, namely functors, monads, contravariant functors and contravariant monads. We saw based on some rudimentary examples that these concepts could be used to write code that is more readable and more elegant in managing both dependencies and side effects. The basic mechanisms by which all of this could be achieved can be summarized as follows:

* Encapsulation of the glue between functions: the wrapping and unwrapping of data structures, and the passing on of data from one function to the next could be encapsulated under the hood by canonical functions such as *map()* and *flatmap()*;
* Lazy (deferred) evaluation: this means that a function, or a composed chain of functions could be stored as data, wrapped in a container, allowing for both the input to and the output from the function(s) to be gradually transformed prior to running the code

Today, we will go one step forward and gently introduce a functional data structure called the **free monad**. The wonder of free monads is that they can be used to model any sequence of operations in such a way that *not only the execution, but also the interpretation of those operations can be deferred to a later time*. Free monads can be used to achieve this by encapsulating both an operation as data and a sequence of transformations on that operation in a canonical way.

# The why and what of free monads

Apparently, many branches of mathematics use the term 'free' to denote a concept that is in some sense the least restricted example in its class. We don't need to go into the details to understand free monads, but maybe this interpretation will help reinforce the idea that free monads are a very minimal, canonical and therefore highly versatile type of monad.

Briefly stated, a **free monad is a composite data type whose instances can either be of type** *Suspend* **, or of type** *Pure*.

The **Suspend** (also known as **Impure**) variant of a free monad encapsulates some value, together with a deferred operation on that value.

The **Pure** variant of a free monad, in turn, simply holds a value; usually, the starting point or final results of some computations.

Free monads hold the potential to represent more than one kind of data processing pattern. As suggested by the trampoline example, we can implement stack-safe recursive behaviors by having a function that keeps returning a *Suspend* object, as long as further recursion is necessary; and a *Pure* object, once the recursion terminates. If, on the other hand, the value part of a *Suspend* object represents a reified operation (as opposed to some literal data in the classical sense), we can reinterpret the semantics of that operation at a later time. In this way, we can create domain-specific languages (DSLs) with multiple alternative interpreters - an approach that can be useful for mocking / testing some behavior without actually running it to get unwanted side effects.

Before going into specific examples, let's take a look at a possible implementation of free monads in Javascript:

{% highlight javascript %}
const Pure = x => ({
  type: () => 'free.pure',
  map: f => Pure(f(x)),
  flatMap: g => g(x),
  toString: () => `Pure(${JSON.stringify(x)})`,
  run: () => x
})

const Suspend = (x, f) => ({
  type: () => 'free.suspend',
  map: g => Suspend(x, input => f(input).map(g)),
  flatMap: g => Suspend(x, input => f(input).flatMap(g)),
  toString: () => `Suspend(${JSON.stringify(x)}, f),\n  where f(${JSON.stringify(x)}) = ${f(x)}`,
  run: () => f(x),
  getX: () => x
})

{% endhighlight %}

First, let's consider *Pure*. Since in native Javascript at least, we don't have the option of creating algebraic data types, we can implement a *type()* function to be able to query the variant we are working with. The canonical functions, *map()* and *flatMap()* operate exactly as one would expect. In the case of *flatMap()*, the flatmapped function g will take care of wrapping the result into a new free monad, so we don't need to do this explictly as in the case of *map()*. *toString()* will be useful for printing the Pure object to the console, and *run()* will be necessary if we want to get back the value *x* that is stored inside the Pure object.

The case of *Suspend* is less trivial. Remember, in this case, we are working not only with some data *x* (whatever that may be), but also with some transformations we already want to carry out on *x* (represented by *f*). So, if we wanted to *run()* the Suspend object, we would have to call *f(x)*. If we wanted to map or flatmap a second function, *g*, onto the Suspend object, the way to go would be to return a new Suspend object that still encapsulates the same data, along with a new transformation that somehow represents a composition of the original transformation and the second transformation, *g*. But recall that in the case of *map()*, we do not expect *g* to wrap its result into the same structure that is returned by the original *f*, therefore, we need to map *g* onto *f(x)*. By contrast, in the case of *flatMap()*, we can flatmap *g* onto the output of *f* and be sure that we will obtain an analogous type.

Of course, in both cases, we would have to make sure that *f(x)* will return a type that has a *map(g)* or *flatmap(g)* function, respectively, and that the type of *g* conforms to either of those functions; othrewise the chain of computations would break down. To help guarantee this, for the purposes of our later examples, let's create a basic monadic wrapper that can encapsulate any basic value as follows:

{% highlight javascript %}
const Id = x =>
({
  map:f => Id(f(x)),
  flatMap:f => f(x),
  extract: () => x,
  toString: () => `Id(${x})`
})
{% endhighlight %}

In other words, if both *f* and *g* return values wrapped in an *Id* type, we can be sure that the *flatMap* function will always exist and that the computations won't break down.

Before moving on to some more interesting examples, let's put all of this to the test:

{% highlight javascript %}
let computation = Suspend(5, x => Id(x+1))  // A Suspend object that holds a value 5 and a function f that increments it to Id(6)
  .flatMap(x => Id(x*2))                    // A new Suspend object that holds a value 5 and a function that first adds 1 to it, then doubles the result -> Id(12)
  .map(result => {
    console.log(`\t\tresult of computation 2(x+1) = ${result} (where x = 5)`)
    return result
  })                                        // a new Suspend object that holds a value 5 and a function that prints something and once mapped will still result in Id(12)
  .flatMap(x => Id(x+1));                   // a new Suspend object that holds a value 5 and a function f that if run, will return Id(13)

console.log('\nAbout to actually run!')
console.log(`finished... ${computation.run().extract()}`) //indeed, we get 13, but 'result of computation ... = 12' is also printed
{% endhighlight %}

## Simulating other monads using the free monad

Since we mentioned that free monads are canonical, let's take a look at an example where we implement another monad using free monads. For this example, we consider the **Maybe monad**, which is often used to represent the results of a computation that can be either successful or fail with no result. Accordingly, an instance of the Maybe monad can be *Just(x)* (where x is the result of a successful operation), or *Nothing()*, which is a primitive representing an unsuccessful computation.

To map the Maybe monad onto the Free monad, so to speak, we first need to *lift* these two objects into the context of Pure and Suspend. Let's take a look at how we might do this:

{% highlight javascript %}
// Simulating other monads with Pure and Suspend...
// First, we define lowercase *just* and *nothing* as a reified representation of the Maybe monad
const just = x => ({x, type: () => 'maybe.just', toString: () => `just(${x})`})
const nothing = () => ({type: () => 'maybe.nothing', toString: () => `nothing()`})

const liftF = operationObject => Suspend(operationObject, Pure)

// when run, the uppercase Just function creates a Suspend w/ data {x, type: 'maybe.just', toString: () => `just(${x})`}}
// and a promise to eventually wrap it into a Pure
const Just = x => liftF(just(x))
// when run, this function creates a Suspend w/ data {type: 'maybe.nothing', toString: () => `nothing()`}
// and a promise to eventually wrap it into a Pure
const Nothing = () => liftF(nothing())
{% endhighlight %}

In our application code, we will (mostly) use the capitalized versions *Just* and *Nothing*. These are both functions that either take a value (*x* in the case of *Just*) or nothing (in the case of *Nothing*) and *lift them into the context of a Suspend object*. This Suspend object promises to wrap the data that represents a lowercase *just* or *nothing* object into a *Pure* object at some later time (when the Suspend is fully finalized and run). Leaving out the *toString* functions, the data itself that represents the *just(x)* or *nothing()* objects either has the form *{x, type: () => 'maybe.just'}* or *{type: () => 'maybe.nothing'}*.

Since a *Just(x)* or a *Nothing()* is now really just a *Suspend* object under the hood, in theory we can now map or flatmap other functions onto them. The benefit of storing a function *f* inside the Suspend object in each case that will return a *Pure* object is that if we flatmap a new function, *g* onto the *Suspend* object, the new *Suspend* object that we obtain will still either contain the same (lowercase) *just(x)* or *nothing()*, but the function it promises to run on it will be *Pure(...).flatmap(g)* (where the three dots are either the *just(x)* or the *nothing()*), which just translates to *g(just(x))* or *g(nothing())*.

If, in turn, this function *g* returns either a *Just(x)* or a *Nothing()*, we can repeat this same process again and again.

The big question is, how can we actually run a chain of computations built up in this way? You might think that you could always just call *run()* on it, as usual. But apart from the most basic cases, this will not work! Consider the following examples:

{% highlight javascript %}
const maybe1 = Just(4)
console.log(`maybe1 is ${maybe1}.\nResult of run(): ${maybe1.run()}`)
// maybe1 is: Suspend({"x":4}, f), where f({"x":4}) = Pure({"x":4})
// result of run(): Pure({"x":4})

const maybe2 = Just(4).flatMap(res => Just(res + 1))
console.log(`maybe2 is ${maybe2}.\nResult of run(): ${maybe2.run()}`)
// maybe2 is: Suspend({"x":4}, f), where
//   f({"x":4}) = Suspend({"x":"just(4)1"}, f), where
//   f({"x":"just(4)1"}) = Pure({"x":"just(4)1"})
// result of run(): Suspend({"x":"just(4)1"}, f), where
//   f({"x":"just(4)1"}) = Pure({"x":"just(4)1"})
{% endhighlight %}

In the first case, we really just have a *Suspend* object on our hands that, once run, will return the stored data wrapped inside a *Pure*. But something seems amiss in the second case. Recall that we said that flatmapping a function *g* on a *Just(x)* object will just run *Pure(just(x)).flatMap(g)*, which is the same as *g(just(x))*. So, in our case, when calling *maybe2.run()*, since *g* is *res => Just(res + 1)*, what we get back is a *Just(just(x) + 1)*. As we've said before, this in turn is just a *Suspend* object -- now containing the datum *just(x) + 1* (or in other words, *just(x)1*, since the Javascript interpreter will just treat both operands of the + operator as strings), and promising to wrap it into a *Pure* object!

What can we glean from this analysis?

* First, since it's not possible to add together a *just(x)* and a number, clearly, one would have to check whether the input to the flatmapped function, *g*, is a *just(x)* or a *nothing()* and unwrap that object, like this: *res => Just(res.x + 1)* instead of *res => Just(res + 1)*. This would take care of the issue with the 1 being appended as a string rather than added to the internal 4 as a number.

* But the more important point is this: When composing Suspend objects using either *map()* or *flatmap()*, we are basically just transforming the original one to represent more elaborate computations on the inside. But what those computations actually do is a different question, which we will find out when we actually run them. If at any point, such a computation returns as an intermediate result a Suspend object (i.e., a *Just(x)* or a *Nothing*), this will halt the computations until this new Suspend is actually run. In fact, all of the subsequent computations will just have the effect of transforming that new Suspend object, to encapsulate all of the remaining computations which we wanted to carry out but haven't yet gotten to!

So we have two problems with running a Suspend object composed in this way. First, we don't know how many times to run it. Second, the internal data that it contains still may need to be processed in unique ways. In our case, we still need to check whether the result from the earlier computation is a *nothing()* or a *just(x)* with a value! In the former case, we would have to return a *Nothing()*, not a *Just(x)*. And in the latter case, we would still need to unpack the value inside it.

Instead of taking care of the type check in every flatmapped function, we can use a special *runner* function, designed specifically for the Maybe monad (in this case). Note that we cannot implement this runner function as part of the *Pure()* and *Suspend()* functions, since in the Suspend case (see below) we cannot know in advance what data structure we are unpacking (e.g., a *Nothing()* or a *Just(x)*):

{% highlight javascript %}
const runMaybe = capitalJorN => {
  let current = capitalJorN
  while(true) {
    if (current.type() === 'free.pure') {
      return current.run() // for Pure, this just returns x
    }
    // if it is a Suspend, it has getX()
    if (current.getX().type() === 'maybe.nothing') {
      return current.getX() // we cannot do anything further on a nothing
    }
    // we know we have a Suspend that we can run to either get a Pure or a Suspend
    current = current.run()
  }
}
{% endhighlight %}

The good news is that eventually our loop of constantly getting a new *Just(x)* (or *Nothing()*) back will terminate, and the final *Suspend*, when run, will return a *Pure* object. So we can keep calling *runMaybe()* recursively (or in this case, terminate the infinite loop). 

Now let's take a look at some examples:

{% highlight javascript %}
const maybeExample1 = Just(4)
  .map(value => just(value.x + 1))
  .flatMap(input => Just(2*input.x))
console.log(`result of maybeExample1 is: ${runMaybe(maybeExample1)}\n`) // just(10)

const maybeExample2 = Just(4)
  .flatMap(input => Just(2*input.x))
  .flatMap(input => Nothing())
  .flatMap(input => Just(input.x + 1))
console.log(`result of maybeExample2 is: ${runMaybe(maybeExample2)}\n`) // nothing()
{% endhighlight %}

Note that in the second case, the fact that the penultimate computation returned a *Nothing()* was handled seamlessly by our *runMaybe()* function.

## Re-interpreting abstract syntax trees using the free monad

As we've seen in the previous example, in principle it's possible to implement any kind of monad using the free monad, by:

* reifying the monad as a set of data structures,
* piggybacking the data structures on the data field of a Suspend object,
* such that the operation inside the Suspend object promises to wrap the data field into a *Pure*

We've also seen that when doing this, a special runner function is also needed to make sure that the reified data structures are recursively (and in each iteration, suitably!) processed when the operations (i.e., mapping and flatmapping of functions) return a new Free monad. By necessity this runner function will be unique to the piggybacked monad since the number of parameters and their semantics in the reified monad will vary on a case-by-case basis.

As a variation on this pattern, prior to actually running the operations in the Suspend object(s), we could also transform the data contained in them into something else. This is useful when this data represents operations that can be carried out as opposed to just data (as in the Just(x) and Nothing() case), assuming that we want to interpret the semantics of those operations on the fly.

Let's consider a simple algebraic example with two different interpretations: one literal, and one that simply prints the operations as a string.

To do this, we first define the (lowercase) reified operations and their uppercase versions that are lifted into the realm of the free monad:

{% highlight javascript %}
const div = (x, y) => ({x, y, type: 'div', toString: () => `div(${x}, ${y})`}) // a function that returns a data structure representing division
const add = (x, y) => ({x, y, type: 'add', toString: () => `add(${x}, ${y})`}) // a function that returns a data structure representing addition
const mul = (x, y) => ({x, y, type: 'mul', toString: () => `mul(${x}, ${y})`}) // a function that returns a data structure representing multiplication

const Div = (x, y) => liftF(div(x, y)) //a function that if run, returns a Suspend(div(x,y), f) such that f() returns Pure(div(x,y))
const Add = (x, y) => liftF(add(x, y)) //a function that if run, returns a Suspend(add(x,y), f) such that f() returns Pure(add(x,y))
const Mul = (x, y) => liftF(mul(x, y)) //a function that if run, returns a Suspend(mul(x,y), f) such that f() returns Pure(mul(x,y))
{% endhighlight %}

Here, again, *Div(x, y)*, *Add(x, y)* and *Mul(x, y)* are actually just *Suspend* objects under the hood that contain the objects representing the lowercase variants *div*, *add* and *mul*, with the deferred operation of wrapping those into a *Pure*. However, what is different compared to the example with the Maybe monad is that *div*, *add* and *mul* now represent operations that can be carried out, as opposed to being plain numbers (as in the case of, e.g., *just(4)*).

Next, we define our two *interpreter* functions, whose task it is to transform the data contained within each intermediate *Suspend* object prior to their being run. In other words, when we have our first *Suspend* object -- which promises to wrap the reified operation into a *Pure* object -- and then flatmap our next function, *g* onto it, instead of computing, e.g. something like:

{% highlight javascript %}
Pure(div(6, 2)).flatMap(x => Add(x, 2)) = Add(div(6, 2), 2)
{% endhighlight %}

we want to compute:

{% highlight javascript %}
Pure(interpreter(div(6, 2))).flatMap(x => Add(x, 2)) = Add(interpreter(div(6, 2)), 2)
{% endhighlight %}

... and repeat this process until we finish. Here, however, it's important that the interpreter function itself return a common data structure (ideally, the *Id()* monad), rather than just a plain value. Why? Because, in general we can have as many interpreter functions as there are stars in the sky, and we need to make sure that the values we obtain can be handled in a uniform way. In addition, using a monad like *Id()* can be useful in that we can be sure that the *map()* and *flatMap* functions will be defined on the result, so we can continue to string together operations at a higher level.

Our two interpreter functions, then, might look like this:
 
{% highlight javascript %}
const interpretAsValue = x => {
  // returns an Id monad containing a numerical value
  if (x.type == 'div') {
    return Id(x.x / x.y)
  }
  if (x.type == 'mul') {
    return Id(x.x * x.y)
  }
  if (x.type == 'add') {
    return Id(x.x + x.y)
  }
}

const interpretAsString = x => {
  // returns an Id monad containing a string representation
  if (x.type == 'div') {
    return Id(`(${x.x} / ${x.y})`)
  }
  if (x.type == 'mul') {
    return Id(`(${x.x} * ${x.y})`)
  }
  if (x.type == 'add') {
    return Id(`(${x.x} + ${x.y})`)
  }
}
{% endhighlight %}

Regardless of which interpreter function we will be working with, we can be sure that when transforming the value inside any given Free monad, we will get an Id() monad with something inside it.

Our application code, in turn, would still look something like this:

{% highlight javascript %}
const computationX = Add(5,1) //Suspend(add(5,1), f), such that f(add(5,1)) = Pure(add(5,1))
  .flatMap(x => Mul(x, 2))
  .flatMap(x => Div(x, 6))

console.log(`computationX: ${computationX}`)
//  Suspend(add(5, 1), f), where
//    f(add(5, 1)) = Suspend(mul(add(5, 1), 2), f), where
//    f(mul(add(5, 1), 2)) = Suspend(div(mul(add(5, 1), 2), 6), f), where
//    f(div(mul(add(5, 1), 2), 6)) = Pure(div(mul(add(5, 1), 2), 6))
{% endhighlight %}

The question now is, how can we adapt our earlier runner function to carry out the transformations while performing the computations. One naive solution might look like this:


{% highlight javascript %}
const runnerWithInterpreter = (piggybackedMd, interpreter) => {
  let current = piggybackedMd
  while (true) {
    if (current.type() == 'free.pure') {
      return current.run()
    } else {
      current = interpreter(current.getX()).flatMap(current.getF())
    }
  }
}

console.log(runnerWithInterpreter(computationX, interpretAsString)) // (((5 + 1) * 2) / 6)
console.log(runnerWithInterpreter(computationX, interpretAsValue)) // 2
{% endhighlight %}

This will work, assuming that we augment the definition of *Suspend*, to include a *getF()* function, which predictably will return the function itself stored in the Suspend object.Notice that since the interpreter function function returns a monad, we can now flatmap that function onto the result.

On the other hand, notice that in the case of this runner function, at no point did we invoke anything specific to the *Mul*, *Div* and *Add* objects. Based on this, we could generalize the solution and implement it as part of the *Suspend* and *Pure* data types themselves! Formally, the function we need to define is called *foldMap*, since we want to combine all of the operations (fold) into a single result, while transforming the intermediate results (map).

To ensure that our *foldMap* function has the same number and type of arguments in both the *Pure* and *Suspend* case, *foldMap()* will accept two arguments: first, the interpreter function itself, and second, the wrapper type that the interpreter function uses to wrap its result (the latter is necessary because in the case of *Pure*, we might still want to wrap the result into a new monad).

Thus, in the case of *Pure*, all we have to do is wrap the contents of the *Pure* object into the wrapper type:

{% highlight javascript %}
const Pure = x => ({
  type: () => 'free.pure',
  map: f => Pure(f(x)),
  flatMap: g => g(x),
  toString: () => `Pure(${JSON.stringify(x)})`,
  run: () => x,
  foldMap: (interpreter, constructor) => constructor(x)
})
{% endhighlight %}

In the case of *Suspend*, our task is a tiny bit less trivial. First, we want to use the interpreter to transform the datum inside the object. Next, we will call the function *f* on the result, to obtain our next *Suspend* or *Pure* object in the chain, and call foldMap() on that new free monad as well:

{% highlight javascript %}
const Suspend = (x, f) => ({
  type: () => 'free.suspend',
  map: g => Suspend(x, input => f(input).map(g)),
  flatMap: g => Suspend(x, input => f(input).flatMap(g)),
  toString: () => `Suspend(${JSON.stringify(x)}, f), where f(${JSON.stringify(x)}) = ${f(x)}`,
  run: () => f(x),
  getX: () => x,
  foldMap: (interpreter, constructor) => interpreter(x).flatMap(result =>
    f(result).foldMap(interpreter, constructor))
{% endhighlight %}

Note that in this case, it's useful that the interpreter function will return an *Id()*, or similar monad, since in this case we can elegantly *flatMap* the original function *f* on it, and -- under the hood -- this function will then be called on the wrapped data, in just the same way as was the case with *Pure*.

Now, we have:

{% highlight javascript %}
console.log(`string interpretation: ${computationX.foldMap(interpretAsString, Id).extract()}`)
// string interpretation: (((5 + 1) * 2) / 6)
console.log(`value: ${computationX.foldMap(interpretAsValue, Id).extract()}`)
// value: 2
{% endhighlight %}

## An example of the free monad used for mocking / testing applications

The arithmetics example in the previous section was edifying, but perhaps not very interesting from a practical perspective. So let's imagine we are working on a drone and want to mock various operations on it without actually carrying them out (lest we make it self-destruct before even deploying it!)

So let's assume we have some functions to manage our drone like *launchDrone()*, *sendSamplesDrone()* and *selfDestructDestruct()*. However, instead of calling them directly, we can reify them as data structures, and then lift them into the realm of the Free Monad:


{% highlight javascript %}
const launch = (drone) => ({drone, type: 'launch', toString: () => `launch(${drone})`})
const send_samples = (drone, data, addr) => ({drone, data, addr, type: 'send_samples', toString: () => `send_samples(${drone}, ${data}, ${addr})`})
const self_destruct = (drone) => ({drone, type: 'self_destruct', toString: () => `self_destruct(${drone})`})

const Launch = (drone) => liftF(launch(drone)) //a function that if run, returns a Suspend(launch(drone), f) such that f() returns Pure(launch(drone))
const SendSamples = (drone, data, addr) => liftF(send_samples(drone, data, addr))
const SelfDestruct = (drone) => liftF(self_destruct(drone))
{% endhighlight %}

Let's now create a simple *Exec* monad (just a minimal object, actually) so that we can *flatMap()* subsequent functions onto it, after carrying out a side-effect like printing something to the console (or writing something to a log file):

{% highlight javascript %}
const Exec = x =>
({
  flatMap: f => {x(); f();}
})

const interpreterDroneMock = x => {
  // returns an Exec monad containing the function to carry out
  if (x.type == 'launch') {
    return Exec(() => console.log(`Drone ${x.drone} has been launched`))
  }
  if (x.type == 'send_samples') {
    return Exec(() => console.log(`Drone ${x.drone} sent data ${x.data} to address ${x.addr}`))
  }
  if (x.type == 'self_destruct') {
    return Exec(() => console.log(`Drone ${x.drone} has self-destructed`))
  }
}
{% endhighlight %}

Now, we can implement our logic using the capitalized versions *Launch*, *SendSamples* and *SelfDestruct*, and then mock the functionality using *foldMap* together with *interpreterDroneMock*:


{% highlight javascript %}
const topSecretOperation = Launch("NiftySleuth")
  .flatMap(x => SendSamples("NiftySleuth", "{'isTargetVisible': true}", "127.0.0.300:8500"))
  .flatMap(x => SelfDestruct("NiftySleuth"))

topSecretOperation.foldMap(interpreterDroneMock, Exec)
// will print:
// Drone NiftySleuth has been launched
// Drone NiftySleuth sent data {'isTargetVisible': true} to address 127.0.0.300:8500
// Drone NiftySleuth has self-destructed
{% endhighlight %}

However, if we wanted to run the whole shebang for real, we could do:

{% highlight javascript %}
const interpreterDroneRealDeal = x => {
  if (x.type == 'launch') {
    return Exec(() => launchDrone())
  }
  if (x.type == 'send_samples') {
    return Exec(() => sendSamplesDrone())
  }
  if (x.type == 'self_destruct') {
    return Exec(() => selfDestructDrone())
  }
}

const topSecretOperation = Launch("NiftySleuth")
  .flatMap(x => SendSamples("NiftySleuth", "{'isTargetVisible': true}", "127.0.0.300:8500"))
  .flatMap(x => SelfDestruct("NiftySleuth"))

topSecretOperation.foldMap(interpreterDroneRealDeal, Exec)
{% endhighlight %}

Of course, we didn't go into the details of passing the right parameters to the real functions; the details can be filled out easily.