An orphan instance, in Haskell, is an instance for a type class that has been
defined neither in the type's module nor the type-class' module. A Scala orphan
instance is similar; an instance (implicit val/object/def) that is defined not
in the type's companion object or the type-class' companion object:

```haskell
module DataFoo where

data Foo = Foo String

module ClassFoo where

class DoFoo a where
  doSomething :: a -> String

module InstanceFoo where

instance DoFoo Foo where
  doSomething (Foo msg) = "I am a Foo! My msg is: " ++ msg
```

In the example above, the instance for Foo (in `InstanceFoo`) is said to be an
orphan. What this means is that if I would like to call
`doSomething(Foo "Hello World")`, I would need to import three modules:

```haskell
import DataFoo
import ClassFoo
import InstanceFoo

doSomething(Foo "Hello World")
```

However, if I were to move the instance declaration to either the `DataFoo` or
the `ClassFoo` module, I would save one import. But if I do that, then I
wouldn't be able to use another instance:

```haskell
module InstanceFoo2 where

instance DoFoo Foo where
  doSomething (Foo msg) = "I am a second version of Foo! I say: " ++ msg
```

If we do this (use an orphan instance) we can simply just import the
instance we want. Now in Haskell, this is actually pretty hard to maintain
since (in Haskell) type class instances spread like wildfire. This means that
instances can't be excluded from exports or imports: they are always exported
from modules and always imported from imports. And to make the fire spread even
more, they are always re-exported from any imports! So if a module A defines an
instance for some class and module B imports module A, and C import B, C will
see the instances defined in A even though module A isn't imported.

![Haskell's instance re-export system](/assets/scala-orphans/haskell-instance-reexports.png)

This means we can't reliably use multiple instances for a given class and type
unless we make sure that we never, ever, have two import *paths* merge. Of
course, in large codebases this is pretty hard (or flat out impossible).
