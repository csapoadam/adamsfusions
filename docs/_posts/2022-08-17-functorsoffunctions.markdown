---
layout: post
title: Functional Dependency Injection Using Monadic Contravariant Functors of Functions
date: 2022-08-17 12:20:23 +0900
category: SoftwareDesign
---

There - I knew that title would surprise you! But just like monads from my previous post, you will soon see that functional function modeling is fun, especially using contravariant functors of functions that are also monads.

# Quick recap - What is a functor?

From my previous post, we know that a functor *F\<A\>* is basically a wrapper for a type A that offers the convenience of being able to chain operations using the *map()* operation. The following *NumberContainer* class provides a basic example:

{% highlight javascript %}
class NumberContainer {
    constructor(val) {
        this.value = val;
    }
    
    map(fn) {
        return new NumberContainer(fn(this.value));
    }
}

const pow2 = val => Math.pow(val, 2);
const mul3 = val => val * 3;
const add2 = val => val + 2;

let w = new NumberContainer(5)
    .map(pow2)
    .map(mul3)
    .map(add2)
    .map(val => console.log('result is:', val))
{% endhighlight %}

The obvious benefit here is that the operations *pow*, *mul3*, etc. can be chained together in a highly readable way, by abstracting away the mechanics of storing interim results and passing them on to the next function call.

Note also that in the above example, we were mapping functions that returned the same type as the type of their argument. Functors that map functions of this kind are referred to as **endofunctors** ('endo', of course, referring to the fact that the resulting value is of the same type as the original value). Whenever we have an endofunctor, it can also naturally support a so-called *concat()* operation, which allows one to combine different chains of operations at any point. For example, if instead of computing *3(5)^2 + 2*, we wanted to compute *3((5)^2 + 2)*, we could modify the previous example as follows:

{% highlight javascript %}
class NumberContainer {
    constructor(val) {
        this.value = val;
    }
    
    map(fn) {
        return new NumberContainer(fn(this.value));
    }

    concat(other, op) {
        return new NumberContainer(op(this.value, other.value))
    }
}

const pow2 = val => Math.pow(val, 2);
const mul3 = val => val * 3;
const add2 = val => val + 2;

let w = new NumberContainer(3)
    .concat(
        new NumberContainer(5)
            .map(pow2)
            .map(add2),
        (x, y) => x*y)
    .map(val => console.log('result is:', val))

{% endhighlight %}

We could do this because we were certain that at any point during the chain of computations, the interim result was still a Number (more specifically, an integer value).

# Now let's consider functors containing functions

Interesting new possibilities arise if we relax our brains a bit further and consider functors that contain objects that belong not to a value type, but to a class of functions with a single input and a single output (we could generalize to a broader class of functions later if we wanted). What would be the implications of doing this?

#### What'll happen to map()?

First, let's consider what would happen to *map()*. Note that in general, a function that transforms from type A to type B can be written as *g: A -> B*, or, to use prefix notation, as *g: (-> A B)*.

A "plain" functor *F* of type *A* could in general map functions of form *A -> X*, and the result would be a new functor of type *F\<X\>*. In this case, *map()* would have the form:

*map<sub>F\<A\></sub>: (A -> X) -> F\<B\>* 

Returning to functors of functions, we can in fact obtain the same form by transforming our original function *g: (-> A B)*, to create a new function *g<sub>A</sub>: () -> B*, which already encapsulates, deep within its internals, a value of type *A* and just has to hold a value of type *B* (which it will return when executed). Supposing we had a function *h: B -> X*, the *map()* function for the associated functor, *F<sub>gA</sub>\<B\>*, then, would simply look like this:

*map<sub>F<sub>gA</sub>\<B\></sub>: (B -> X) -> F<sub>hgA</sub>\<X\>*

Here, we have of course changed the input type of the mapped function to *B* because this is the type that the starting functor is holding. But the main point is that as we can see, by mapping a function that transforms between types *B* and *X* to such a functor holding a value of type *B*, we can obtain a new similar functor that holds a type *X* (and encapsulates a function that transforms from type *A* to type *X*). Such behavior can be naturally obtained by first calling the function stored inside F<sub>gA</sub>\<B\>, then calling the mapped function *g: (B -> X)* on the result, and wrapping the output into a new functor of type F<sub>gA</sub>. This is basically function composition!

And now that we have ascertained that this is the case, we can return to our original notation with a slight modification, and write the following:

*map<sub>F<sub>A -> B</sub></sub>: (B -> X) -> F<sub>A -> X</sub>*

To summarize, when we want to map a function onto a function stored inside a functor, we first execute the stored function, then call the mapped function on the result, and finally wrap up the result into the same type of functor.

#### Introducing contramap()

But wait, the fun goes even further. In our previous derivation of *map()*, we eventually realized that mapping a function onto a functor holding a function basically results in function composition: if *g* is mapped onto *F\<h\>*, what we get is basically an *F\<gh\>* - which encapsulates a function that first calls *h*, then calls *g*.

One natural variation, of course, would to do this the other way around. Who's to say (other than our type checker, if we have one) that we couldn't call *g* first, and then *h* on its result? Indeed, we could do this if our types aligned properly:

*contramap<sub>F<sub>A -> B</sub></sub>: (X -> A) -> F<sub>X -> B</sub>*

Functors that have a contramap operation are known as **contravariant functors**. Having a contramap operation is useful if we want to use a functor containing a function, like *F<sub>gA</sub>\<B\>*, but the input value that we have belongs to type *X*, and not type *A*. Let's consider a short example.

#### An example with both map() and contramap()

In the following example, we have a type *FnWrapper* that wraps a function with one argument, and exposes ways both to transform the output (using *map()*) and to transform the input of that function (using *contramap()*):

{% highlight javascript %}
class FnWrapper {
  constructor(fn) {
    this.fn = fn
  }

  map(f) {
    return new FnWrapper(x => f(this.fn(x)))
  }

  contramap(f) {
    return new FnWrapper(x => this.fn(f(x)))
  }

  run(x) {
    return this.fn(x)
  }
}

const plus2 = x => x + 2;
const times3 = x => 3 * x;

const xPlus2Times3 = new FnWrapper(plus2)
  .map(times3)

console.log(`xPlus2Times3 applied to '5' = ${xPlus2Times3.contramap(x => Number(x)).run('5')}`) // 21 indeed...
{% endhighlight %}

A more succinct way of writing this example would be to use an object-based approach as advocated in Brian Lonsdorf's tutorials:

{% highlight javascript %}
const FnWrapper = fn => ({
  map: f => FnWrapper(x => f(fn(x))),
  contramap: f => FnWrapper(x => fn(f(x))),
  run: x => fn(x)
})

const plus2 = x => x + 2;
const times3 = x => 3 * x;

const xPlus2Times3 = FnWrapper(plus2)
  .map(times3)

console.log(`xPlus2Times3 applied to '5' = ${xPlus2Times3.contramap(x => Number(x)).run('5')}`) // 21 indeed...
{% endhighlight %}

How nice...

# Introducing contravariant functors (holding functions) that are also monads

Now let's move on to the icing on the cake. An exciting tool for solving the problem dependency injection.

As a programmer, we are often confronted with the task of parameterizing the behavior of a system based on so-called environment variables. For example, in a test scenario, we might want to connect to a different database than the one used in production. We might also want to generate log messages that are more verbose than otherwise. This requirement forces us to pass around an environment object in the whole application, even if many parts of the application may have no use for it!

Especially when we want to structure our application in a way that relies on functions composed together with functions of functions of functions, it would be akin to a nightmare if we had to change the signature of all functions and return values so that the composition would work.

Let's see if we can't solve this problem by adding a bit of secret sauce to our earlier contravariant functor (or once I share it with you, not so secret sauce...)

#### Monadic contravariant functors

Recall that a monad is just a functor that also exposes a *flatmap()* function. Compared to *map()*, *flatmap()* will accept a function that maps not from the stored value type to a resulting stored value type, but from the stored value type to a new monad of the resulting stored value type.

So, for example, if we have a functor *F\<A\>*, we could map a function *g: A -> B* onto it and obtain a functor *F\<B\>*. However, if we had a monad of type *M\<A\>*, we could flatmap a function *g: A -> M\<B\>* onto it to obtain a monad *M\<B\>*. The key difference is whether it is the flatmapped function that creates the new monad, or if it is the functor itself, onto which we are mapping our function, that has to wrap the result.

Now let's consider, in a similar vein to our interpretation of *map()*, what it would mean to flatmap a function onto our contravariant functor.

Recall that for a functor encapsulating a function *g: A -> B*, we could write the map operation like so:

*map<sub>F<sub>A -> B</sub></sub>: (B -> X) -> F<sub>A -> X</sub>*

Now, *flatmap()* is a slight variation on the above:

*flatmap<sub>M<sub>A -> B</sub></sub>: (B -> M<sub>A -> X</sub>) -> M<sub>A -> X</sub>*

To map our new function, *h: B -> M<sub>A -> X</sub>*, we would first have to run the function inside *M<sub>A -> B</sub>*, thus obtaining an intermediate result of type *B*. Then, we would run the flatmapped function to obtain a monad of type *M<sub>M<sub>A -> X</sub></sub>*. We would then have to flatten out the result, i.e., create a new monad encapsulating just the function of type *A -> X*.

#### Monadic adaptation of our previous FnWrapper example

To continue our previous example, consider the following modifications:

{% highlight javascript %}

const FnWrapper = fn => ({
  map: f => FnWrapper(x => f(fn(x))),
  contramap: f => FnWrapper(x => fn(f(x))),
  run: x => fn(x),
  flatmap: other => FnWrapper(x => other(fn(x)).run(x)),
  concat: other => FnWrapper(x => other.run(fn(x))),
})

const plus2 = x => x + 2;
const times3 = x => 3 * x;
const times2 = x => x * 2;

const xPlus2Times3PostProcessed = FnWrapper(() => 5)
  .map(plus2)
  .map(times3)
  .flatmap(res => FnWrapper(pproc => pproc(res)))

console.log(`xPlus2Times3 applied to 5 then doubled = ${xPlus2Times3PostProcessed.run(times2)}`) // 42 indeed...

console.log(`xPlus2Times3 applied to 5 then plus 2 = ${xPlus2Times3PostProcessed.run(plus2)}`) // 23 indeed...

{% endhighlight %}