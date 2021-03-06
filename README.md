# newfangled [![Join the chat at https://gitter.im/refried/newfangled](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/refried/newfangled?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge) [![Build Status](https://travis-ci.org/refried/newfangled.svg?branch=master)](http://travis-ci.org/refried/newfangled)

An sbt plugin enabling the latest/greatest scala plugins, libraries, and compiler options.

Any feedback is appreciated!

## Auto-enabled compiler plugins
* https://github.com/non/kind-projector 0.6.0 
  * Compiler plugin for making type lambdas (type projections) easier to write
* https://github.com/mpilquist/local-implicits 0.3.0 
  * Compiler plugin which provides syntax for working with locally declared implicit values

## Auto-enabled libraries
* https://github.com/typelevel/machinist 0.3.0 
  * Spire's macros for zero-cost operator enrichment
* https://github.com/mpilquist/simulacrum 0.3.0 
  * First class syntax support for type classes in Scala
* https://github.com/scalamacros/paradise 2.0.1 (used by simulacrum)

## Available libraries
* https://github.com/scalaz/scalaz 7.1.3 (via `libraryDependencies += scalaz("core")`, etc.) 
  * An extension to the core Scala library for functional programming
* https://github.com/non/cats snapshot (via `catsSnapshot("core")`, etc.)
  * Lightweight, modular, and extensible library for functional programming
* https://github.com/julien-truffaut/Monocle 1.1.1 (via `monocle`)
  * Optics library for Scala
* scala-reflect (via `scalaReflect`)

## Default project settings
```scala
scalaVersion := "2.11.7",
scalacOptions ++= Seq(
  "-deprecation",
  "-encoding", "UTF-8",       // yes, this is 2 args
  "-feature",
  "-language:existentials",
  "-language:higherKinds",
  "-language:implicitConversions",
  "-unchecked",
  "-Xlint",
  "-Yno-adapted-args",
  "-Ywarn-dead-code",        // N.B. doesn't work well with the ??? hole
  "-Ywarn-numeric-widen",
  "-Ywarn-value-discard",
  "-Xfuture"
)
```

## Available project settings
The following settings are made available to your build definition (but not automatically enabled):
```scala

// enable reflection
val scalaReflect = libraryDependencies += "org.scala-lang" % "scala-reflect" % scalaVersion.value

// import scalaz
def scalaz(module: String): ModuleID = "org.scalaz" %% s"scalaz-$module" % "7.1.3"

// import cats
def catsSnapshot(module: String) = "org.spire-math" %% s"cats-$module" % "0.1.0-SNAPSHOT"
def cats(module: String, gitReference: String = "#master"): ProjectRef // depend on particular cats git ref

// enable monocle
val monocle: Seq[Setting[...]] // monocle core, generic, macros v1.1.1

// more scalac options
val noPredef = scalacOptions += "-Yno-predef"
val noImports = scalacOptions in compile += "-Yno-imports" // will break REPL if not restricted to compile

val warnUnusedImport = scalacOptions in compile += "-Ywarn-unused-import"
val fatalWarnings = scalacOptions in compile += "-Xfatal-warnings"
val picky = Seq(warnUnusedImport, fatalWarnings)
```

## Example Usage:
`project/plugins.sbt`:
```scala
addSbtPlugin("net.arya" % "newfangled" % "0.2.0")
```

`build.sbt`
```scala
organization := "net.arya"
name := "example"

// newfangled helpers
libraryDependencies += scalaz("effect")
monocle
```

`Test.scala`
```scala
package net.arya

import monocle.macros.Lenses
import monocle.syntax._

import scala.language.postfixOps

import scalaz.effect.{IO, SafeApp}
import scalaz.syntax.bind._

@Lenses case class Foo(a: Int, b: String)

import simulacrum._
import Semigroup.ops._

@typeclass trait Semigroup[@specialized A] {
  @op("|+|", alias=true)
  def append(x: A, y: A): A
}

object Semigroup {
  val intAdd = new Semigroup[Int] { def append(x: Int, y: Int) = x + y }
  val intMult = new Semigroup[Int] { def append(x: Int, y: Int) = x * y }
  implicit val string = new Semigroup[String] { def append(x: String, y: String) = x + y }
}

object Test extends SafeApp {

  val foo1 = Foo(3, "Hello")
  val foo2 = Foo(4, "")

  implicit def fooSemigroup(implicit i: Semigroup[Int], s: Semigroup[String]): Semigroup[Foo] =
    new Semigroup[Foo] { def append(x: Foo, y: Foo) = Foo(x.a |+| y.a, x.b |+| y.b) }

  override def runc =
    imply(Semigroup.intMult) {
      IO.putStrLn(foo1 |+| foo2 toString)
    } >> imply(Semigroup.intAdd) {
      IO.putStrLn(foo1 |+| (foo2 applyLens Foo.b modify (_ + "!!!")) toString)
    }
}
```
Output:
```
Foo(12,Hello)
Foo(7,Hello!!!)
```

## Special thanks
Thanks to pfn and OlegYch__ in #sbt, tpolecat for the `scalacOptions`, and all the amazing library and plugin developers.
