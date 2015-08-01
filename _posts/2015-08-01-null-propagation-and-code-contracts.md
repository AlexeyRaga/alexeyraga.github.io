---
title: On null propagation and code contracts
layout: post
---

I am a software developer with 11 years of experience in C#. After that I moved on to F#, and then to Scala, and a bit of Haskell, but it doesn’t matter much except for the fact that I can see things from different perspectives. I still try to keep in touch with the .NET land, particularly with C# and F# because, well, it is still very familiar and is very interesting to me.

Recently I had an interesting conversation, almost an argument, with my former colleague about two relatively new C# features: null propagation and code contracts. In this article I will try to explain my point of view on these features and their impact on developers.

### Null Propagation.

In our developers life we all dealt with `NullReferenceException`s. Perhaps way too many times. This is probably one of these nasty exceptions that hard to predict, handle and compensate for.

Sir Antony Hoare, speaking at QCon in 2009, apologised publicly for inventing null reference:

>I call it my billion-dollar mistake. It was the invention of the null reference in 1965. <…> I couldn't resist the temptation to put in a null reference, simply because it was so easy to implement. This has led to innumerable errors, vulnerabilities, and system crashes, which have probably caused a billion dollars of pain and damage in the last forty years."

Yet we are still dealing with null references, still paying the price. Some of us, developers, still deliberately return `null` from functions with the excuse that “null means that there is no value”, but let’s keep it for later.

What is the problem with `null` anyway? The problem is that `null` is indistinguishable from a valid value. If there is “no value” then you can still use it was there, and compiler allows you to do it. Then you get `NullReferenceException`.

It gets much nastier when there are functions in the call chain implemented in a way of “Oh, I checked my input and I got `null`, but I don’t know what to do with `null` so I just return `null` myself”. Now when we get `NullReferenceException`, we don’t know who was responsible and what just happened.

What is "null propagation"? It is a way to deal with `null` values safely. Some languages, like Groovy, and now C# have a special syntax for it:

{% highlight csharp %}
var minPrice = product?.PriceBreaks?[0]?.Price;
{% endhighlight %}

This code means that on every step where null propagation operator ("?") is used the compiler is going to automatically insert a null check. And if the value happens to be `null` then the computation is aborted and the whole result becomes `null`. There is actually some weirdness happening around value types, but it is not important from our conceptual perspective. What we get is a short and concise syntax for dealing with nulls.

Does it look nicer than having manual checks? Hell yes!
Does it solve the problem? Hell no!

Having the "ghost of null” around, you typically cannot trust anything. When you call a function, or a method, or a property, how do you know that it doesn’t give you `null` at some point? You never know. And the compiler doesn’t help you here. And even if you go and check the code of the function you call, understand it and ensure that it will not give you `null`, you cannot guarantee that this function is not going to be changed tomorrow. And you will never know about it until you get NullReferenceException.

So do you want to start polluting your code with the null propagation operator everywhere, just in case? Because that would be a solution :) Although, wait, a limited and an ugly one.

Can we do better? Of course.
Unlike Groovy, C# is a statically typed language. So use types! This solution is not new, it exists in other statically typed languages, including F# and now even Java.
Just as all these good principles of “separation of concerns” and “single responsibility” teach us, let’s capture the concern of non-existent (optional) values into a type: `Option<T>`

The `Option<T>` is just a container that can either contain some value of type `T`, or not. If it is hard to understand then look at it as if it was a list that can either be empty or can contain exactly one element.

Now, if we want to express the fact that the return value may be absent, we can do it explicitly:

{% highlight csharp %}
public interface IUserRepository {
    public Option<User> GetByEmail(Email email);
}
{% endhighlight %}

The important part here is that now anyone who uses this function, including the compiler is aware of the fact that the value can be missing. More, because the GetByEmail method returns not `User`, but `Option<User>`, it is impossible to use it where `User` is expected and then have `NullReferenceException`! Because `User` and `Option<User>` are different types! We lifted the whole non-existence concept into a compiler’s problem, not ours! Isn’t it the whole point of having a type system?

Now, let’s see what we can do with this type and how we could use `GetByEmail` method that returns an optional value:

{% highlight csharp %}
public class ApplicationService {
    private IUserRepository _repository; //initialised somehow

    public Option<Message> PrepareEmail(SpamTemplate template, Email to) {
        return _repository.GetByEmail(to)
            .Where(user => user.IsSubscribedToSpam == true)
            .Select(user => user.FirstName + " " + user.LastName)
            .Select(fullName => template.SetRecipiend(fullName, to))
            .Select(templ => templ.Render());
            //or we could do it in a single select block if we wanted
    }

    public Option<Message> ConnectingPeople(ConnectTemplate template, Email sender, Email to) {
        return from sndr in _repository.GetByEmail(sender)
               from rcpt in _repository.GetByEmail(to)
               let sndrName = sndr.FirstName + " " + sndr.LastName
               let recptName = rcpt.FirstName + " " + rcpt.LastName
               where(rcpt.IsAcceptingConnections == true)
               select template.SetSender(sndrName, sender)
                   .SetRecipient(recptName, to)
                   .Render();
    }

    public Statistics GetStatistics(Email email) {
        return _repository.GetByEmail(email)
            .Select(user => CalculateStatistics(user))
            .GetOrElse(Statistics.Empty);
    }

    public Statistics CalculateStatistics(User user) { … }
}
{% endhighlight %}

Above are three simple examples of how `Option<T>` type can be used (it obviously have standard `LINQ` extension methods, like `SelectMany`, `Select`, `Where`, etc. defined.

First example, `PrepareEmail`, demonstrates how to get an optional value, filter it (so it can become non-existent it it doesn’t match criteria), transform the value and get it back (as an option). Not different at all from a single element list, isn’t it?

Second example, `ConnectingPeople`, demonstrates how to operate on different optional values, combine them together, and have a result back as a single option.

Third example, `GetStatistics`, shows how you could use “normal” functions like `CalculateStatistics` that have nothing to do with any non-existent concepts and optional values. This is important because it shows that optional values do not pollute the rest of your code. It is not possible to just pass the result of `GetByEmail` into `CalculateStatistics` because compiler wouldn’t let you do it (it knows that the value is optional, it knows the type). But it is still possible to use functions which expect just value, in a safe manner.

Working with `Option<T>` can be viewed as some “normal” computation running “within” the optional context, just like when you run computations within the Enumerable context, that’s how `LINQ` comes handy. When the value exists in the box, the computation run and you have a new value in that box. When the box is empty then, well, you just have an empty box.

Now, what about existing code, for example existing apis that may or will give you `null`?
You can still stay safe by wrapping it into an Option:

{% highlight csharp %}
Option<User> optionalUser = Option.of(oldBadRepository.GetById(userId));
{% endhighlight %}

or even

{% highlight csharp %}
Option<User> optionalUser = oldBadRepository.GetById(userId).ToOption();
{% endhighlight %}

One of these methods will check the returned value for `null` and will construct the Option type instance accordingly. That’s the only check for `null` you need to do, and you even don’t do it explicitly! Since then your code is safe and reasonable.

It is sad that given that C# is a statically typed language, Microsoft keeps introducing more and more language features around `null`, promotes (and propagates) `null` further and further instead of eliminating nulls by leveraging the type system. I would partially understand it for Groovy (since it is a dynamic language), but not for C#.
And, I repeat, even Java does better job in addressing this problem more properly.

Microsoft could just add an `Option<T>` class into BCL (that is what Java or F# do), but instead they decided to keep changing the language providing questionable solutions.  
However, since `Option<T>` is simply a type, anyone could spend some time and write 50-100 lines of code implementing it and all sorts of convenient methods around it.
And that’s what people do in this good open-source world: there are a number of nuget packages that are implementing `Option<T>` ready to be used. For example [https://github.com/nlkl/Optional](https://github.com/nlkl/Optional) looks nice and has lots of examples.

### Code contracts

Code contracts came from the “design by contract” idea. The essence of this idea is to give developers the ability to express the shape of function’s expected input and output to the outside world. Then whoever uses this function can clearly see what these expected inputs and outputs are. It would also allow the compiler and other static analysis tools to better reason about the code and to prevent different sorts of undesired behaviours.

In C# code contracts could be expressed as:

{% highlight csharp %}
public City GetCity(float latitude, float longitude) {
    Contract.Requires(latitude  >= -90.0  && latitude <= 90.0);
    Contract.Requires(longitude >= -180.0 && latitude <= 180.0);
    var city = ..... ;
    return city;
}
{% endhighlight %}

So now the compiler will have the (limited) ability to reason about input parameters. The method will throw an exception if someone tries to call it with “wrong” values in runtime and everything appears to be good.
The code contracts project spent years in research, required to implement an additional set of tools and another post-compilation step, crack some significant performance issues, and finally… It doesn’t solve the problem.

Developers are still required adding these contract checks everywhere, hoping they are consistent. For example, if I have two methods that deal with coordinates, I perhaps want to add the similar set of checks to that second method. The ability for compiler to reason about the code is very limited, too. In fact, C# compiler itself currently doesn’t care about contracts at all, all the heavy lifting is deferred to some potential “static analysis tools” that are supposed to come and analyse your code. Potentially.

And what happens when the contract is violated? Well, you have an exception.
"Fail fast”, you say? Was it fast enough when the wrong value already made its the way to my beautifully crafted method? I’d say it could be too late.

The problem here is not in contracts. The problem here is in types, to be precise in not using types.
Look at the `GetCity` method. It takes two `float` values as its parameters, and then immediately requires these values to be in some shape. So just being `float` wasn’t good enough since the beginning! __This__ is the problem, not how to check for some conditions and how to build a whole toolset around these checks!

Generally, when you write a function that takes a parameter of a certain type, in your mind try to add “for any” to the type. If the function still makes sense - go for it, but if not - it’s the wrong type!

In my example above, `GetCity(for any float, for any float)` doesn’t make sense. It was never meant to take any arbitrary `float` values. It meant to take latitude and longitude, which are much more precise. So give them types.

After that, the function looks like:

{% highlight csharp %}
public City GetCity(Latitude lat, Longitude lon) { … }
{% endhighlight %}

or, since these coordinates can point to somewhere in the middle of Pacific Ocean:

{% highlight csharp %}
public Option<City> GetCity(Latitude lat, Longitude lon) { … }
{% endhighlight %}


That’s what “design by contract” means. Your types are your contract. Types come first, they represent data that we deal with, with all the restrictions enforced. It would be impossible to create an incorrect value of `Latitude`, because it’s constructor will not allow it. It is __correct by construction__. If you try to construct it incorrectly, _then_ you will fail fast, _at the earliest stage possible_, without even being able to pass the wrong value anywhere downstream.

Also notice, that if you only look at the types in the function signature then you can see pretty much everything: a function takes a `Latitude` and a `Longitude` values and probably returns you a `City`. You don’t need to guess nulls, or valid ranges, or look how it is implemented, everything is just there in the signature. And it is checked by compiler.

On top of that you get the ability to compensate when the input values appear to be incorrect:

{% highlight csharp %}
//assuming that latitude and longitude are passed as command line parameters
public static void Main(string[] args) {
    var maybeCity = from lat in Latitude.OptionFrom(float.Parse(args[1]))
                    from lon in Longitude.OptionFrom(float.Parse(args[2]))
                    from cit in GetCity(lat, lon)
                    select cit.Name;

    Console.WriteLine(maybeCity.GetOrElse(“Atlantida"));
}
{% endhighlight %}

So “latitude" is not a `float` that is believed to be in between -90 and 90. And it has never been.
“Student Age" is not an `int` that is believed to be in between 0 and 100. And it has never been.
“Country code” is not a `string` that is believed to be in some dictionary somewhere. And it has never been.
“Australian postal code” is not an `int` that is believed to have 4 digits. And it has never been.

These are __different types__. Define your contract appropriately (types) and the “design by contract” will follow.
Because this is what you do literally: designing a function that takes a `Latitude`, and a `Longitude`, and produces a `City`.
And if I am not able to conform to this contract, then it is my problem, not yours, I will simply be unable to call your code if I don’t have correct values.

### Conclusion

I personally think that both "null propagation" and "code contracts" approaches are wrong and misleading. They discourage developers from learning and using better solutions and, in my opinion, only exist because someone in Microsoft has consciously decided to take the road that is more familiar to .NET developers rather to provide us with a solution. And they seem to continue making this decision, which is a bit sad.
All the time and money that are wasted on chasing `NullReferenceExceptions` and writing `null` checks, all the years of effort spent on implementing code contracts (half implementing, since this feature still didn’t make its way to the compiler) could perhaps better be spent on taking the language, and the type system further.

But, again, C# is a good statically typed language, so we can enjoy its benefits and can make it works for us. For example, we can free ourselves from nulls and can __actually__ design our code by contract.

Have a nice day. Use types.
