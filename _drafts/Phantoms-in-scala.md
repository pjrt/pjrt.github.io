---
title: Phantom types in Scala
tags: [scala, types, phantom, typeclass]
layout: post
---

Type variables are a powerful piece of type-level programming. They can appear
in a variety of forms and factors; phantoms and existential types being two
cases. In this article we will be exploring phantoms (lightly) and existential
(more detailed) types and some use cases for them. We will be going over
each conceptually, using a simpler syntax that borrows from `type`, and then
practically, using real Scala code.

# Type variables

Normally, this is the form in which we see type variables:

{% highlight scala %}
type F[A] = SomeClass[A]
{% endhighlight %}

`A` is said to be a "type variable". A property of `F` is that it is fully
dependent on the type of the data for `SomeClass`. For example:

{% highlight scala %}
SomeClass("hello"): F[String]
SomeClass(1: Int): F[Int]
SomeClass(true): F[Boolean]
{% endhighlight %}

So this means that we can't just, for example, do the following:

{% highlight scala %}
SomeClass("hello"): F[Int]
{% endhighlight %}

Now lets expand this into something more concrete:

{% highlight scala %}
sealed trait F[A]
case class SomeClass[A](a: A) extends F[A]

SomeClass("hello"): F[String]
SomeClass(1: Int): F[Int]
SomeClass(true): F[Boolean]
{% endhighlight %}

This is pretty straight forward and you probably aren't impressed, but it is
important to see the basic example since the following ones will be variations
on it. Now, lets move on to the next kind: Phantoms.

# Phantom Types

A phantom type refers to any type that cannot be created (at runtime): they
exist on their own. For example:

{% highlight scala %}
sealed trait A

sealed trait B
final case class MkB() extends B
{% endhighlight %}

`A` is considered to be phantom since it has no constructor (ignoring the
implicit constructor that all traits have). `B`, on the other hand, isn't
phantom since we can create a `B` via `MkB`.

As a side note, the following `A` is a true phantom types that can't be created
(no implicit constructor):

{% highlight scala %}
object TruePhantoms {
  type A
}
{% endhighlight %}

But it is harder to use this form in Scala, so it is better to pretend that
sealed traits, with no cases, are also phantom.

# Phantom Type Variables

Phantom type *variables* are similar to normal phantom types in that they
exist on their own and can't be created. To show this, lets modify our plain
example from before:

{% highlight scala %}
type F[A] = SomeClass
{% endhighlight %}

Note that the `A` is now **only** on the left side; it is no longer bound to
whatever the type of data `SomeClass` takes. For example:

{% highlight scala %}
SomeClass("hello"): F[Boolean]
SomeClass("hello"): F[Int]
SomeClass("hello"): F[User]
{% endhighlight %}

It doesn't matter what the type of the data we put into `SomeClass`, the
resulting type will be whatever we choose it to be.

As we did before, lets expand it into a usable form:

{% highlight scala %}
sealed trait Phantom[A]

final case class MkPhantom[A, B](b: B) extends Phantom[A]
{% endhighlight %}

Though `A` is part of the definition of `MkPhantom`, note that it is NOT
dependent on the data passed to `MkPhantom`. This means we can still do what
we did before:

{% highlight scala %}
MkPhantom("hello"): Phantom[Boolean]
MkPhantom("hello"): Phantom[Int]
MkPhantom("hello"): Phantom[User]
{% endhighlight %}

Now, `A` may not be dependent on any data **in `MkPhantom`**, but there is
nothing stopping us from creating another case where **it is** dependent on
data. For example:

{% highlight scala %}
sealed trait Phantom[A]

final case class MkPhantom[A, B](b: B) extends Phantom[A]
final case class MkPhantom2[A](a: A) extends Phantom[A]
{% endhighlight %}

This is the most common form of Phantoms we see in the wild. For example,
`Option` is defined as such:

{% highlight scala %}
sealed trait Option[A]

final case class None[A]() extends Option[A]
final case class Some[A](a: A) extends Option[A]
{% endhighlight %}

Of course, this isn't the [true definition of `Option`](option-link) since it
would be wasteful, but the concept still applies.

`List` is another example: `Nil` being phantom while `Cons` isn't. There are
other [more complex](tagged-link) examples but we won't be going into them.
Instead, we will be moving on to Existential types.
