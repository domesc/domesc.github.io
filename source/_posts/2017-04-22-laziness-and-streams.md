---
layout: post
title: "Laziness and Streams"
permalink: laziness-and-streams
published: true
date: 2017-04-22 17:19:20
comments: true
description: "Laziness and Streams"
keywords: "scala"
categories: scala

tags:

---
In the real world laziness is the feeling, some people have, of doing the least work or effort possible. In functional programming ... well if we think about it, it's kind of the same concept.
Indeed, laziness consists in avoiding any computation that is not needed straight away, but deferring it to a later stage.

One can easily ask: why should I make my life difficult? It turns out that sometime applying this concept to your program can be quite useful.
In the following paragraphs I will try to go step by step from the basis to some applications of laziness like Streams. Every example in this post and probably in every future post will be written in Scala, a functional programming language running on the JVM, which I use in my current job and which I find simply awesome.

So let's start!
<br/>

Deferred evaluation and lazy keyword
------------------------------------

In Scala there are two ways of passing parameters to a function, one is by value the other by name. Understanding the difference between them is the basis so let's see some examples:

{% gist domesc/e8aee98f315d406d5d88114d6b54514b %}

The code above shows a quite important feature of Scala, the possibility to pass a parameter by name through writing the little arrow before the type `x: => Double`. This special syntax will tell Scala compiler to not evaluate immediately the expression passed to `callByName` function. Note instead that in `callByValue` the parameter has no special syntax, which means it is passed by value like it happens by default in Java for example.

Now that Scala syntax is clear let's try to call in the REPL `callByValue` and `callByName`:

{% highlight scala %}
scala> callByValue(square(2))
The square is 4.0
Square by value is  4.0
Square by value is 4.0

scala> callByName(square(2))
The square is 4.0
Square by name is 1 4.0
The square is 4.0
Square by name is 2 4.0
{% endhighlight %}

The two functions have the same body but the REPL gives different results. There are two main reasons why this happen:

1. the body of both functions contain `println` calls, which create side-effects killing the purity of the function (referential transparency property is broken).
2. in `callByValue` the `square` function is evaluated before everything and the evaluated parameter is passed to the body of the function. On the other hand in `callByName` the evaluation is deferred to the point when the parameter is used and every time re-evaluated.

The first reason is not the subject of this post so I will not spend time on that, but I strongly recommend you to look for *referential transparency*, if you don't know already what it means. Here we are interested more in the second reason, deferring the parameter until it is used is the definition of laziness!
There is a problem though, the parameter is re-evaluated every time, and in case we are passing a function by name, which takes up a lot of time to evaluate we will repeat the computation twice eating up CPU.

Let's see how we can modify `callByName` to avoid the aforementioned problem:

{% gist domesc/8ddf86261fcc08311031d1fc233348e4 %}

What I did here was to assign the parameter reference to a `lazy val`. `lazy` keyword is another awesome feature of Scala (missing for example in Java). It still let us deferring the computation, but the first time the evaluation is required the result is cached, so that the computed value can be reused and no further evaluation is needed. The `lazy` keyword is a great tool, but it has to be used with some care. The issues that you can incur are described in [SIP-20](http://docs.scala-lang.org/sips/pending/improved-lazy-val-initialization.html).

Let's now come back to our new `callByName` implementation and see if I'm saying the truth on the REPL:

{% highlight scala %}
scala> callByName(square(2))
The square is 4.0
Square by name is 4.0
Square by name is 4.0
{% endhighlight %}

Clearly this time the evaluation of `square(2)` happened only once and that's because of the `lazy val` construct.

These were all the bits we needed to understand and implement laziness in Scala, next step now is to see a powerful application of it: Streams.

Streams
-------

Probably every developer had to deal in his career with some sort of collection like lists or arrays. We build a collection of elements, which take space in memory and once the allocation is done we can start making operations on it, like iterating, adding an element etc. But what if we don't want to load all the elements in memory from the beginning? What if we want to store only the type of computations we want to make on the collection without actually applying them straight away?
That is actually what a `Stream` does.

A `Stream` can be thought as a lazy `List` where the head is evaluated while the tail is not. That can be easily seen in the following code snippet:

{% gist domesc/c92c21206ca38175cc3157abaf70def1 %}

The above code has been extracted from Scala 2.12 repository, it is probably the simplest way to show how the `Stream` constructor take the `head` by value and the `tail` by name. Let's prove everything in the REPL:

{% highlight scala %}
scala> val list = List(1, 2, 3)
list: List[Int] = List(1, 2, 3)
{% endhighlight %}

in a `List` every element is loaded and stored into the collection.
On the other hand if we create a Stream of elements with `Stream.cons`:

{% highlight scala %}
scala> val stream = Stream.cons(1, Stream.cons(2 , Stream.cons(3, Stream.empty)))
stream: Stream.Cons[Int] = Stream(1, ?)
{% endhighlight %}

there is now a question mark instead of the list of elements we passed to the constructor and this proves the evaluation of the tail is deferred. By the way don't worry there is a more syntactic sugar way to create a `Stream`:

{% highlight scala %}
scala> val stream = Stream(1, 2, 3)
stream: scala.collection.immutable.Stream[Int] = Stream(1, ?)
{% endhighlight %}

We have now our new `stream` object, what about accessing the third element for example:

{% highlight scala %}
scala> stream(2)
res15: Int = 3

scala> stream
res16: scala.collection.immutable.Stream[Int] = Stream(1, 2, 3, ?)
{% endhighlight %}

Did you notice anything? When we accessed the 3rd element Scala evaluated everything until the third index included.

Streams are also quite flexible since we can do the same kind of operations supported on normal lists like map, filter etc. but in a lazy way:

{% highlight scala %}
scala> val doubled = s.map(e => e * 2)
doubled: scala.collection.immutable.Stream[Int] = Stream(2, ?)

scala> doubled(2)
res18: Int = 6
{% endhighlight %}

But we can do even more and create an infinite `Stream`.
<br/>

Infinite Streams
----------------

The way Scala allows the creation of an infinite `Stream` is quite straight-forward:

{% highlight scala %}
scala> val infiniteStream = Stream.from(1, 1)
infiniteStream: scala.collection.immutable.Stream[Int] = Stream(1, ?)
{% endhighlight %}

The code above does nothing but creating a `Stream` of integers starting from 1 and increasing by 1. To better
understand how this is built let's look at the code !

Here I paste the implementation of `from` method in the `Stream` class Scala version 2.12:

{% gist domesc/fca6365fb4730627e6793749f0ac8cf6 %}

This snippet calls the constructor I showed you before passing the initial value as head of the `Stream` and the function itself as tail. If you try to access any index of an infinite `Stream` it will always work (or almost always since computer resources are not infinite :D).

Conclusion
----------

To summarize a `Stream` has three main functional properties:

* *lazy*: as we explained and as we can see from the `Stream` constructor, the tail of the collection is passed by name (so the evaluation is deferred).
* *recursive*: the infinite `Stream` is defined in terms of itself.
* *infinite*: which is possible thanks to the lazy property.

We saw how laziness and streams work in Scala, but this doesn't include all the possible
uses and APIs examples, though it gives you an idea on how things works. For more information
on the `Stream` APIs this is the link to the [Scala Doc](http://www.scala-lang.org/api/2.10.2/#scala.collection.immutable.Stream).
