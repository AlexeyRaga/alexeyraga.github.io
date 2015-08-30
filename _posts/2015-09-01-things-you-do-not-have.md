---
title: On things you don't have
layout: post
tags: [F#, C#]
---

Any (good) programming language offers a lot of very useful things. These things are usually pretty much well known among developers programming in this language.
However, there are many other things that particular language **does not** provide, and these things are typically remain completely unknown or totally misunderstood by the same developers, well, because they do not exist in their universe.
It is like early iPhone users who would say "who needs copy/paste on a phone, nonsense!". Right until this feature became available to them and they realised how useful it was.

Now, why one would even look at another programming language?  
One reason would be to escape from the [Blub paradox](https://en.wikipedia.org/wiki/Paul_Graham_(computer_programmer)#The_Blub_paradox) and to better understand his options.  
Another reason is that a programming language is just a tool in your toolbox. No carpenter would ever say "I am comfortable with a Sledge Hammer so I don't even need to know about other options". That would probably be a silly thing to say. We, developers, are no different.  
It is very pragmatic to know the options and then to be able to choose the most effective tool for the job rather than the most familiar one.

In this article I point to **some** of the very useful "features" that C# doesn't offer, but other languages do. For the purpose of this article (and to stay within the .NET boundaries) I will only mention things that exist in F# but not in C#.

Note, that it is not a comprehensive list by any means, just some features I picked and decided to write about this time.

### Type Providers
"**Type providers**" is a mind blowing feature that is, as far as I know, unique to F# and does not exist in other languages.  

It is better to start with a screenshot:

![FSharp SQL Type Provider]({{ site.url }}/assets/FSharpSqlTypeProvider2.png)

What happens here is just fantastic. When SQL Type Provider is pointed to a database (given a connection string), it gets the schema, interrogates it and "automagically" creates the types. In design time, so on the very next line you have the access to all the tables and fields, with an intellisense and an autocomplete. All statically typed.  
It is like if you generated these types using some ORM such as EntityFramework, except that you don't need to do anything. It is even better because the type provider and the compiler make sure that the code is always in sync with the database schema.  
And it works for any database, all you need is to provide it with a connection string. ORM in two lines of code, isn't it great?!

"Not a big deal, I am just comfortable with my EntityFramework", one can say.  
OK, and what about CSV? Or JSON? Or XML? Or Atom? What if this JSON or XML or Atom is a feed somewhere in the Internet? Like stock market statistics, or Netflix, or [Freebase](https://www.freebase.com/), or WorldBank data sources? Or your own JSON API?

Here is an example of how you could programmatically query StackOverflow in F#:

![FSharp Atom Type Provider]({{ site.url }}/assets/FSharpAtomTypeProvider.png)

Similar two lines of code except that now the OData Type Provider is used. As you can guess, not only StackOverflow, but any OData-compatible source (such as atom) can be used with this provider.

Type providers is a mind blowing feature. There are lots of type providers exist of all sorts: Amazon S3, COM, Azure, HTTP, Json, XML, SQL, OData, R, Yaml, MathLab, DynamicsCRM, WorldBank, etc., etc.  
There is even WMI Type Provider so you can easily query and manage your operating system from F# accessing pretty much everything, in 2 lines of code.

And it is not that hard to build a type provider for something if there isn't one already. For example, you can build a type provider for your product and it will serve as a nearly perfect API that you can build on top or use when integrating with other systems.

If you are a bit intrigued by now then have a look at the [WorldBank type provider](http://fsharp.github.io/FSharp.Data/library/WorldBank.html) examples and play with it trying to find some interesting correlations.


### Record types
Records are simple types that contain named values and, optionally, methods.

{% highlight fsharp %}
type FirstName = FirstName of string
type LastName  = LastName  of string
type Age       = Age       of int

type Person = { name: FirstName; surname: LastName; age: Age }
{% endhighlight %}

The first 3 lines are just immutable type declarations, consider them as wrappers. You could (and, as we agreed [before]({% post_url 2015-08-01-null-propagation-and-code-contracts %}), should) do it in C# too. Unfortunately it will take you about 50 lines of code for each type, but it can never be an excuse, can it?  
In F# it is a one-liner, you just declare the essence of your type and the compiler does all the heavy lifting.

The last line is declaring a **record type**. It is also immutable, and it also gives you all these `Equals`, `GetHashCode`, `IComparable`, etc.  
On top of that it declares a "copying constructor" so you can easily "update" the record. Of course, in F# you don't think much in terms of "constructors", it is taken care of, so you can concentrate on your business logic.

A toy example:

{% highlight fsharp %}
// creating an instance of Person
let mary26 = { name = FirstName "Mary"; surname = LastName "Stuart"; age = Age 26 }

// create a copy with `age` incremented
let mary27 = { mary26 with age = Age 27 }

// evaluates to `true`
let older = mary27 > mary26
{% endhighlight %}

You can look at these type declarations as at just syntactic sugar and "not a big deal", but it actually **is a big deal**. It changes how you think about your types and your code.  
And anyway, isn't it great to be able to express something fairly complex in just a couple of lines of a **very simple** code? Hurray, no more ridiculous rules like "one file per type"!

### Sum types
C# only has what is called **product types**. When you declare a class that contains a couple of properties - it is a **product type**. It is typically written in literature as `P = A * B * C`. For languages like C# it means that type `P` is a composition of `A` **and** `B` **and** `C`. For example, `Point` is a composition two `int`s, and `Tuple<A, B>` is a <s>composition</s> product of `A` and `B`.  
In fact, the type `Person` that is declared above is a product type too, as most of the types that you create daily.

**Sum types** are different. These are disjunctions. What is written as `P = A + B` means that type `P` can be **either** `A` **or** `B`.

This is something that is very hard to model in C#. We are working around it by inventing all sorts of complexity and ceremony simply because C# doesn't offer anything better. And because we don't know any better we think that this complexity is normal as it should be.  
F# has sum types baked in the language, they are called `discriminated unions` (because MS likes naming things differently).

In fact one of the examples of sum types we have already discussed [before]({% post_url 2015-08-01-null-propagation-and-code-contracts %}), it is the `Option` type that is declared as:

{% highlight fsharp %}
type Option<'a> = Some of 'a | None
{% endhighlight %}

This means exactly how it reads: the `Option` is a generic type that can **either** be `None` **or** `Some`, and if it is `Some` then it also carries a value of type `'a`.

{% highlight fsharp %}
type DownloadResult =
  | OK of string
  | NotChanged
  | Error of string
{% endhighlight %}
This is how you could model some download result: simply declare a type that can represent **either** a success with the content, **or** indicate that nothing has changed since you did it last time, **or** an error.

The following example is interesting because the structure is recursive. It is a tree where each node can **either** be a leaf **or** a branch of two trees (left and right). Every developer has implemented this structure at least once, here how it looks like when you have sum types in your language:

{% highlight fsharp %}
type Tree<'a> =
  | Branch of left: Tree<'a> * right: Tree<'a> //'left' and 'right' trees
  | Leaf of 'a
{% endhighlight %}

Now imagine (or remember) implementing it in C# and compare the effort required.

In C# we often abuse `enum`s for things they don't really supposed to be used for. `Enum`s are often used for result codes, colours, statuses, roles, etc.  
Sum types are often much better than `enum`s in these cases because:

- Sum types are real types, not just aliases to `int` or `byte`.
- Sum types are closed. `Enum`s are just numbers, and you can assign `9876` to your `enum` value of "role", even if it is "incorrect" and is "outside of the range" of your `enum`. With sum types it is not possible.
- Sum types can carry more information. Like `Some` that carries a value, and `Branch` that carries a couple of trees. (Look at the `DownloadResult` example again)
- Compiler won't let you shoot yourself in the foot! It will not be possible to use `NotChanged` where `OK` is expected, etc.

Consider this example where we want to model a small kingdom:

{% highlight fsharp %}
type Rank = Sovereign | Heir | Peasant

type Command =
    | RegisterBirth of Person * Rank
    | IncreaseRank  of PersonId
    | RegisterDeath of PersonId

let personId = sendCommand RegisterBirth(mary, Heir)

//after 6 days Mary's father dies...
let newRank  = sendCommand IncreaseRank(personId) //and Mary is a queen now!
{% endhighlight %}

This example looks simple, it is declarative and easy to understand, but it alone worth pages and pages of C# code in several files. Note also that the code above is trivial to write, read and maintain.

### Pattern matching
Pattern matching is often viewed as a "switch on steroids". Indeed, it can be used like one:

{% highlight fsharp %}
let guess a =
    match a with
    | 1         -> "One"
    | 2 | 3 | 4 -> "Not too many"
    | value     -> sprintf "%d is a lot!" value
{% endhighlight %}

Here the function `guess` **matches** the value of the argument `a` against some cases and produces the result, in the same way as the `switch` statement in C# would be used.

But there is much more to pattern matching. Starting with a feature that is so heavily requested in C# that it will probably be included in a one of its next releases:

{% highlight fsharp %}
let someTuple = ("Mia", 27)
let (name, age) = someTuple
{% endhighlight %}

The first line just creates a tuple of two elements.  
The second line is interesting: it "decomposes" the tuple into two individual values so you don't have to write things like `name = someTuple.Item1` etc. When implemented in C# it will probably be some special case, a syntactic sugar only for tuples (let's hope that I'm wrong).  
In F# it is yet another example of **pattern matching**.

There is another one:

{% highlight fsharp %}
let guessList list =
    match list with
    | []                    -> "Empty list"
    | [ x ]                 -> sprintf "Singleton list with %O" x
    | [ x ; y ] when x = y  -> sprintf "%O twice!" y
    | [ _ ; y ]             -> sprintf "Two elements with second %O" y
    | [ x ; _ ;  z]         -> sprintf "Three elements, starts with %O and ends with %O" x z
    | _                     -> "Didn't expect that..."
{% endhighlight %}
Here is how we could match on lists. This cannot be done with the regular `switch` because patterns here are more complex: the length of the list is involved, elements are extracted, conditions are applied (the `when` clause in the 3rd pattern).  
Now you can probably see why it is called **pattern matching**.

A slightly more complicated example would be:

{% highlight fsharp %}
let buyAlcohol person =
    match person with
    | {Person.name = FirstName("Mary")} -> "Mary is a queen, buy at will"
    | {Person.age = a} when a >= Age 21 -> "Here is your beer"
    | {Person.name = FirstName(nm)}     -> sprintf "%s is not allowed to buy alcohol" nm
{% endhighlight %}

This example above is matching against the record type (as well as against our "wrapper" types such as `FirstName`).  
The first pattern matches any person whose first name is "Mary".  
The second one doesn't care about anything but the person's age, and it makes the age accessible as `a`. The `when` guard guarantees that the pattern only matches if the person is older than 21.  
The last pattern matches any person at all. It just extracts person's first name into the value named `nm`, so it can be used in the result string.

Pattern matching goes way beyond this, you can match against records, arrays, lists, tuples, types, unions, etc. You can have conditions for patterns to express that "it matches this shape, but only if that condition holds". You can even define your own patterns.  
Look at the [list of the supported patterns](https://msdn.microsoft.com/en-us/library/dd547125.aspx) on MSDN and you will be truly surprised of what it can do for you. Then, to be impressed even more, read the [Active Patterns](https://msdn.microsoft.com/en-us/library/dd233248.aspx) section that explains how you can define your own patterns for your specific cases and then to use them as they were built in the language.

### Type inference
You have probably noticed but none of the examples above specifies types in function declarations. Not even parameters' types. But it doesn't mean that types are not there! F# is a statically typed language (just like C#), but it has much better type inference system.

In C# type inference pretty much ends with the `var` keyword and lambda functions. In F# specifying types explicitly is almost never required. The compiler infers types for you!  
You still **can** specify types explicitly **if you want**, but it is not how people typically write F# code.

### Conclusion
In this article I picked some of the features I really miss in C#. It is not a comprehensive list, and I only lister things from F#, simply because I wanted to make this article more relevant for developers who want or have to stay in .NET ecosystem.  

For those I could advice: give F# a try. Do it not for the sake of learning another language, but for very pragmatic and result-oriented reasons: you write **much** less code, you write **much** simpler and trivial code, you write **much** less boilerplate and ceremony. And you still stay with .NET, so you don't change anything except the words you type in your IDE :)

One caveat though, from my personal experience with F#: it will be very painful when you try writing in F# just as you would write C# code, only changing the syntax. The mindset will have to change, and it will change quickly as soon as you realise that. Then the pain goes away, and the fun, easiness and joy come and stay.  
So if you feel pain - remember what I just said and think.

F# offers, perhaps, everything that C# does, and it offers more. Someone said once that for a long time there was no feature introduced in C# language without a disclaimer of "and F# already can do it".

Not falling into a trap of "I am comfortable with a Sledge Hammer", looking around and knowing your options is always beneficial **for pragmatic reasons**. The [Blub paradox](https://en.wikipedia.org/wiki/Paul_Graham_(computer_programmer)#The_Blub_paradox) is yet another trap to avoid, and knowing things we don't have could be just as important as knowing things we do have. It is because it gives us choice and lets us to be more efficient in what we do.

Have a nice day. Use languages.
