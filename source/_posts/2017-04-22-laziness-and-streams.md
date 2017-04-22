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
This is the first post of my blog dedicated mostly to programming and maybe some machine learning. As first subject I decided to write about laziness, I'm not speaking of course about the feeling some people have of doing the least work or effort possible, but what laziness means in functional programming, which if we think about, it is kind of the same concept. Indeed laziness in programming consists on avoiding any computation that is not needed right now, but deferring those computations in a later stage.
Why doing this? Well one of the reason could be performance for example, why using potentially a lot of memory and computational resources for computing something that is not needed yet?

In this post I will show how you can exploit laziness using Scala, a functional programming language running on the JVM.

So, let's start!  
<br/>

Deferred evaluation and lazy keyword
------------------------------------

In Scala there are two ways of passing parameters to a function, one is by value the other by name. Let's see the difference through some examples:

{% highlight scala %}
def square(n: Double): Double = {
  val s = n*n
  println("The square is "+s)
  s
}

def callByValue(x: Double): Unit = {
  println(“Square by value is ” + x)
  println(“Square by value is ” + x)
}

def callByName(x: => Double): Unit = {
  println(“Square by name is ” + x)
  println(“Square by name is ” + x)
}
{% endhighlight %}

When we call `callByValue` and `callByName` functions we see two different results in the REPL
despite the fact that the body of both does exactly the same thing:

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

In `callByValue` the `square` function is evaluated before everything and the evaluated parameter is passed to the body of the function. On the other hand in `callByName` the evaluation is deferred to the point when the parameter is used and every time re-evaluated.
This shows how in Scala we can defer the evaluation of a function parameter and make it happen later. Wait ... that's awesome, but why re-evaluating every time the parameter? In case we are passing by name a function which takes up a lot of time to evaluate we will repeat the computation twice eating up CPU. The solution is to use the `lazy` keyword, let's see how we can modify `callByName`:

{% highlight scala %}
def callByName(x: => Double): Unit = {
  lazy val square = x
  println(“Square by name is ” + square)
  println(“Square by name is ” + square)
}
{% endhighlight %}

This time if we call the function:

{% highlight scala %}
scala> callByName(square(2))
The square is 4.0
Square by name is 4.0
Square by name is 4.0
{% endhighlight %}

Clearly this time the evaluation of `square(2)` happened only once and that's because the `lazy val` construct enabled us to defer the computation, but at the same time caching the result the first time the evaluation is required. In this way the cached value can be reused and no further evaluation is needed.

Now that laziness concept is hopefully clear and we saw how this is handled in Scala, we can see another powerful tool which is built upon laziness: the Streams.

Streams
-------

Since the first year of study in computer science every student has to deal with some sort of collection like lists or arrays. We build a collection of elements which take space in memory and once the allocation is done we can start making operations on the collection like iterating, adding an element etc. But what if we don't want to load all the elements in memory from the beginning? What if we want only to store the type of computations we want to make on the collection without actually applying them straight away?
Welcome to the `Stream` world. Let's see the difference between a non lazy collection like List and Stream:

{% highlight scala %}
scala> val list = List(1,2,3,4,5)
list: List[Int] = List(1, 2, 3, 4, 5)
{% endhighlight %}

We can see that when we create a List all the elements are loaded and stored in the collection.
Let’s see what happens if we create a Stream of elements:

{% highlight scala %}
scala> val stream = Stream(1,2,3,4,5)
stream: scala.collection.immutable.Stream[Int] = Stream(1, ?)
{% endhighlight %}

We have now a question mark instead of the list of elements we passed to the constructor. This is because `Stream` is lazy version of a List and the evaluation of the elements is deferred. An example which proves this:

{% highlight scala %}
scala> stream(2)
res15: Int = 3

scala> stream
res16: scala.collection.immutable.Stream[Int] = Stream(1, 2, 3, ?)
{% endhighlight %}

Did you notice anything? When I accessed the 3rd element of the `stream` variable Scala evaluated all the elements until the third index included.
That's awesome, we can make any kind of operation supported on lists like map, filter etc. but in a lazy way:

{% highlight scala %}
scala> val doubled = s.map(e => e * 2)
doubled: scala.collection.immutable.Stream[Int] = Stream(2, ?)

scala> doubled(2)
res18: Int = 6
{% endhighlight %}

At this point a valid question can be raised, since the `Stream` doesn't evaluate all the elements of the collection
until it is not required, what stop us from creating an infinite `Stream`?
<br/>

Infinite Streams
----------------

The way Scala allows the creation of an infinite `Stream` is quite straight-forward:

{% highlight scala %}
scala> val infiniteStream = Stream.from(1)
infiniteStream: scala.collection.immutable.Stream[Int] = Stream(1, ?)
{% endhighlight %}

The code above does nothing but creating a `Stream` of integers starting from 1 and increasing by 1. To better
understand how an infinite `Stream` is built the best way is to look at the code !!
Here I paste the implementation of `from` method from the `Stream` class in Scala version 2.12:

{% highlight scala %}
def from(start: Int, step: Int): Stream[Int] = cons(start, from(start+step, step))
{% endhighlight %}

and this is the `cons` method (which stands for constructor by the way):

{% highlight scala %}
object cons {
    def apply[A](hd: A, tl: => Stream[A]) = new Cons(hd, tl)

    def unapply[A](xs: Stream[A]): Option[(A, Stream[A])] = #::.unapply(xs)
}
{% endhighlight %}

From the above code snippets we can extract some of the functional properties of an infinite `Stream`:

* *lazy*: as we explained until now and as we can see from the `Stream` constructor, the tail of the collection is passed by name (so the evaluation is deferred).
* *recursive*: the infinite `Stream` is defined in terms of itself, just look at the `from` method above.
* *infinite*: of course, which is possible thanks to the lazy property.


In conclusion we saw how laziness and streams works in Scala, the post of course doesn't include all the possible
uses and APIs examples, but it gives you a wide idea on how things works. For more information
on the `Stream` APIs this is the link to the [Scala Doc](http://www.scala-lang.org/api/2.10.2/#scala.collection.immutable.Stream).
