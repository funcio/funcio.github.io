---
title: Ad-hoc polymorphism in scala
authorId: jose
date: 2016-11-10
tags:
  - scala
  - leaky_abstractions
 
---

[Ad-hoc polymorphism](https://en.wikipedia.org/wiki/Ad_hoc_polymorphism), also known as *operator overloading* allows a function to behave differently for different parameters. In this posts we will see some of the limitations of operator overloading and how to overcome them in an idiomatic way.

TL;DR:

* Scala supports ad-hoc polymorphism as a language feature
* But it has issues with generics because of type erasure
* Some generics are not obvious (for instance, functions)
* Pattern matching is a really bad solution to this problem
* We can solve these issues with type classes

<!-- more -->

## Method overloading in Scala
Scala supports method overloading and, thus, this should not be a problem for us:

```scala
class Connection {
  var isOpen = true
  def close() {
    isOpen = false
  }
}

case class OneConnection(conn: Connection)
case class TwoConnections(conn1: Connection, conn2: Connection)

def close(conn: OneConnection): Unit = {
  conn.conn.close()
}

def close(conn: TwoConnections): Unit = {
  conn.conn1.close()
  conn.conn2.close()
}

val c1 = new Connection()
close(new OneConnection(c1))
println(s"c1 is ${c1.isOpen}")

val c2 = new Connection()
val c3 = new Connection()
close(new TwoConnections(c2, c3))

println(s"c2 is ${c1.isOpen}")
println(s"c3 is ${c1.isOpen}")

```

As you can see in this [fiddle](https://scalafiddle.io/sf/60vTlmn/0) this code works and prints 

```
c1 is false
c2 is false
c3 is false
```

## Runtime type erasure

So far, so good but what happens if we pass a function as a parameter?

```scala
case class OneConnection(conn: Connection) {
  def close(): Unit = {
    conn.close()
  }
}
case class TwoConnections(conn1: Connection, conn2: Connection) {
  def close(): Unit = {
    conn1.close()
    conn2.close()
  }
}

def closing[T](conn: OneConnection)(f: OneConnection => T): T = {
  val res = f(conn)
  conn.close()
  res
}

def closing[T](conn: TwoConnections)(f: TwoConnections => T): T = {
  val res = f(conn)
  conn.close()
  res
}
```
It still [works](https://scalafiddle.io/sf/60vTlmn/1) so, what is the big deal? Well, this code works because it has a non-generic argument to disambiguate, but what happens if the only parameter is a function?

```scala
def closing[T](f: OneConnection => T): T = {
  val conn = new OneConnection(new Connection())
  val res = f(conn)
  conn.close()
  res
}

def closing[T](f: TwoConnections => T): T = {
  val conn = new TwoConnections(new Connection(), new Connection())
  val res = f(conn)
  conn.close()
  res
}

closing { c: OneConnection => 
  println(s"c1 is ${c.conn.isOpen}")
}
```

[This code](https://scalafiddle.io/sf/60vTlmn/2) does not compile!

```
ScalaFiddle.scala:27: error: double definition:
def closing[T](f: ScalaFiddle.OneConnection => T): T at line 26 and
def closing[T](f: ScalaFiddle.TwoConnections => T): T at line 33
have same type after erasure: (f: Function1)Object
  def closing[T](f: TwoConnections => T): T = {
      ^ 
```
And indeed the compiler is right. Although the two types are different, because of the erasure, we cannot overload the method and, thus, we cannot have ad-hoc polymorphism in this case. In my opinion, this is a leaky abstraction (even if it is so for a good reason).

## Multi method inspired pattern matching

At this point, we need an alternative. Inspired by clojure multi methods, we might consider [pattern matching](https://scalafiddle.io/sf/60vTlmn/3):

```scala
trait ConnectionAggregator {
  def close(): Unit
}

case class OneConnection(conn: Connection) extends ConnectionAggregator {
  def close(): Unit = {
    conn.close()
  }
}
case class TwoConnections(conn1: Connection, conn2: Connection) extends ConnectionAggregator {
  def close(): Unit = {
    conn1.close()
    conn2.close()
  }
}

def closing[T,Q<: ConnectionAggregator](f: Q => T): T = {
  val conn: Q = f match {
    case f: (OneConnection => T) =>
      new OneConnection(new Connection()).asInstanceOf[Q]
    case f: (TwoConnections => T) =>
      new TwoConnections(new Connection(), new Connection()).asInstanceOf[Q]
  }
  val res = f(conn)
  conn.close()
  res
}

closing { c: OneConnection => 
  println(s"c1 is ${c.conn.isOpen}")
}

closing { c: TwoConnections =>
  println(s"c2 is ${c.conn1.isOpen}")
  println(s"c3 is ${c.conn2.isOpen}")
}
```
The result is:
```
c1 is true
ERROR: undefined
```
That is, the first time everything is ok but the second time the pattern matches the first case instead of the second because of the erasure and, thus, we are even worse than before because the code does compile but does not work (this is one of the reasons why type casts are discouraged, sometimes they look safe but they are not).

## Type classes to the rescue

One possible solution is to use a type class and use the evidence parameter to disambiguate:

```scala
object ConnectionAggregator {
  trait Closeable[T<: ConnectionAggregator] {
    def create(): T
  }
  implicit object oneConnectionCloseable extends Closeable[OneConnection] {
    def create() = new OneConnection(new Connection())
  }
  implicit object twoConnectionsCloseable extends Closeable[TwoConnections] {
    def create() = new TwoConnections(new Connection(), new Connection())
  }
}

def closing[T,Q<: ConnectionAggregator](f: Q => T)(implicit ev: ConnectionAggregator.Closeable[Q]): T = {
  val conn: Q = ev.create()
  val res = f(conn)
  conn.close()
  res
}
```
If you [run it](https://scalafiddle.io/sf/60vTlmn/4) the result is what we expected:

```
c1 is true
c2 is true
c3 is true
```

