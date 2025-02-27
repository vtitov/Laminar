---
title: Virtual DOM is a Hack
---

First of all, I should say that I very much appreciate all the libraries mentioned here. But I do need to explain why I think Laminar's approach is superior, so I will be making comparisons from my point of view. A lot of this is subjective. The problem at hand is too complicated for there to be the one unquestionable solution.


## Virtual DOM vs FRP

Facebook needed a performant, reactive way to build web applications, and so React.js popularized virtual DOM because (in part) type-safe functional reactive programming was not a viable option for their target audience.

Despite its wide popularity, there is nothing special about virtual DOM, it is merely a solution to a specific problem, and not a great one – it's [pretty](https://reactjs.org/blog/2015/12/18/react-components-elements-and-instances.html) [complicated](https://reactjs.org/docs/reconciliation.html), and its performance is limited by the often-significant amount of work the framework needs to perform when diffing virtual element trees.

All of that diffing machinery is required because virtual DOM elements / subtrees have no continuity – they're ephemeral. Every time a React component's state or props change, it will re-run its `render` method (which often means doing the same recursively for its child components) to generate a new virtual subtree that is completely separate from the previous subtree generated by this same component.

All of this only gets worse when you put wrappers around native javascript libraries. ScalaJS-React for example is not just a bunch of thin `@js.native` interfaces, it brings in its own concepts and runtime code.

Laminar on the other hand does not use virtual DOM diffing. Instead, it uses event streams and reactive state variables for **precise DOM updates**. So when your Laminar code says it needs an element's color property to be updated, that's exactly what will happen, immediately, directly, and nothing else. For comparison, with virtual DOM a bunch of render methods would have been run, new virtual elements would have been created, the framework would have generated the diff of the new and old subtrees of virtual elements, and finally translated this diff into an actual one-line DOM update that we actually care about.

Laminar does maintain its own tree of `ReactiveElement`-s, but it's only used to track parent-child relationships between elements, not their attributes and properties. We never need to diff virtual elements. Also, unlike ephemeral virtual DOM elements, the lifespan of Laminar's reactive elements matches the lifespan of corresponding real DOM elements, so you get stable references that you can easily work with. 

I personally see virtual DOM as a very elaborate way to avoid using actual reactive data structures like streams. And virtual DOM is not useful even if you do use FRP. Early versions of Laminar did actually use virtual DOM (Snabbdom), but I ran away from that when I realized how counter-productive the combination of those paradigms is, both to simplicity and performance.


## Virtual DOM _with_ FRP

> I've provided a more in depth explanation for why Virtual DOM and FRP don't mix in this blog post: [My Four Year Quest For Perfect Scala.js UI Development](https://dev.to/raquo/my-four-year-quest-for-perfect-scala-js-ui-development-b9a)

Some other libraries like Outwatch (or Cycle.js in the JS world) use both FRP and virtual DOM as a way to achieve complete functional purity. To me this concept is largely lost for frontend development. I don't think the effort, conceptual complexity, and performance penalties of wrapping everything in effect types and virtual elements is worth the marginal safety that some of this could provide. The entire frontend application is _supposed_ to be a collection of DOM and network IO effects, there is barely anything else to it. Like any other technique, pure functional programming is only good for the benefits that it provides, and I don't think it provides a net benefit in frontend development when you consider all the compromises you have to make for it, and all of their effects (ha). This is, of course, a matter of preference, and so both Outwatch and Laminar exist.

_A bit more on Cycle.js [in our Gitter](https://gitter.im/Laminar_/Lobby?at=5b749e5c5b07ae730ac330c4)._

As mentioned above, I did previously use virtual DOM in Laminar, and this experience convinced me that reactive UI development is much, much simpler with FRP alone, without virtual DOM.

Outwatch and Laminar are quite similar cosmetically – their arrows notation was just too good not to steal – but they're vastly different under the hood. If you're curious, you can dive into sources to compare how a simple expression such as `div(widthAttr <-- widthStream)` is handled. See what exactly happens when a new event is sent to `widthStream`. Ignore the internals of the streaming libraries (RxJS and Airstream) which call `Observer.onNext` (as they'll be quite similar), but do look at the rest of the code path otherwise. Here it is for Laminar v0.8:

1) Go to definition of this `<--` method, it's in `ReactiveHtmlAttr` because that's what `widthAttr` is:

```scala
  def <--($value: Observable[V]): Binder[HtmlElement] = {
    Binder { element =>
      ReactiveElement.bindFn(element, $value) { value =>
        DomApi.setHtmlAttribute(element, this, value)
      }
    }
  }
```

2) Putting our reactive boilerplate aside for a moment, you see that we call `DomApi.setHtmlAttribute` when `widthStream` (i.e. `$value` here) emits a new `value`. Going to its definition we see:

```scala
  def setHtmlAttribute[V](element: ReactiveHtmlElement.Base, attr: HtmlAttr[V], value: V): Unit = {
    val domValue = attr.codec.encode(value)
    if (domValue == null) { // End users should use `removeAttribute` instead. This is to support boolean attributes.
      removeHtmlAttribute(element, attr)
    } else {
      element.ref.setAttribute(attr.name, domValue.toString)
    }
  }
```

That `element.ref.setAttribute(...)` is a call to native Javascript DOM API. There is no library code behind it. You can see for yourself how the same code path goes in Outwatch. It's quite a bit more involved, especially if you look into Snabbdom. I won't even ask you to try this with React.

I guess I skipped over how `Binder` works. On a high level, when the element is mounted, the Binder subscribes the provided callback to fire every time the `$value` observable emits a new value, and kills that subscription when the element is unmounted. You can of course follow the `Binder.apply` method and `ReactiveElement.bindFn` to see how this works, but that's going very deep into the innards our reactive system, and you'll want to read our docs. We do explain how everything works in great detail.
