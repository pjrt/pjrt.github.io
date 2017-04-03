---
title: Phantom and Existential types in Scala
tags: [scala, types, existential, typeclass]
layout: post
---

There are many different ways to types in Scala [TODO]

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

So this means that I can't just, for example, do the following:

{% highlight scala %}
SomeClass("hello"): F[Int]
{% endhighlight %}

The above won't work. Now lets expand this into something more concrete:

{% highlight scala %}
sealed trait F[A]
case class SomeClass[A](a: A) extends F[A]

SomeClass("hello"): F[String]
SomeClass(1: Int): F[Int]
SomeClass(true): F[Boolean]
{% endhighlight %}

This is pretty straight forward and you probably aren't impressed, so lets move
on to the next kind; but before we start on existential, I want to show
"phantom Types" since they are nicely dichotomic to existential types.

# Phantom Types

A phantom type refers to any type that cannot be created from data. They exist
on their own. To show this, lets modify our example above:

{% highlight scala %}
type F[A] = SomeClass
{% endhighlight %}

Note that the `A` is now ONLY on the left side; it is no longer bound to
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

# Existential Types

As mentioned before, Existential types can be thought of as the opposite of a
phantom type. This essentially means reversing where the type variable `A`
appears.

{% highlight scala %}
type F = SomeClass[A]
{% endhighlight %}

Now `A` only appears on the right side, which means that the final type, `F`,
will not change regardless of what `A` is. For example:

{% highlight scala %}
SomeClass("hello"): F
SomeClass(1: Int): F
SomeClass(user): F
{% endhighlight %}

Now lets expand it. Beware, this expansion isn't as clean as the previous ones:

{% highlight scala %}
sealed trait Existential {
  type Inner
  val value: Inner
}

final case class MkEx[A](value: A) extends Existential { type Inner = A }
{% endhighlight %}

Here were are using path dependent types in order to create a better interface
for our `Existential` trait. We could just have an empty trait but that would
mean that we would need to case match `MkEx` whenever we wanted to access its
fields. However, the concept still holds and well as the property we described
above:

{% highlight scala %}
MkEx("hello"): Existential
MkEx(1: Int): Existential
MkEx(user): Existential
{% endhighlight %}

We could think of `MkEx` as a type eraser: if you give it some data, it will
erase the type while keeping the data.

Time for a game, lets say you have some function that, when called,
returns an `Existential`. What is the type of the inner `value` field?

{% highlight scala %}
val ex: Existential = getSomeExistential(...)
ex.value: ???
{% endhighlight %}

The answer is...`ex.Inner` of course. But what is `Inner`? The answer to that
is...we can't know. The original type defined in `MkEx` has been erased from
compilation, never to be seen again. Now that's not to say that we have lost
all information about it. We know that it *exists* (we have a
reference to it via `ex.Inner`), hence why it is called "existential", but
sadly that's pretty much all the information we know about it.

This means that, in its current form, `Existential` and `MkEx` are useless.
We can't pass `ex.value` anywhere expect where `ex.Inner` is required, but
even `Inner` is pretty bare bones with no properties for us to use: so once we
accept it in a function, what do we do with it? What can we do with it? Well
nothing right now, but we could add *restrictions* to the type, which in terms
would allows us to do something with it.

And here is where Existential types shine: they have the interesting property
of unifying different types into a single one with shared restrictions. These
restrictions could be anything: from upper bounds to type classes or even a
combination. To show this we will be creating a type-safe wrapper around an
unsafe Java library.

Lets say you have a Java library that contains the following signature:

{% highlight scala %}
def bind(objs: Object*): Statement
{% endhighlight %}

This signature isn't special, many SQL Java libraries have a method similar to
it. Normally you would define some string with many question marks (?) followed
by a call to a `bind` method that would bind the passed objects to the question
marks. Like so:

{% highlight sql %}
SELECT * FROM table WHERE a = ? AND b = ?;
{% endhighlight %}

The problem though is that `Object` is a very general thing. Obviously, if we
pass `bind(user)`, where `user` is some class we defined, that wouldn't work.
However it would compile and crash at runtime. Additionally, a lot of these
Java libraries are, well, for Java, so certain Scala types won't work either
(eg: BigInt, Int, List). In fact, you *could* think of `Object` as an
existential type.

{% highlight scala %}
SomeClass(1): Object
SomeOtherClass("hello"): Object
Boolean.box(true): Object
{% endhighlight %}

the problem is that it is too wide, it encompasses ALL AnyRef types (which are
all types expect AnyVals).

This all means that we need to somehow "restrict" the number of types that are
allowed to go into `bind`. This new bind, we shall call it `safeBind` will:

* Convert types to their Java equivalent (BigInt -> Long, Int -> Integer)
* Fail to compile any nonsensical types (User, Profile)

We aren't going to be making the length of parameters passed safe (that would
require a lot more than existentials). Nor are we checking that the types
passed match the expected types in the SQL query.

`safeBind` will be defined something like this:

{% highlight scala %}
def safeBind(columns: AnyAllowedType*): Statement =
  bind(columns.map(_.toObject):_*)
{% endhighlight %}

Note that `safeBind` doesn't look that much different that `bind` except for
`AnyAllowedType`. But before we can talk about `AnyAllowedType`, we need to
define what types are allowed. For this, we will use a typeclass, called
`AllowedType`, since they lend themselves well for defining a set of types that
have a specific functionality.

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

Now, lets define some types that will be allowed through:

{% highlight scala %}
object AllowedType {
  implicit val intInstance: AllowedType[Int] = instance(Int.box(_))
  implicit val strInstance: AllowedType[String] = instance(identity)
  implicit val boolInstance: AllowedType[Boolean] = instance(Boolean.box(_))
  implicit val instantInst: AllowedType[Instant] = instance(Date.from(_))

  // For Option, we turn `None` in `null`; this is why we needed that `:> Null`
  // restriction
  implicit def optionInst[A](implicit ev: AllowedType[A]): AllowedType[Option[A]] =
    instance[Option[A], ev.JavaType](s => s.map(ev.toJavaType(_)).orNull)
}
{% endhighlight %}

This gives us our filter and converter: we can only call
`AllowedType[A].toObject(a)` on values of a defined type. And if the type needs
to be converted, it will be done so.

Great, now we need to define `AnyAllowedType`. As we saw above, we want
`AnyAllowedType` to behave somewhat like `Object`, but for our small universe
of types. We can achieve this using a Existential type but with an evidence
for our `AllowedType` typeclass:

{% highlight scala %}
sealed trait AnyAllowedType {
  type A
  val value: A
  val evidence: AllowedType[A]
}

final case class MkAnyAllowedType[B](value: A)(implicit val evidence: AllowedType[A])
  extends AnyAllowedType { type A = B }
{% endhighlight %}

This existential is similar to the `Existential` we defined above, except that
we are now asking for an evidence for `AllowedType[A]` and capturing as part
of the trait. This means that, unlike before, we can't pass any random type
anymore:

{% highlight scala %}
MkAnyAllowedType("Hello"): AnyAllowedType
MkAnyAllowedType(1: Int): AnyAllowedType
MkAnyAllowedType(Instant.now()): AnyAllowedType

MkAnyAllowedType(user): AnyAllowedType // won't compile since we don't have an instance for User
{% endhighlight %}

Great! Now we have our `AnyAllowedType` defined. Lets see it in action by
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

And it works! We are essentially done now, developers just need to wrap their
values in `MkAnyAllowedType` and the compiler will do the rest
(or yell at them).

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

Now, instead of hard-coding it to AllowedType, we take the typeclass type as
`TC[_]`. We still take a `TC[A]` implicitly in `MkTCBox` along with the value.
Note though that `TC[_]` isn't existential, nor phantom, it is just a common
type variable.

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
be quite powerful when mixed with the correct restrictions. The use case I
presented is one of the simpler uses but they can be as complex or as simple
as your use case requires them to be.
