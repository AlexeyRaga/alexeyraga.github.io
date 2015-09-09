---
title: On scary terms
layout: post
---

In my previous posts I tried to avoid using terminology as much as possible. Hey, I have been doing exactly what Microsoft has been doing all these years: trying to invent new terminology to avoid scaring people :)  
With one big difference: I don't expect my "simple" terminology to replace the "real" one in my readers' heads pretending that the "real" and "scary" one does not exist.

I don't know why some developer freak out when they see or hear some terms, but that's a fact. When you say "context" or "`Select`" or "Workflow" then everything is fine and easy. But when you call these things with their "real" names then some people push back and think that it is overcomplicated or otherwise crazy.  
From this perspective Microsoft's move (inventing new "easy" terms) is smart, except that the "common" terminology remains unknown by developers.  

In fact, Microsoft is not alone here. The `Rust` language recently has introduced one of the "scary" concepts in the language, the-one-we-don't-name, and they called it "Burrito". Yes, they make fun of this situation with the terminology, but nevertheless...

In this article I will not introduce anything new. All the stuff you already know. But I will name things with their "scary" names and explain what they are.

In the end of the day terms are just words, and it is strange to be afraid of one word and be happy use another one when both words mean the same thing :) It is also somehow cool and often useful to know what things really are.

### `.Select(...)` or "mapping over a context"
I mentioned [contexts]({% post_url 2015-09-05-contexts %}) before. I also mentioned how we can **map a function over a context** so a function can access and transform a value within a context. In C# you can use `.Select` combinator to do it. Some contexts don't define `.Select` but provide other methods instead. For example, `Task<T>` provides `.ContinueWith` method to do basically the same thing: apply a function to the value within the context of a future.  

Using `.Select` is just easier because it unifies the API for the same pattern. The pattern is, again, to transform a value within a context. Which context? Does't matter that much, really. Could be anything.

Pretty much everywhere except in C# this operation is called `map`. That's why we say that we **map a function over a context**. C# calls it `Select`.  
When we have a context that we can map over, we can think of it as of some "mappable" context. We could even imagine some `IMappable` (or `ISelectable`) interface this context could implement. C# gives us extension methods, so we can define `.Select` without introducing an extra interface.

So what is expected from that hypothetical `IMappable`? Or without one, what is expected from a "mappable context"?  

Not much.  
**First**, it would only be logical to assume that nothing changes if we map a function that doesn't do anything over a context:

{% highlight csharp %}
var somethingNew = something.Select(x => x);
{% endhighlight %}

Here the function `x => x` (otherwise called an **identity function**) doesn't change anything. Obviously we expect that `something` and `somethingNew` are identical.

**Second**, if map two functions over a context then the result should be the same as if we map one first, and then map another one:

{% highlight csharp %}
var one = something
      .Select(x => DoSomething(x));
      .Select(y => DoSomethingElse(y))

//is the same as
var two = something
      .Select(x => DoSomethingElse(DoSomething(x)))
{% endhighlight %}

In this example we also expect `one` and `two` to be identical.

Very logical, isn't it? Your would be surprised if these simple rules didn't hold, it would be impossible to reason about the code or to refactor it. All of this you already know, no surprises.

The context that provides this ability to map over, this hypothetical `IMappable` is otherwise called a **Functor**.  

`IEnumerable<T>` is a **functor** because there exist a `map` operation that obeys these two rules. This operation is provided by the `.Select` combinator.  
`Task<T>` is also a functor and its `map` operation is provided by the `.ContinueWith` combinator. It is convenient, however, to unify the API and to have the same combinators for the same concept. Defining `.Select` on `Task<T>` (and other contexts) is useful from this perspective and here is how [it can be done](http://blogs.msdn.com/b/pfxteam/archive/2013/04/03/tasks-monads-and-linq.aspx)

A **Functor** term is not scary, it just means that you have something that you can `map` over in a predictable way. It is a pattern. In a pseudo code its definition would be:

{% highlight haskell %}
fmap :: (a -> b) -> F a -> F b
{% endhighlight %}

which means that the `fmap` operation (`fmap` stands for `functor map`) takes a function from `a` to `b`, and some value `a` in a context of `F`, and returns some value `b` in the same context `F`. So it transforms a value within `F`  
What is `F`? We don't care. Some context. It could be `IEnumerable`, or it could be `Task`, or it could be `Option`, or anything else.  
How `F a` is transformed into `F b`? Well, there is no other way but to apply a function `a -> b` within `F a`.

{% highlight haskell %}
fmap :: (a -> b) -> List a   -> List b
fmap :: (a -> b) -> Task a   -> Task b
fmap :: (a -> b) -> Option a -> Option b
{% endhighlight %}

Not surprisingly the `.Select` combinator for `IEnumerable` looks very similar (see how my pseudocode is shorter and cleaner):

{% highlight csharp %}
public static IEnumerable<B> Select<A, B>(
  this IEnumerable<A> source,
	Func<A, B> selector
)
{% endhighlight %}

It is designed to be a `map` operation. And C#'s syntax is specifically designed to support LINQ for any types, not only for `IEnumerable` so you can "promote" other contexts to be **functors** by implementing `Select` extension methods for them.

### `.SelectMany` and composition
OK, we know how to transform a value within a functor. But how we compose contexts themselves? How do we compose two functions when both of them return contextful values?

Imagine we have two asynchronous operations, one that resolves dns names from ip addresses, and another one that goes the other way around. They both return `Task` for a reason: it takes time to make a dns query.   

{% highlight csharp %}
public static Task<IpAddress> ToIp(DnsName name) { ... }
public static Task<DnsName> ToName(IpAddress) { ... }
{% endhighlight %}

All we want is to check if the name/address pair can be resolved both ways: a name is resolved to an IP address and that IP address is resolved back to the same name. We want to compose these calls somehow.

We cannot use Functor and map one function over the result of another because the result type would not be what we want:

{% highlight csharp %}
Task<Task<DnsName>> result = ToIp(name).Select(ip => ToName(ip));
{% endhighlight %}

The result type here is `Task<Task<DnsName>>` instead of expected `Task<DnsName>`.

This problem is not unique for `Task`, other functors have the same behaviour:

{% highlight csharp %}
//convert list of strings into list of chars
IEnumerable<IEnumerable<Char>> result = GetWords(text).Select(w => ToChars(w));

//create database connection from the config
Option<Option<Connection>> conn = config
    .GetConnStr("db")
    .Select(connstr => MakeConn(connstr));
{% endhighlight %}

In the last example the `.GetConnStr` returns `Option` because the configuration file may not have the required settings, and the `.MakeConn` also returns `Option` because it is possible that the connection cannot be created for the specified connection string. Again, the result type is `Option<Option<Connection>>` when `Option<Connection>` is expected.

We already know the solution to this problem: use `.SelectMany` instead of `.Select`. It is especially easy to see in the "list of strings into list of chars" example.

`.SelectMany` allows us to compose two functions when both of them return contextful values:

{% highlight haskell %}
ToIp   :: DnsName   -> Task IpAddress
ToName :: IpAddress -> Task DnsName

GetWords :: String   -> List String
ToChars  :: String   -> List Char

GetConnStr :: String           -> Option ConnectionString
MakeConn   :: ConnectionString -> Option Connection
{% endhighlight %}

With `SelectMany` the result types would be as expected:

{% highlight csharp %}
//reverse dns query
Task<DnsName> reverseName = ToIp(name).SelectMany(ip => ToName(ip));

//convert list of strings into list of chars
IEnumerable<Char> result = GetWords(text).SelectMany(w => ToChars(w));

//create database connection from the config
Option<Connection> conn = config.GetConnStr("db").SelectMany(cs => MakeConn(cs));
{% endhighlight %}

Thats what **Monads** are about: they allow us to compose two functions when each of them returns a contextful value.  
Just like if you have two functions `a -> b` and `b -> c` you can compose them into `a -> c`, you can do exactly this if when the return types are contextful:

{% highlight haskell %}
'normal' functions     | 'contextful' functions
-----------------------|--------------------------
  f :: a -> b          |  f :: a -> M b  
  g :: b -> c          |  g :: b -> M c
-----------------------|--------------------------
  h :: a -> c          |  h :: a -> M c
{% endhighlight %}

What is `M` here? I don't know. Can by anything. Can be `IEnumerable`, `Option` or `Task`, it's all the same pattern.  
How the composition happens? Well, `SelectMany` makes it happen. Each of the contexts implements `SelectMany` in a way that it can "bind" two parts together and provide the result value.  

Just as `Select` represents **Functor**'s `map` operation, `SelectMany` represents what is called **Monad**'s `bind` operation. Microsoft just renamed it, again.

The monadic `bind` combinator has the following type:

{% highlight haskell %}
bind :: M a -> (a -> M b) -> M b
{% endhighlight %}

It takes some value `a` in a context of `M`, and a function from `a` to a contextful value `b` (the context `M` stays the same), and returns that `M b`.

and it is not a surprise that `SelectMany` for `IEnumerable` has the same shape:

{% highlight csharp %}
public static IEnumerable<B> SelectMany<A, B>(
	this IEnumerable<A> source,
	Func<A, IEnumerable<B>> selector
)
{% endhighlight %}

It takes some value `A` in a context of `IEnumerable`, and a function `selector` that goes from `A` to a contextful `B` (in the same context of `IEnumerable`), and returns that contextful `B`.

`SelectMany` is an extension method, and, again, `LINQ` is designed in a way that its extension methods could be implemented for any other type, not only for `IEnumerable`. We can substitute `IEnumerable` with other contexts (such as `Task` or `Option`) and it will be the same pattern.  
[Here is](http://blogs.msdn.com/b/pfxteam/archive/2013/04/03/tasks-monads-and-linq.aspx) an example from Microsoft on how to use LINQ with `Task<T>`.

Look, LINQ is a **Monad**! :)  
And it is not just my imagination, or something that I wish it to be. It has beed carefully designed and implemented this way. Erik Meijer, the "father" of LINQ highlighted this fact many times, it is not hard to google. Or just read his paper [The World According To LINQ](http://delivery.acm.org/10.1145/2030000/2024658/p60-meijer.pdf) if you really want to dig into details.  

The fact is: monads are nothing to be scared of. We do have them in C#, and we do love them. It is hard to imagine how we survived without the LINQ monad before it became available in our toolbox. It is just a term that people are usually scared of, not the concept. And because of this irrational fear some languages call monads "workflows" or "burritos" or "fluffy clouds", etc.

Ok, that's clear now. But what are our expectations of `bind` (or `.SelectMany`) to perform safe and predictable?  

**First**, we should be able to put a value into a context (monad) without changing the value itself. For example, `new List<int> { 42 }` puts `42` in a context of a list, but the value itself remains the same and doesn't become `41` or `43` (it would be very strange otherwise).  

**Second**, it's logical to assume that if we don't do anything with the value within the context then nothing changes:

{% highlight csharp %}
var newList = oldList.SelectMany(x => new [] { x });
{% endhighlight %}

Here we expect `newList` and `oldList` to be identical. It is a very straightforward assumption because we haven't done anything, just put `x` back into the same context again!

**Third**, we assume associativity:  
(f `bind` g) `bind` h == f `bind` (g `bind` h)

{% highlight csharp %}
var allRoles = GetCompanies()
  .SelectMany(c => GetUsers(c))
  .SelectMany(u => GetRoles(u))

//is the same as
var allUsers = GetCompanies().SelectMany(c => GetUsers(c))
var allRoles = allUsers.SelectMany(u => GetRoles(u))

//is the same as
var allRoles = GetCompanies()
  .SelecyMany(c => GetUsers(c).SelectMany(u => GetRoles(u)))
{% endhighlight %}

Once more, nothing new here, we truly expect these "rules" to hold during our day-to-day refactoring, otherwise it would be just crazy.

These three rules above represent **Monadic Laws**. To be a monad a context must provide a `bind` (`SelectMany`) operation and it must obey these very logical laws.

### But why?..
Scary words such as **monad** or **functor** are patterns. Just as other patterns they represent some functional aspects and capture them in a language.  
We say "use a factory", or "make a singleton", or "have a façade", or "implement it as MVC" and our colleagues immediately understand what it means. In the same way we could say "make it a functor" or "use a monad".

Recognising patterns also helps to unify approaches when dealing with similar concepts. It is true for patterns in general, but it is even more important for a "monad" pattern because C# (and some other languages) has a special support, a special syntax for monads.   

{% highlight csharp %}
Context<D> result = from a in Context<A>
                    from b in Context<B>
                    from c in Context<C>
                    select CombineABC(a, b, c);
{% endhighlight %}

This code is very readable and very understandable even if we don't know what the `Context` is. Do we really care? It could be `Task<T>`, or `List<T>`, or `Option<T>`, or `Producer<T>`, or `Observable<T>`, etc. But the code reads the same: use three contextful values and apply some function to them.  

This C# syntax is designed to be extensible, it is not only for collections as some people may think. All you need is to identify a context and provide it with the extension methods such as `Select` and `SelectMany` keeping in mind the simple and logical laws that were mentioned above.

The syntax above is called "**from comprehension**" or "**monadic comprehension**".

To me this fancy syntax feels less awkward compare to `map` and `bind` renamed to `Select` and `SelectMany`. Because, disregarding "SQL-ish" flavour, it still makes lots of sense: `from value 'a' in this context, and from value 'b' in that context select their product`. In fact it is much less "SQL-ish" and much less "collection-oriented" than the `Select` and `SelectMany` counterparts.  

Unfortunately C# doesn't allow to go far beyond that in unifying approaches when dealing with different contexts due to its type system limitation (I just hope that some day it will). But this alone is a very convenient and a very useful feature.

### Conclusion
Being afraid of terms is illogical, especially if these terms represent something very familiar, something that we comfortably use every day.  
**Functor** and **Monad** are these terms that people are often scared of.  

In this article I tried to explain that **Functor** is just something that can be mapped over. It is just some hypothetical `IMappable` which is easy to imagine and which is not really needed because C# has extension methods. With extension methods we can add the `map` operation to virtually anything, and that's what `Select` method does. Of course is important to remember the simple laws **Functor** must obey when implementing `Select` method, but these laws are very logical and anyone would expect them to hold anyway.

A **Monad** is a context that has `SelectMany` implementation (and obeys simple laws, similar to **Functor**). Monads are not that hard to understand and not difficult to deal with. LINQ for collections is an example of a monad that we use every day we write code in C#.  
It is all about composition. How do we compose `a -> List b` and `b -> List c`? Or, more generally, `a -> M b` and `b -> M c`? A **Monad** is the answer.  

C# has a special syntax for monads that works for **any** monad, not only for `IEnumerable` and that is designed to be this way. This syntax may initially look collection-oriented, but in fact it makes lots of sense when dealing with different contexts too. 

Have a nice day. Use [http://goo.gl/mLGwTL](http://goo.gl/mLGwTL)
