# A Metaprogramming Introduction for Spatial

## Table of Contents


## Introduction
[Spatial](https://github.com/stanford-ppl/spatial) is an embedded DSL in the Scala language. This document discusses the primary differences between Spatial and vanilla Scala, as well as how to leverage Scala to create more complex and powerful constructs in Spatial.

## Virtualization
The embedding for Spatial in Scala happens in two major stages. First, language-level changes are performed by re-writing structures tagged with `@spatial`. Then, Scala-level overrides occur through the definition of new functions, objects, and implicits.


### Compiler Rewrites
For objects and classes tagged with `@spatial`, the compiler makes code-level changes. These are documented in [forge.tags.Virtualizer](https://github.com/stanford-ppl/spatial/blob/master/forge/src/forge/tags/Virtualizer.scala). Of particular note is that `if-else` and `var` are transformed, and as a result do not behave as you would normally expect in Scala. As a result, any code which uses these constructs (among others) should be moved into utility classes/objects which are not tagged with `@spatial`.

<a id="example1"></a>
```Scala
import spatial.dsl._

object Utils {
  def utilityFunc(...)(implicit state: argon.State) = {
    // Branches, vars, etc. outside of @spatial are not transformed
    if (something) {
      value
    } else {
      other value
    }
  }
}

@spatial object Thing extends SpatialApp {
  def main(args: Array[String]): Unit = {
    Utils.utilityFunc(...)
  }
}
```

Currently, there do not appear to be vim plugins or traditional IDEs which can intelligently reason about these changes. As a result, bugs due to rewriting are quite difficult to trace.

### Scala-level overrides
In addition to the compiler rewrites, Spatial also contains overrides of common constructs. Note here that there are two "versions" of spatial at play here: [`spatial.dsl`](https://github.com/stanford-ppl/spatial/blob/master/src/spatial/dsl.scala#L25) and [`spatial.libdsl`](https://github.com/stanford-ppl/spatial/blob/master/src/spatial/dsl.scala#L8). These differ in that `spatial.dsl` shadows many common Scala names, such as `Array`, `String`, etc. The full list of these can be found in [`spatial.lang.Aliases`](https://github.com/stanford-ppl/spatial/blob/master/src/spatial/lang/Aliases.scala).

### Extra Bullshit
Scala's [implicit lookup rules](https://docs.scala-lang.org/tutorials/FAQ/finding-implicits.html) mean that if you have a function defined which takes `argon.State` as an implicit (which is necessary for all of the operations tagged `@stateful`, consisting of almost everything), you end up pulling in `argon.lang.api.Implicits`, which includes `IntWrapper`, which converts range expressions like `1 until 3` into `argon.lang.Series[I32]` objects instead of `scala.collections.immutable.Range`. The current workaround is to avoid using `x until y` in favor of `x to (y - 1)`.

## Building a Spatial Graph
Once outside the clutches of `spatial.dsl._`, we are empowered to use standard Scala constructs, provided we have the right implicits. These implicits are either provided by the application (in particular `argon.State`) or from general implicits (such as conversions). In the [code example](#example1), the state implicit is used to construct the corresponding Spatial Graph.

Note that if we were operating outside a scope where the original types are available (such as when creating a type-parameterized utility function), a large block of implicit parameters are necessary to pull in all of the associated data in order to create the operations. We currently do not have a workaround for this problem.

### Interfacing Between Different Storage Types
In order to easily handle differences between different storage structures such as different `DRAM` dimensions, `SRAM`s, Scala arrays of `Reg`, etc., we need a common interface across dramatically different types. Note that in this environment, Spatial and Scala constants can be almost interchanged.

```Scala
// Scala int to spatial value
spatial_value = uconst[T](FixedPoint.fromInt(scala_int))

// Spatial value back to Scala value (FixedPoint becomes a Scala BigInt)
scala_value = spatial_value.c.get.asInstanceOf[emul.FixedPoint].value
```

Note that in the example above, we need to know how the actual type of the value (`emul.FixedPoint`).

In order to build this interface, we can consider reading from a data store to have the signature `(IndexType => ValueType)`, and writing to have the signature `(IndexType, ValueType) => argon.Void`.

```Scala
object Utils {
  def copy[T](reader: (I32 => T), writer: ((I32, T) => Void), lower: Int, upper: Int): Void = {
    (lower to (upper - 1)) foreach {
      index => writer(I32(index), reader(I32(index)))
    }
  }
}

@spatial object Example extends SpatialApp {
  def main(args: Array[String]): Unit = {
    ...
    Accel {
      val sram = SRAM[T](N)
      val rf = RegFile[T](N)
      Utils.copy(
        {ind: I32 => sram(ind)},
        {(ind, val): (I32, T) => rf(ind) := val}
      )
    }
    ...
  }
}
```

Note that since these techniques are simply Scala at this point, we can use almost arbitrary constructs inside of the access functions with the catch that functions defined inside of the `@spatial` object are still subject to the compiler rewrites.


## Type-ical Metaprogramming
One thing to note is that all metaprogramming so far is done at the Scala level, meaning that while we have flexibility on the computation, we do not have the ability to directly interact with the types involved.

This problem arises when we wish to do a data-dependent generation of types. For example, if we know that a particular array is to be loaded into a SRAM, then we can size the individual elements to be precisely the correct precision.

```scala
// Array values
val values = Array(1.125, 2.25, 1.3125, 2.75)

// Proper S, I, F format can be easily computed from the data.

val sign = values.exists { _ < 0 }
val integer = (values map {x => if (x > 0) log2(x) else log2(-x + 1)}).max
val frac = (Stream.from(0).find {
  shift => values forall {
    value =>
    {
      val low_prec = (value << shift).round / pow(2, shift)
      abs(low_prec - value) < error
    }
  }
}).get
```

However, since we represent numerical formats by type (i.e. `FixPt[TRUE, _3, _13]`), we are unable to actually create such a value. However, with a fair bit of magic we can make this happen.

```scala
// Note that quasiquoting + eval essentially discards all compiler type-checking between the quasiquoted portion and the rest of the program.

import scala.reflect.runtime.currentMirror
import scala.tools.reflect.ToolBox
import scala.reflect.runtime.universe._

def createFixPt(sign: Boolean, integer: Int, fractional: Int, value: Double): FixPt[_, _, _] = {
  val toolbox = currentMirror.mkToolBox()
  val signType = TypeName(if (sign) "TRUE" else "FALSE")
  val intName = TypeName(s"_$integer")
  val fracName = TypeName(s"_$fractional")
  val qq = tq"spatial.lang.FixPt[$signType, $intName, $fracName]"

  val constant = Literal(Constant(value))

  toolbox.eval(
    q"argon.lang.uconst[$qq](spatial.emul.FixedPoint.fromDouble($constant))"
  ).asInstanceOf[FixPt[_, _, _]]
}
```
