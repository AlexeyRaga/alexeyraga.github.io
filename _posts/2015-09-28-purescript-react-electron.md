---
title: PureScript + React + Electron
layout: post
tags: [PureScript]
---

A while ago I stumbled upon [Electron](http://electron.atom.io/) - a shell/framework developed by GitHub for writing cross-platform desktop applications using modern web technologies.
You have probably seen it if you use [Atom](http://atom.io) or [Slack](https://slack.com/) or [Visual Studio Code](https://code.visualstudio.com/) or [Avocode](http://avocode.com/) or [Kitematic](https://kitematic.com/), etc. If not then I recommend you to check out at least these links.

I wanted to play with it, and the "modern technologies" now almost inevitably include [React](https://goo.gl/o8zXjK). I have heard lots of positive things about React having countless conversations with my colleagues, so building an `Electron` solution using `React` and learning both things together seemed like a logical choice.

There was one inconvenience however: JavaScript. I don't like working with this language, just as many other people who use TypeScript, CoffeScript, Elm, Dart, ScalaJS, etc. Just as JavaScript developers themselves who use Babel to compile JavaScript to JavaScript because writing in "just" JavaScript is unpleasant.  

### Why PureScript?
I decided to choose [PureScript](http://goo.gl/rztpWR). In short, the reasons I made this decision were:

- PureScript is a statically typed language. This is great for refactoring and maintainability. It is also a "must have" feature for me personally because I don't believe in a "dynamic languages" myth.
- PureScript has a great type inference which is a huge win over, say, TypeScript. Seriously, it is **so** good at types so it is almost hard to believe.
- PureScript has a powerful, but nice and concise syntax. This is again a huge win over TypeScript which is very noisy and much less expressive.
- PureScript is a functional language. Which means lots of great things including immutability, ADT, etc.
- PureScript has [higher-kinded polymorphism](http://goo.gl/NHulQF). That says last "goodbye" to TypeScript and eliminates languages like Elm.
- PureScript has a relatively small (and readable) footprint after compiling to JavaScript. This may not be very relevant for building desktop applications with Electron, but if you want to use the same code for the browser version of your product then it is good to have.
- PureScript has a very good way to communicate with a "native" JavaScript and to still stay safe.
- PureScript has a baked-in modules system.
- PureScript **doesn't have `null`**!! :)

I also like how this [Little Pony](https://goo.gl/jzTc2R) game ([click to play online](http://goo.gl/QlRGSL)) is implemented in PureScript in just 150 lines of code.

### The problem
Although there is a [free book about PureScript](https://goo.gl/DSxJZF) and a good [documentation search engine](http://pursuit.purescript.org/), and a helpful community, my biggest problem was to find good examples that I could use as a starting point.  
Maybe it was partly because I was learning not only PureScript, but also React and Electron, but my impression was mirroring this joke:

> 1 + 2 = 3. Here is how you do addition. 2 * 2 = 4. Here is how you do multiplication. Now, when you understand math, you can easily prove that this differential equation has only one solution...

Something like that :)

There are countless PureScript+React examples of how to increment a counter by clicking a button. And there are more complicated projects, which I found hard to learn from being a PureScript newcomer with zero experience in React.

### The example
So I decided to build one :) The full source code can be found [on GitHub](https://goo.gl/hf73sp) and below I explain how it works.

As I said, I wanted to build something working but yet short (~100 lines of code) and simple. It is a React application, so it should show how to create react components in PureScript and how to make them interact with each other.

I ended up building a very simple stock viewer:

![PureScript-React-Electron Example]({{ site.url }}/assets/PS-React-Example.png)

It consists of a number of components, such as:

- Stock list (left pane)
 - Stock list item (an element of a list is a separate component)
- Selected stock item pane (right pane)
  - Selected stock item header
  - Selected stock item info (the one displaying a chart)

As you can guess, clicking on a list element updates the header and and the graph. More components and behaviors should be easy to plug in when needed.

The data model is declared in `Stock.purs` and is simply a `record` type with 3 fields:

{% highlight haskell %}
type Stock = { symbol :: String, name :: String, sector :: String }
{% endhighlight %}

Then I have a (hardcoded) list of SP500 stocks of this type that I call `sp500`. This is all the model that I need.

Creating a React component in PureScript (using [purescript-react](https://goo.gl/zqZhoi) bindings) looks like:

{% highlight haskell %}
helloWorldComponent = createClass $ spec initState \ctx -> do
  props <- getProps ctx    --not used, just for demo
  state <- readState ctx   --not used, just for demo
  return $ D.span [ P.className "my-span" ] [ D.text "Hello World" ]
{% endhighlight %}

Here I declare a component called `helloWorldComponent` by using React's `createClass` function and providing it with a "specification". The specification is created by the `spec` function that takes two arguments: an init state and a "render" function (a lambda function in my case). The "render" function takes some React context and produces some elements to be displayed on the screen.  

Within that "render" function I have access to the component's `props` through the `getProps` function, and to its state through the `readState`, `writeState` and `transformState` functions.

Well, creating a simple component is easy, and since I don't need to access `props` or `state` in my `helloWorldComponent` it can even be 1 or 2 lines of code. But what about more "real" example?

Let's look at the `stockPage` component that renders what is on a screenshot above.   
It uses a state to represent a currently selected stock:

{% highlight haskell %}
type PageState = { currentStock :: Stock }
initState = { currentStock: head sp500 }
{% endhighlight %}

The type `PageState` is a record that only contain currently selected stock, but can be expanded to hold more data if needed.  
The `initState` is a value of type `PageState` that is the first element of my `sp500` list to start with.

The `stockPage` component (declared in the `Page.purs` file) looks like:

{% highlight haskell %}
{- Stock Page -}
stockPage =
  let updateCurrent stock state = state { currentStock = stock }
      onStockSelected ctx stock = transformState ctx (updateCurrent stock)
  in createClass $ spec initState \ctx -> do
     state <- readState ctx
     return $ D.div [ P.className "content" ]
                    [ createFactory stockListBox {selectStock: onStockSelected ctx}
                    , D.div [ P.className "main-pane" ]
                            [ createFactory stockHeader state.currentStock
                            , createFactory stockInfo state.currentStock
                            ]
                    ]
{% endhighlight %}

Here I first declare two "helper" functions `updateCurrent` and `onStockSelected` that I want to use in my component. I could declare them outside of `stockPage`, but I prefer doing it this way because they are only needed within the component.  
The `updateCurrent` function creates a new state by copying the given one and replacing its `currentStock` property.  
The `onStockSelected` is a callback I want to use when user selects a stock. I uses `transformState` (provided by `purescript-react`) to update the state within `ctx` by applying `updateCurrent` to it.

Then I create the component by using `createClass` function and providing the `initState`, just like in the "hello world" example.

Within its "render" function I use React's `createFactory` to create my other components such as `stockListBox`, `stockHeader` and `stockInfo`.

The `createFactory` takes two parameters: a component to create and its props.  

For `stockHeader` and `stockInfo` components I use the `currentStock` as their props because that's what they need to display.

Rendering `stockListBox` is different: it doesn't need to know about the `currentStock`, but it needs to know what to do when user clicks an item. I want it to use my `onStockSelected` callback so as props I pass it a `record` that holds this callback. The `stockListBox` component then can access it via `props.selectStock` (see `Page.purs` to see how).

Now it is clear how it works: when a list item is clicked the `onStockSelected` is called, and it will update the current `stockPage`'s state. And React is smart enough to understand that there are dependencies on the state and to re-render affected parts (which are `stockHeader` and `stockInfo` components).

Notice one thing in the example above, the line

{% highlight haskell %}
createFactory stockListBox {selectStock: onStockSelected ctx}
{% endhighlight %}

the `props` that I pass to the `createFactory` function is a record that I haven't declared the type for. In this case PureScript will still infer the type and will still check if it matches with something that is expected. And if it is not, say, if `stockListBox` expects its props to be of a different shape, it will be a compiler error. Isn't it amazing?!

The `stockPage` component is about 10 lines of code and it is actually the most complicated part of the whole example. The rest of the components are much simpler since they are stateless and most of them don't even have any logic. You can have a look at the project [on GitHub](https://goo.gl/hf73sp) and run it yourself in Electron or in a browser.

### Conclusion
In contrast to TypeScript, I found the overall experience of writing PureScript very pleasant. It wouldn't let me do wrong things (which is one of the main points of having a type system), and it was very helpful because I didn't really know the way things are supposed to be done with React. It took me a couple of hours to figure out how things supposed to work in PureScript though, perhaps reading [the book](https://goo.gl/DSxJZF) would be helpful, but I think any new language requires some initial investment of time.

I found React very impressive. Being a relatively small library it very well facilitates the way of structuring the code in a nice decoupled way. The approach of getting rid of templates and defining components layouts just in code seemed a bit strange at the beginning. But after giving it a thought and playing with it I tend to thing that this approach is right and that it simplifies things a lot, at the same time making the code more maintainable.

I found it useful that I can use the same code for both desktop (using Electron) and online-based (browser) solutions. In the end of the day, Electron is just Chromium repackaged without some browser restrictions. Being a desktop application it has the access to the file system, can do cross-domain ajax calls, etc.  
It is not important in this example project, so it works in both browser and Electron. But it could be helpful if I wanted to access some 3rd party web API (say, https://www.google.com/finance) to get more data, because cross-domain ajax requests most likely would not be allowed by the browser.
