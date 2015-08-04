---
title: On fluent interfaces
layout: post
---

Since I touched C# in my last post I've decided to put another remark here. This time it is about **fluent interfaces**.

This subject pops up here and there from time to time when people discuss convenient ways of building APIs or just ways to express things better in code.

In F# we don't really need "fluent interfaces" because we can compose a "pipeline" that looks and reads very natural.  

**I promise that the point of this article is to explain C# stuff**, and I will come back to C#, but just in case if you haven't done any F# I'll explain pipelines a bit so it becomes clear.

### How it works in F#?
Consider the following example:

{% gist alexeyraga/bc07a9ac731e054a35f8 1-Pipelines-FSharp-example.fs %}

The example above uses the pipeline operator `|>` to compose the computation. What does it do?  
I put its full code for a reference in the first line of the example so you can see it yourself, but basically it is a function that takes a value `a` and a function `f` and then apply `f` to `a`, nothing else. So it takes a value and "pipes" it into the function. On lines #5 an #6 the value `user` is piped into the function `login`, which is equivalent of just calling `login user`.

Then it gets more interesting. We continue piping the result of a "previous" function to the "next" function and so on. This way we always have exactly **one** value to pipe into the "next" function, and the signature of `|>` clearly suggests that.  
But what if the "next" function takes more than one parameter?

This is still possible due to F#'s language features such as **currying** and **partial application**.

>Currying is the technique of translating the evaluation of a function that takes multiple arguments into evaluating a sequence of functions, each with a single argument (partial application)

Currying basically means that any function that takes more than one argument can be viewed as a function that takes only one argument and returns a function that takes the second argument, and returns a function that takes third argument and so on until all the arguments are taken, then the result is produced.

For example, if we have a function that takes 2 `int`s and sums them together:
{% highlight fsharp %}
let add x y = x + y
{% endhighlight %}

We could write the type of this function as:
{% highlight fsharp %}
add: int -> int -> int

//it is equivalent of
add: int -> (int -> int)
{% endhighlight %}

We read this type signature as "function `add` takes an `int`, then another `int`, and then produces an `int`".  
Or we can read it as: "function `add` takes one `int` and returns a function from `int` to `int`."  

These two are equivalent in F#.  
It is like the compiler is thinking "well, I need 2 arguments, but you only gave me one and I still need another one, so I return you a function that takes a missing argument and then produces the result".  
All functions in F# are curried by default so in reality it is just given, we don't need to think about it much when we write code.

But how it is helpful?  
It comes very handy with the concept of **partial application**. Let's look at the example:
{% highlight fsharp %}
//add5: int -> int
let add5 = add 5
{% endhighlight %}

Notice two things:

1. `add5` is expressed in terms of `add` that is only provided with one argument (the second one is "missing").
2. The result of `add 5` has type `int -> int` because `add` still needs another argument! So `add5` is a function from `int` to `int`.

We used function `add` and kind of "fixed" its first argument to `5` so it will always adds `5` to the remaining argument.  
`add5` is a **partially applied** `add`.

This is exactly how the whole `latestRequest` works. Functions like `clickMenu` or `openFilter` are **partially applied**, and the remaining (last) argument is piped into them from the result of the "previous" function.  

As you see, the concept is nice and simple, and the F# makes it easy by providing carrying out of box. With this approach we can compose any functions in the way we want "fluently" and we don't need to do anything special.

###Back to C#  
Fluent interfaces in C# often involve, well, building interfaces, and functions on them, and implementing them, and figuring out whether return types of these functions should also be interfaces (because we want to continue to be fluent), and/or doing some tricks with generic interfaces, and thinking forward from the evolution/extentibility point of view, etc.

It is often not super complex, but it is complex enough to require making design decisions and careful implementation. It **is** more complex than being able to compose functions in a way we want and have the result. Couldn't we do just this?

C# doesn't do carrying for us, nor partial application is possible. When you supply not enough arguments to the function you simply get a compilation error. But it is still possible to achieve the same compositional goals, although with a little bit of boilerplate:

{% gist alexeyraga/bc07a9ac731e054a35f8 2-Pipelines-CSharp-substitution.cs %}

All these functions like `Login`, `ClickMenu`, `OpenFilter`, etc. are just "normal" C# functions, nothing special about them at all. But instead of just calling them one-by-one, getting results and passing these results further we compose them using `Then` combinator.

`Then` is an extension method is very simple.

- As any other extension method that takes a "source" value `self`,
- And it takes a function that it is going to call (`selector`),
- And a list of parameters that we want to pass to that function, except the last one, because it will use that `self` as a last parameter.

So `Then` kind of simulating partial application in C#, we are able now to "fix" all the parameters except the last one. The last parameter is not fixed and it will be the value we call `Then` extension method on.

Now we have our "fluent interface" for free. We only need functions that can be "chained" together so the result of a "previous" function is used as a parameter for the "next" one. We can now concentrate of building functions that deliver results instead of thinking about which interfaces they should be declared in and which types to return to maintain "fluency". "None" is the best answer :)
And, of course, compiler takes care of types so it would be impossible to pipeline "incompatible" functions.

Have a nice day. Use functions.
