---
title: Orphan instances in Scala
tags: [scala, typeclasses, haskell, instances, style]
layout: post
---

## Orphan Instances in Scala

Type classes in Scala are interesting, in that even
though they aren't part of the language, they are widely used. This is mostly
attributed to type classes' usefulness and the ability for Scala.

[Type classes first appeared in Haskell as a way to implement overloading in a
principled way][1]. Since then many other applications have been found for
them. I will not describe type classes in detail (if you are unfamiliar with
them, you should read [Sohum Banerjea's excellent intro][typeclass-post]),
instead I would like to focus on orphan instances.

An orphan instance in Scala means an instance that exists neither in the type's
companion object nor the type class' companion object. For example:

{% highlight scala %}
object FooPackage {
  case class Foo(msg: String)
}

object DoFooPackage {
  trait DoFoo[A] {
    def doSomething(f: A): String
  }
  object DoFoo {
    def apply[A](implicit ev: DoFoo[A]) = ev
  }
}

object InstanceFoo {
  implicit object FooInst extends DoFoo[Foo] {
    def doSomething(f: Foo) = "I am a Foo! My msg is: " + f.msg
  }
}
{% endhighlight %}

In the above example, `FooInst` is said to be an "orphan instance". This is
because it is not attached to neither `DoFoo` or `Foo` (via their companion
objects). This means that in order to use the type class, we must import three
packages: the package with `Foo`, the package with `DoFoo` and the package with
the instance declaration (in the code above, we used objects to represent
packages).

```scala
import FooPackage._
import DoFooPackage._
import InstanceFoo._

DoFoo[Foo].doSomething(Foo("Hello, World!"))
```

Something you will notice from this style is that we can now define a different
instance for Foo and import it in order to change `doSomething`'s behaviour.
For example:

```scala
object InstanceFooV2 {
  implicit object FooInst2 extends DoFoo[Foo] {
    def doSomething(f: Foo) = "I am a second version of Foo! I say: " + f.msg
  }
}
```

Now we can change the way the program works by simply changing an import! If
the previous statement didn't scare you, it should have.

The examples that we have used so far have been quite simple, we are importing
a couple of packages and that's it. But in reality, your import statement is
probably 10-20 (or maybe even more) lines long, and filled with wild-card
imports, this leads to a couple of problems.

The first problem has to do with referential transparency. If you see `1 + 1`
anywhere in your code, you would like it to always return `2`; it would be
quite odd if in some parts of your code `1 + 1 = 3`. Same applies to
type-classes, if we see `doSomething(Foo("Hello"))`, it would be great if that
didn't change next time we see it. However, having multiple orphan instances
breaks this transparency; what `doSomething(Foo("Hello"))` does is now
dependent on one of the many imports at the top of the file. This can make
refactoring and cleaning a very scary process.

The second problem you might encounter when using orphans is that changing a
single line in your (very long) import statement can have drastic effects on
the program while still compiling. This happens when declaring *overlapping
instances*. This won't happen with the example above since importing both
`InstanceFoo` and `InstanceFoo2` (and then trying to call `doSomething`) would
have given an implicit conflict compile error due to both of them being instances
for the same exact type (Foo). However, this doesn't happen with overlapping
instances.

```scala
object ListInstance {
  implicit def listInstance[A](implicit ev: DoFoo[A]): DoFoo[List[A]] = new DoFoo[List[A]] {
    def doSomething(list: List[A]) =
      "I am a list...of something that defines `DoFoo`:\n\t" + list.map(e => ev.doSomething(e)).mkString(",")
  }
}

object ListofFooInstance {
  implicit object ListFooInst extends DoFoo[List[Foo]] {
    def doSomething(listOfFoo: List[Foo]) =
      "I am a specific instance of list of foos!\n\t" + listOfFoo.map(e => doSomething(e)).mkString("\n")
  }
}
```

Above, we made a generic instance for the container "List". As part of the
instance definition, we stated that the inner type has to also define a `DoFoo`
instance; this means that `doSomething(List(Foo("Hello"), Foo("World")))` will
work fine. However, we also defined a specific instance for `List[Foo]`. What
would happen if we import both of them? According to the Scala language
specification:

    If there are several eligible arguments which match the implicit
    parameter’s type, a most specific one will be chosen using the rules of
    static overloading resolution (§6.26.3).

This means that the `ListofFooInstance` will be picked if we import both. But,
more importantly, if we remove it from the import statement, it will then pick
the `ListInstance` and continue to compile *even though it is doing something
very different*. This can, again, make cleaning and refactoring a painful
process.

So what if you do actually want to have a different representation (in a
type-class) for the same type? This is where value classes come to the rescue.
A value class allows us to create a new type that exists only at compile time:
at runtime, it is replaced by the underlying type. But because it is a
different type, we can create a new, non-orphan instance for it. And because it
is a simple type wrapper, we can quickly and easily switch from it to the
underlying type:

```scala
package NewFoo

case class NewFoo(toFoo: Foo) extends AnyVal

object NewFoo {
  implicit object NewFooInst extends DoFoo[NewFoo] {
    def doSomething(nf: NewFoo) =
      "A new message for Foo! It is: " + nf.toFoo.msg
  }
}
```

Of course, this means that in order to use this new functionality you have to
wrap `Foo`, like so `doSomething(NewFoo(Foo("Hello")))`. However, it is now
explicit that this is different; we can expect that line to do the same thing
everywhere and never what `doSomething(Foo("Hello"))` would do.

Now, this does not mean you should avoid orphans like the plague. There are
certain cases where they can't be avoided (or avoiding them would be too annoying).
For example, if you have two libraries: one has a case class you would like to
use and the other has a type class you would also like to use. You could create
a new type wrapper (like `NewFoo`) around the class, but if you are going to
be using said class extensively, then maybe it is ok to simply create an orphan
instance for it (just make sure you don't create more than one). If you have
access to either the type class or the case class, just place the instance in
either one's companion object.

In conclusion, orphan instances can be helpful at time, but they can quickly
grow unwieldy as your codebase grows and becomes more complex if not kept in
check. Value classes can help avoid them all together, at the cost of having to
wrap unwrap values. But these may be worth the cost as your codebase grows and
your type classes become more complex.

-------------------------------------------------------------------------------

[1]: http://homepages.inf.ed.ac.uk/jmorri14/d/final.pdf
[typeclass-post]: http://www.cakesolutions.net/teamblogs/demystifying-implicits-and-typeclasses-in-scala
