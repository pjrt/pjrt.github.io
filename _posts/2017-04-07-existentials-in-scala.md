---
title: Existential types in Scala
tags: [scala, types, existential, typeclass, typelevel]
layout: post
---

Type variables are a powerful piece of type-level programming. They can appear
in a variety of forms and factors; phantoms and existential types being two
cases. In this article we will be exploring existential types via a specific
use case you may encounter in production code.

But before we can talk about existential types, we must talk about type
variables in general.

# Type Variables

Normally, this is the form in which we see type variables:

{% highlight scala %}
type F[A] = SomeClass[A]
{% endhighlight %}

`A` is said to be a "type variable". Note that `A` appears on both sides of the
equation: on `F` and on `SomeClass`. This means that `F` is fully dependent on
the type of the data for `SomeClass`. For example, all of these are valid:

{% highlight scala %}
SomeClass("hello"): F[String]
SomeClass(1: Int): F[Int]
SomeClass(true): F[Boolean]
{% endhighlight %}

While the following isn't:

{% highlight scala %}
SomeClass("hello"): F[Int] // does not compile
{% endhighlight %}

Now let's expand this into something more concrete (with traits and classes):

{% highlight scala %}
sealed trait F[A]
final case class SomeClass[A](a: A) extends F[A]

SomeClass("hello"): F[String]
SomeClass(1: Int): F[Int]
SomeClass(true): F[Boolean]
{% endhighlight %}

This is pretty straight forward and you probably aren't impressed, but it is
important to see the basic example since existential types (and phantom ones)
are a variation on it.

# Existential Types

Existential types are like the normal type variables we saw above, except
the variable only shows up on the right:

{% highlight scala %}
type F[A] = SomeClass[A] // `A` appears on the left and right side, common case

type F = SomeClass[A] forSome { type A } // `A` appears only on the right, existential case
{% endhighlight %}

Now `A` only appears on the right side, which means that the final type, `F`,
will not change regardless of what `A` is. For example:

{% highlight scala %}
SomeClass("hello"): F
SomeClass(1: Int): F
SomeClass(user): F
{% endhighlight %}

Side note: if `A` were to only appear on the left side, then `A` would be a
Phantom type. We aren't covering those in this article though, so let's ignore
them.

Like before, let's now expand it. Beware, this expansion isn't as clean as the
previous one:

{% highlight scala %}
sealed trait Existential {
  type Inner
  val value: Inner
}

final case class MkEx[A](value: A) extends Existential { type Inner = A }
{% endhighlight %}

Here we're using path dependent types in order to create a better interface for
our `Existential` trait. We could just have an empty trait but that would mean
that we would need to case match `MkEx` whenever we wanted to access its
fields. However, the concept still holds and shares the properties we described
above:

{% highlight scala %}
MkEx("hello"): Existential
MkEx(1: Int): Existential
MkEx(user): Existential
{% endhighlight %}

We could think of `MkEx` as a type eraser: it doesn't matter what type of data
we choose to put into `MkEx`, it will erase the type and always return
`Existential`.

Time for a game, let's say you have some function that, when called,
returns an `Existential`. What is the type of the inner `value` field?

{% highlight scala %}
val ex: Existential = getSomeExistential(...)
ex.value: ???
{% endhighlight %}

The answer is `ex.Inner` of course. But what is `ex.Inner`? The answer to
that is that we can't know. The original type defined in `MkEx` has been erased
from compilation, never to be seen again. Now that's not to say that we have
lost all information about it. We know that it *exists* (we have a reference to
it via `ex.Inner`), hence why it is called "existential", but sadly that's
pretty much all the information we know about it.

This means that, in its current form, `Existential` and `MkEx` are useless. We
can't pass `ex.value` anywhere expect where `ex.Inner` is required, but even
`ex.Inner` is pretty bare bones with no properties for us to use: so once we
accept it in a function, what do we do with it? Well nothing right now, but we
could add *restrictions* to the type, which in terms would allow us to do
something with it without knowing *what* it is.

And here is where Existential types shine: they have the interesting property
of unifying different types into a single one with shared restrictions. These
restrictions could be anything: from upper bounds to type classes or even a
combination. To show this we will be creating a type-safe wrapper around an
unsafe Java library.

Let's say you have a Java library that contains the following signature:

{% highlight scala %}
def bind(objs: Object*): Statement
{% endhighlight %}

This signature isn't special, many SQL Java libraries have a method similar to
it. Normally you would define some string with many question marks (?) followed
by a call to a `bind` method that would bind the passed objects to the question
marks (hopefully sanitizing the input in the process). Like so:

{% highlight sql %}
SELECT * FROM table WHERE a = ? AND b = ?;
{% endhighlight %}

The problem though is that `Object` is a very general thing. Obviously, if we
pass `bind(user)`, where `user` is some class we defined, that wouldn't work.
However, it would compile and crash at runtime. Additionally, a lot of these
Java libraries are, well, for Java, so certain Scala types won't work either
(eg: BigInt, Int, List). In fact, you *could* think of `Object` as an
existential type.

{% highlight scala %}
SomeClass(1): Object
SomeOtherClass("hello"): Object
Boolean.box(true): Object
{% endhighlight %}

The problem is that it is too wide, it encompasses ALL types. This all means
that we need to somehow "restrict" the number of types that are allowed to go
into `bind`. This new bind, we shall call it `safeBind` will:

* Convert types to their Java counterpart (BigInt -> Long, Int -> Integer)
* Fail to compile any nonsensical types (User, Profile)

We aren't going to be making the length of parameters passed safe (that would
require a lot more than existentials). Nor are we checking that the types
passed match the expected types in the SQL query.

`safeBind` will be defined something like so:

{% highlight scala %}
def safeBind(columns: AnyAllowedType*): Statement =
  bind(columns.map(_.toObject):_*)
{% endhighlight %}

Note that `safeBind` doesn't look that much different that `bind` except for
`AnyAllowedType`. But before we can talk about `AnyAllowedType`, we need to
define what types are allowed. For this, we will use a typeclass, called
`AllowedType`, since they lend themselves well for defining a set of types
unrelated types that have a specific functionality.

This typeclass is defined as:

{% highlight scala %}
sealed trait AllowedType[A] {
  /**
   * The Java type we are converting to.
   *
   * Note the restrictions:
   *
   * * `:> Null` means that we can turn the type into `null`.
   *   This is needed since many Java SQL libraries interpret NULL as `null`
   * * `<: AnyRef` just means "this is an Object". ie: not an AnyVal.
   */
  type JavaType :> Null <: AnyRef

  /**
   * Function that converts `A` (eg: Int) to the JavaType (eg: Integer)
   */
  def toJavaType(a: A): JavaType

  /**
   * Same as above, but upcasts to Object (which is what `bind` expects)
   */
  def toObject(a: A): Object = toJavaType(a)
}
object AllowedType {
  def apply[A](implicit ev: AllowedType[A]) = ev
  def instance[A, J >: Null <: AnyRef](f: A => J): AllowedType[A] =
    new AllowedType[A] {
      type JavaType = J
      def toJavaType(a: A) = f(a)
    }
}
{% endhighlight %}

The stuff in the companion object are just helper functions we will use later.
Now let's define some types that will be allowed through:

{% highlight scala %}
object AllowedType {
  implicit val intInstance: AllowedType[Int] = instance(Int.box(_))
  implicit val strInstance: AllowedType[String] = instance(identity)
  implicit val boolInstance: AllowedType[Boolean] = instance(Boolean.box(_))
  implicit val instantInst: AllowedType[Instant] = instance(Date.from(_))

  // For Option, we turn `None` into `null`; this is why we needed that `:> Null`
  // restriction
  implicit def optionInst[A](implicit ev: AllowedType[A]): AllowedType[Option[A]] =
    instance[Option[A], ev.JavaType](s => s.map(ev.toJavaType(_)).orNull)
}
{% endhighlight %}

This gives us our filter and converter: we can only call
`AllowedType[A].toObject(a)` if `A` implements our typeclass.

Great, now we need to define `AnyAllowedType`. As we saw above, we want
`AnyAllowedType` to behave somewhat like `Object`, but for our small set of
types. We can achieve this using an Existential type but with an evidence for
our `AllowedType` typeclass:

{% highlight scala %}
sealed trait AnyAllowedType {
  type A
  val value: A
  val evidence: AllowedType[A]
}

final case class MkAnyAllowedType[A0](value: A0)(implicit val evidence: AllowedType[A0])
  extends AnyAllowedType { type A = A0 }
{% endhighlight %}

This existential is similar to the `Existential` we defined above, except that
we are now asking for an evidence for `AllowedType[A]` and capturing it as part
of the trait. This means that, unlike before, we can't pass any random type
anymore:

{% highlight scala %}
MkAnyAllowedType("Hello"): AnyAllowedType
MkAnyAllowedType(1: Int): AnyAllowedType
MkAnyAllowedType(Instant.now()): AnyAllowedType

MkAnyAllowedType(user): AnyAllowedType // won't compile since we don't have an AllowedType instance for User
{% endhighlight %}

Great! Now we have our `AnyAllowedType` defined. Let's see it in action by
defining `safeBind`.

{% highlight scala %}
def safeBind(any: AnyAllowedType*): Statement =
  bind(any.map(ex => ex.evidence.toObject(ex.value)):_*)
{% endhighlight %}

Note that we still don't know what the type of `ex.value` is (just like before).
However, we *do* know that it is the same type as the evidence `ex.evidence`.
That means that all functions, that are part of the evidence (ie: typeclass),
match the type of `ex.value`! So our knowledge of `ex.value` has expanded from,
"all we know is that it exists" to "we know it exists AND that it implements
the typeclass AllowedType".

Finally, when we go to use `safeBind`, we do as such:

{% highlight scala %}
safeBind(MkAnyAllowedType(1), MkAnyAllowedType("Hello"), MkAnyAllowedType(Instant.now()))
{% endhighlight %}

And it works! But the following will not:

{% highlight scala %}
safeBind(MkAnyAllowedType(1), MkAnyAllowedType(user)) // Does not compile, no instance of AllowedType for User
{% endhighlight %}

We are essentially done now, we just need to wrap our values in
`MkAnyAllowedType` and the compiler will do the rest (or yell).

However, there are some extra tweaks we can make to make our interface better.

## Making an AllowedType instance for AnyAllowedType

You many have noticed that it is awkward to call functions in `ex.evidence`.

{% highlight scala %}
ex.evidence.toObject(ex.value)
{% endhighlight %}

We can make this better by creating an instance for `AllowedType`:

{% highlight scala %}
object AnyAllowedType {
  implicit val anyAllowedInst: AllowedType[AnyAllowedType] =
    AllowedType.instance(ex => ex.evidence(ev.value))
}

// Now we can simply do
val ex: AnyAllowedType = …
AllowedType[AnyAllowedType].toObject(ex)
{% endhighlight %}

## Using implicit conversions to avoid wrapping

Having to do this manual wrapping can become old fast:

{% highlight scala %}
safeBind(MkAnyAllowedType(1), MkAnyAllowedType("Hello"), …)
{% endhighlight %}

We can actually avoid it by using an implicit conversion:

{% highlight scala %}
object AnyAllowedType {
  implicit def anyAllowedToAny[A: AllowedType](a: A): AnyAllowedType =
    AnyAllowedType(a)
}
{% endhighlight %}

Now we can simply call:

{% highlight scala %}
safeBind(1, "Hello", Instant.now(), true, …)
{% endhighlight %}

And passing `user` will fail, like before.

## Generalize AnyAllowedType

When we created `AnyAllowedType`, we made it for the typeclass `AllowedType`.
Does this mean that we need to make a new `AnyX` for every `X` typeclass we
have? Nope, we do not. We can generalize `AnyAllowedType` to work for ANY
typeclass. This would require a simple modification:

{% highlight scala %}
sealed trait TCBox[TC[_]] {
  type A
  val value: A
  val evidence: TC[A]
}

final case class MkTCBox[TC[_], B](value: A)(implicit val evidence: TC[A])
  extends TCBox[TC] { type A = B }
{% endhighlight %}

Now, instead of hard-coding it to `AllowedType`, we take the typeclass as a
type `TC[_]`. We still take a `TC[A]` implicitly in `MkTCBox` along with the
value.  Note though that `TC[_]` isn't existential, nor phantom, it is just a
common type variable. A `TCBox[TC]` is, essentially, a "TypeClass Box" for the
typeclass "TC".

Our implicit conversion can also be translated:

{% highlight scala %}
object TCBox {
  implicit def anyTCBox[TC[_], A: TC](a: A): TCBox[TC] = MkTCBox(a)
}
{% endhighlight %}

Finally, a simple type alias `type AnyAllowedType = TCBox[AllowedType]` would
make everything we have written before keep working.

# Conclusion

Existential types don't seem that useful when first encountered, but they can
be quite powerful when mixed with the correct restrictions. The use case we
presented is one of the simpler uses but they can be as complex or as simple
as your use case requires them to be.

PS: you can find all the code we just wrote for TCBox here: <https://gist.github.com/pjrt/269ddd1d8036374c648dbf6d52fb388f>
