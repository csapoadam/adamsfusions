---
layout: post
title: Functional Dependency Injection Using Monadic Contravariant Functors
date: 2022-08-17 12:20:23 +0900
category: SoftwareDesign
---

There - I knew that title would surprise you! I bet you can feel a rush of excitement, and maybe some trepidation as to what it could mean!

But just like monads from my previous post, you will soon see that functional dependency injection is fun and not that hard, especially using contravariant functors that also happen to be monads. So read on, and only the excitement will remain.

(When writing this post, I have to acknowledge that I was standing (standing? more like kneeling...) on the shoulders of giants. Much credit goes to Brian Lonsdorf, as I've highlighted in multiple instances.)

# Quick recap - What is a functor?

From my previous post, we know that a functor *F\<A\>* is basically a wrapper for a type A that offers the convenience of chaining operations using the *map()* operation. The following *NumberContainer* class provides a basic example:

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

Note also that in this example, we were mapping functions that returned the same type as the type of their argument. Functors that map functions of this kind are referred to as **endofunctors** ('endo', of course, referring to this self-referential quality). Whenever we have an endofunctor, it can also naturally support a so-called *concat()* operation, which allows one to combine different chains of operations at any point. For example, if instead of computing *3(5)<sup>2</sup> + 2*, we wanted to compute *3((5)<sup>2</sup> + 2)*, we could modify the previous example as follows:

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
        (x, y) => x*y
    ).map(val => console.log(`result is: ${val}`))

{% endhighlight %}

We could do this because we were certain that at any point during the chain of computations, the interim result was still a Number (more specifically, an integer value).

# Now let's consider functors containing functions

Interesting new possibilities arise if we stretch our capacity for wild mental associations a bit further and consider functors that contain objects belonging not to a value type, but to a class of functions with a single input and a single output (we could generalize to a broader class of functions later if we wanted). What would be the implications of doing this?

#### What'll happen to map()?

First, let's consider what would happen to *map()*. Note that in general, a function that transforms from type A to type B can be written as *g: A -> B*.

A "plain" functor *F* of type *A* could in general map functions of form *A -> B*, and the result would be a new functor of type *F\<B\>*. In this case, *map()* would have the form:

*map<sub>F\<A\></sub>: (A -> B) -> F\<B\>* 

Returning to functors of functions, we can obtain a similar form as follows:

*map<sub>F<sub>A -> B</sub></sub>: (B -> X) -> F<sub>A -> X</sub>*

When asking ourselves what function the resulting functor could represent, an obvious answer is that it could represent a function that arises as a composition of the originally stored function and the function we are mapping. By running the original function on an input of type *A*, we obtain a value of type *B*, which can then be "fed into" the second function, to obtain a value of type *X*.

Of course, all of this is a potentiality, as the actual functions won't be executed until they are explicitly run. The functions are simply composed 'in memory', and the newly obtained function is wrapped into the same functor type *F*. As we will see, this kind of lazy evaluation is a big advantage of functors holding functions: real execution with side effects can be deferred even as the chain of computations is being assembled.


#### Introducing contramap()

Now that we've successfully generalized *map()* to functors holding functions, let's see what other operations may emerge based on this generalization.

In our previous derivation of *map()*, we eventually realized that mapping a function onto a functor holding a function basically results in function composition: if *g* is mapped onto *F\<h\>*, what we get is basically a new functor of type *F\<gh\>* - which encapsulates a function *gh* that operates by first calling *h* on some input, and then *g* on the result.

One natural variation, of course, would be to do this the other way around. Who's to say (other than our type checker, if we have one) that we couldn't call *g* first, and then *h* on its result? Indeed, we could do this if our types aligned properly:

*contramap<sub>F<sub>A -> B</sub></sub>: (X -> A) -> F<sub>X -> B</sub>*

Functors that have a contramap operation are known as **contravariant functors**. Having a contramap operation is useful if we want to use a functor containing a function, like *F<sub>A -> B</sub>*, but the input value that we have belongs to type *X*, and not type *A*. Let's consider a short example.

#### An example with both map() and contramap()

In the following example, we have a type *FnWrapper* that wraps a function with one argument, and exposes ways both to transform the output - using *map()* - and to transform the input of that function - using *contramap()*. (Note that in this example, instead of using a class to define *FnWrapper*, I used the function-to-object style advocated in the tutorials done by Brian Lonsdorf.)

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

How nice... As you can see, we wanted to run our function with a string input, but it was actually expecting to receive as input a number. So we used *contramap()* to make the necessary transformation in advance.

# Introducing contravariant functors (holding functions) that are also monads

Now let's move on to the icing on the cake: an exciting tool for solving the problem of dependency injection... in an elegant way, of course!

The problem can perhaps best be described based on an example. Consider that as developers creating large-scale applications, we are often confronted with the task of parameterizing the behavior of a system based on so-called environment variables. For example, in a test scenario, we might want to connect to a different database than the one used in production. We might also want to generate log messages that are more verbose than otherwise. This requirement forces us to pass around an environment object in the whole application, even if many parts of the application may have no use for it!

Especially when we want to structure our application in a way that relies on functions composed together with functions of functions of functions, it would be akin to a nightmare if we had to change the signature of all functions and return values so that the composition would work.

Let's see if we can't solve this problem by adding a bit of secret sauce to our earlier contravariant functor.

#### Monadic contravariant functors

Recall that (by-and-large, without going into formal details!) a monad is just a functional datatype that exposes a *flatmap()* function. Compared to *map()*, *flatmap()* will accept a function that maps not from the wrapped value type to an instance of the same type, but from the wrapped value type to a new monad of the resulting stored value type.

So, for example, if we have a functor *F\<A\>*, we could map a function *g: A -> B* onto it and obtain a functor *F\<B\>*. However, if we had a monad of type *M\<A\>*, we could flatmap a function *g: A -> M\<B\>* onto it to obtain a monad *M\<B\>*. The key difference is whether it is the flatmapped function that creates the new monad, or if it is the functor itself, onto which we are mapping our function, that has to wrap the result.

Now let's consider, in a similar vein to our interpretation of *map()*, what it would mean to flatmap a function onto our contravariant functor.

Recall that for a functor encapsulating a function *g: A -> B*, we could write the map operation like so:

*map<sub>F<sub>A -> B</sub></sub>: (B -> X) -> F<sub>A -> X</sub>*

Now, *flatmap()* is a slight variation on the above:

*flatmap<sub>M<sub>A -> B</sub></sub>: (B -> M<sub>C -> X</sub>) -> M<sub>C -> X</sub>*

Adapting our earlier thought processes to this new case, let's consider what kind of function the resulting monad might encapsulate. Once again, a viable solution clearly emerges: if one were to compose the originally stored function (mapping from *A* to *B*) with the mapped function (mapping from *B* to *M<sub>C -> X</sub>*), and then wrap the resulting function into a new monad object M (to store the composed function for later execution), one would obtain a result of type *M<sub>A -> M<sub>C -> X</sub></sub>*. To obtain the desired result, then, one would have to actually run the composed function inside this new monad, to obtain the result of type *M<sub>C -> X</sub>*.

If we are especially perceptive, we can already see that this derivation foreshadows the possibility of dependency injection. That's because the appearance of the new type, *C*, is an interesting key point: at no point did *C* figure into the oringal monad, *M<sub>A -> B</sub>*! It is in fact a new type that was introduced into the chain of computations on the spur of the moment.

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

const p2t3postprocessed = FnWrapper(plus2)
  .map(times3)
  .contramap(env => env.input)
  .flatmap(res => FnWrapper(
      env => env.pproc(res)
    )
  )

console.log(`xPlus2Times3 applied to 5 then doubled = ${p2t3postprocessed.run({
  pproc: times2,
  input: 5
})}`) // 42 indeed...

console.log(`xPlus2Times3 applied to 5 then plustwo'd = ${p2t3postprocessed.run({
  pproc: plus2,
  input: 5
})}`) // 23 indeed...

{% endhighlight %}

Here, we can see that *flatmap()* is implemented exactly as described: a new Monad is created, that contains the executed composition of the original and the flatmapped function. Also, what's really interesting is that the flatmapped function could make use of both the interim results of the earlier chain of computations (*res*) as well as the environment variable *env* which we passed into it only at the very end.

I recommend that you take off a day from work to contemplate the wonders of this pattern. I for one am very grateful to Brian Lonsdorf and others in the FP community for awakening me to its beauty.