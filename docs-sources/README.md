![diffx](https://github.com/softwaremill/diffx/raw/master/banner.png)

[![Build Status](https://travis-ci.org/softwaremill/diffx.svg?branch=master)](https://travis-ci.org/softwaremill/diffx)
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/com.softwaremill.diffx/diffx-core_2.13/badge.svg)](https://search.maven.org/search?q=g:com.softwaremill.diffx)
[![Gitter](https://badges.gitter.im/softwaremill/diffx.svg)](https://gitter.im/softwaremill/diffx?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)
[![Mergify Status](https://img.shields.io/endpoint.svg?url=https://gh.mergify.io/badges/softwaremill/diffx&style=flat)](https://mergify.io)
[![Scala Steward badge](https://img.shields.io/badge/Scala_Steward-helping-brightgreen.svg?style=flat&logo=data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAA4AAAAQCAMAAAARSr4IAAAAVFBMVEUAAACHjojlOy5NWlrKzcYRKjGFjIbp293YycuLa3pYY2LSqql4f3pCUFTgSjNodYRmcXUsPD/NTTbjRS+2jomhgnzNc223cGvZS0HaSD0XLjbaSjElhIr+AAAAAXRSTlMAQObYZgAAAHlJREFUCNdNyosOwyAIhWHAQS1Vt7a77/3fcxxdmv0xwmckutAR1nkm4ggbyEcg/wWmlGLDAA3oL50xi6fk5ffZ3E2E3QfZDCcCN2YtbEWZt+Drc6u6rlqv7Uk0LdKqqr5rk2UCRXOk0vmQKGfc94nOJyQjouF9H/wCc9gECEYfONoAAAAASUVORK5CYII=)](https://scala-steward.org)

Pretty diffs for case classes. 

The library is published for Scala 2.12 and 2.13.

## Table of contents
- [goals of the project](#goals-of-the-project)
- [teaser](#teaser)
- [colors](#colors)
- integrations
  - [scalatest](#scalatest-integration)
  - [specs2](#specs2-integration)
  - [utest](#utest-integration)
  - [other](#other-3rd-party-libraries-support)
- [ignoring](#ignoring)
- [customization](#customization)
- [similar projects](#similar-projects)
- [commercial support](#commercial-support)

## Goals of the project

- human-readable case class diffs
- support for popular testing frameworks
- OOTB collections support
- OOTB non-case class support
- smaller compilation overhead compared to shapless based solutions (thanks to magnolia <3)
- programmer friendly and type safe api for partial ignore

## Teaser
Add the following dependency:

```scala
"com.softwaremill.diffx" %% "diffx-core" % "@VERSION@"
```

```scala mdoc
sealed trait Parent
case class Bar(s: String, i: Int) extends Parent
case class Foo(bar: Bar, b: List[Int], parent: Option[Parent]) extends Parent

val right: Foo = Foo(
    Bar("asdf", 5),
    List(123, 1234),
    Some(Bar("asdf", 5))
)

val left: Foo = Foo(
    Bar("asdf", 66),
    List(1234),
    Some(right)
)
 

import com.softwaremill.diffx._
compare(left, right)
```

Will result in:

![example](https://github.com/softwaremill/diff-x/blob/master/example.png?raw=true)


## Colors

When running tests through sbt, default diffx's colors work well on both dark and light backgrounds. 
Unfortunately Intellij Idea forces the default color to red when displaying test's error. 
This means that it is impossible to print something with the standard default color (either white or black depending on the color scheme).

To have better colors, external information about the desired theme is required.
Specify environment variable `DIFFX_COLOR_THEME` and set it to either `light` or `dark`.
I had to specify it in `/etc/environment` rather than home profile for Intellij Idea to picked it up.

If anyone has an idea how this could be improved, I am open for suggestions. 

## Scalatest integration

To use with scalatest, add the following dependency:

```scala
"com.softwaremill.diffx" %% "diffx-scalatest" % "@VERSION@" % Test
```

Then, extend the `com.softwaremill.diffx.scalatest.DiffMatcher` trait or `import com.softwaremill.diffx.scalatest.DiffMatcher._`.
After that you will be able to use syntax such as:

```scala mdoc:compile-only
import org.scalatest.matchers.should.Matchers._
import com.softwaremill.diffx.scalatest.DiffMatcher._

left should matchTo(right)
```

Giving you nice error messages:

## Specs2 integration

To use with specs2, add the following dependency:

```scala
"com.softwaremill.diffx" %% "diffx-specs2" % "@VERSION@" % Test
```

Then, extend the `com.softwaremill.diffx.specs2.DiffMatcher` trait or `import com.softwaremill.diffx.specs2.DiffMatcher._`.
After that you will be able to use syntax such as:

```scala mdoc:compile-only
import org.specs2.matcher.MustMatchers.{left => _, right => _, _}
import com.softwaremill.diffx.specs2.DiffMatcher._

left must matchTo(right)
```

## Utest integration

To use with utest, add following dependency:

```scala
"com.softwaremill.diffx" %% "diffx-utest" % "@VERSION@" % Test
```

Then, mixin `DiffxAssertions` trait or add `import com.softwaremill.diffx.utest.DiffxAssertions._` to your test code.
To assert using diffx use `assertEquals` as follows:

```scala mdoc:compile-only
import com.softwaremill.diffx.utest.DiffxAssertions._
assertEqual(left, right)
```

## Ignoring

Fields can be excluded from comparision by calling the `ignore` method on the `Diff` instance.
Since `Diff` instances are immutable, the `ignore` method creates a copy of the instance with modified logic.
You can use this instance explicitly.
If you still would like to use it implicitly, you first need to summon the instance of the `Diff` typeclass using
the `Derived` typeclass wrapper: `Derived[Diff[Person]]`. Thanks to that trick, later you will be able to put your modified
instance of the `Diff` typeclass into the implicit scope. The whole process looks as follows:

```scala mdoc:compile-only
case class Person(name:String, age:Int)
implicit val modifiedDiff: Diff[Person] = Derived[Diff[Person]].ignore[Person,String](_.name)
``` 

## Customization

If you'd like to implement custom matching logic for the given type, create an implicit `Diff` instance for that 
type, and make sure it's in scope when any `Diff` instances depending on that type are created.

If there is already a typeclass for a particular type, but you would like to use another one, you wil have to override existing instance. Because of the "exporting" mechanism the top level typeclass is `Derived[Diff]` rather then `Diff` and that's the typeclass you need to override. 

To understand it better, consider following example with `NonEmptyList` from cats.
`NonEmptyList` is implemented as case class so diffx will create a `Derived[Diff[NonEmptyList]]` typeclass instance using magnolia derivation.

Obviously that's not what we usually want. In most of the cases we would like `NonEmptyList` to be compared as a list.
Diffx already has an instance of a typeclass for a list. One more thing to do is to use that typeclass by converting `NonEmptyList` to list which can be done using `contramap` method.

The final code looks as follows:

```scala mdoc:nest
import cats.data.NonEmptyList
implicit def nelDiff[T: Diff]: Derived[Diff[NonEmptyList[T]]] = 
    Derived(Diff[List[T]].contramap[NonEmptyList[T]](_.toList))
```

And here's an example customizing the `Diff` instance for a child class of a sealed trait

```scala mdoc:silent
sealed trait ABParent
case class A(id: String, name: String) extends ABParent
case class B(id: String, name: String) extends ABParent

implicit val diffA: Derived[Diff[A]] = Derived(Diff.gen[A].value.ignore[A, String](_.id))
```
```scala mdoc
val a1: ABParent = A("1", "X")
val a2: ABParent = A("2", "X")

compare(a1, a2)
```

You may need to add `-Wmacros:after` Scala compiler option to make sure to check for unused implicits
after macro expansion.
If you get warnings from Magnolia which looks like `magnolia: using fallback derivation for TYPE`,
you can use the [Silencer](https://github.com/ghik/silencer) compiler plugin to silent the warning
with the compiler option `"-P:silencer:globalFilters=^magnolia: using fallback derivation.*$"`

## Other 3rd party libraries support

- [com.softwaremill.common.tagging](https://github.com/softwaremill/scala-common)
    ```scala
    "com.softwaremill.diffx" %% "diffx-tagging" % "@VERSION@"
    ```
    `com.softwaremill.diffx.tagging.DiffTaggingSupport`
- [eu.timepit.refined](https://github.com/fthomas/refined)
    ```scala
    "com.softwaremill.diffx" %% "diffx-refined" % "@VERSION@"    
    ```
    `com.softwaremill.diffx.refined.RefinedSupport`
- [org.typelevel.cats](https://github.com/typelevel/cats)
    ```scala
    "com.softwaremill.diffx" %% "diffx-cats" % "@VERSION@"    
    ```
    `com.softwaremill.diffx.cats.DiffCatsInstances`

## Similar projects

There is a number of similar projects from which diffx draws inspiration.

Below is a list of some of them, which I am aware of, with their main differences:
- [xotai/diff](https://github.com/xdotai/diff) - based on shapeless, seems not to be activly developed anymore
- [ratatool-diffy](https://github.com/spotify/ratatool/tree/master/ratatool-diffy) - the main purpose is to compare large data sets stored on gs or hdfs

## Commercial Support

We offer commercial support for diffx and related technologies, as well as development services. [Contact us](https://softwaremill.com) to learn more about our offer!

## Copyright

Copyright (C) 2019 SoftwareMill [https://softwaremill.com](https://softwaremill.com).
