---
title: Ad-hoc polymorphism in scala with generics
tags:
  - scala
  - leaky_abstractions
authorId: jose
date: 2016-11-10 00:00:00
---


[Ad-hoc polymorphism](https://en.wikipedia.org/wiki/Ad_hoc_polymorphism), also known as *operator overloading* allows a function to behave differently for different parameters. In this posts we will see some of the limitations of operator overloading when combined with generics and how to overcome them in an idiomatic way.

TL;DR:

* Scala supports ad-hoc polymorphism as a language feature
* But it has issues with generics because of type erasure
* Some times, the use of generics is not obvious (for instance, functions)
* Pattern matching is a really bad solution to this problem
* We can solve these issues with type classes

<!-- more -->

## The problem

I have a system with several combinations of connections and wanted to write a method that allowed for a unified way to execute a function inside a block that ensures that connections are correctly closed after the method execution. The idea was to write something like:

```
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

  db.closing { tx: OneConnection =>
    //do whatever that needs to be done here
  }
```

And then, if I need a different combination of connections, change only the type to `TwoConnections` and be done with it.

[This code, however, does not compile:](https://scalafiddle.io/sf/60vTlmn/2)

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

The problem is that the parameter is a function and a function is a generic (Function1[C,T]) and, thus, the compiler does not support method overloading with this parameter:

```
ScalaFiddle.scala:27: error: double definition:
def closing[T](f: ScalaFiddle.OneConnection => T): T at line 26 and
def closing[T](f: ScalaFiddle.TwoConnections => T): T at line 33
have same type after erasure: (f: Function1)Object
  def closing[T](f: TwoConnections => T): T = {
      ^ 
```

In my opinion, this is a leaky abstraction but there is nothing we can do about it and, thus, we need to find a solution.

## The obvious solution: Get rid of ad-hoc polymorphism

This is probably the most obvious and simple solution to the problem: If the compiler does not know which version of the method to call, then make the caller tell by using a different name: 

```scala
def closing1[T](f: OneConnection => T): T = {
  val conn = new OneConnection(new Connection())
  val res = f(conn)
  conn.close()
  res
}

def closing2[T](f: TwoConnections => T): T = {
  val conn = new TwoConnections(new Connection(), new Connection())
  val res = f(conn)
  conn.close()
  res
}

closing1 { c => 
  println(s"c1 is ${c.conn.isOpen}")
}
```

This solution has one big plus: We don't need to tell the type of the connection aggregator because it can be inferred from the method name (now there is only one possible type).

On the other hand, we have to remember the name of the method for every type (something that can be mitigated by following a convention like `closingOneConnection`, `closingTwoConnections`, etc.) and the solution is not very general because we need to write a new method with the same structure for every combination of connections (this can be mitigated by reusing private methods in the implementation of the different closingX)

## Multi method inspired pattern matching

As good coders, it is our duty to research the solution space to see if we can come up with a better solution or, at least, one that makes a different trade-off so that we can be sure that we have adopted the best posible solution to our problem.

One possible solution is to do something like what clojure does with multi methods and use [pattern matching](https://scalafiddle.io/sf/60vTlmn/3):


```scala
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

That is, the first time everything is ok but the second time the code fails at runtime. This is clearly an even worse situation than before (we have tricked the compiler into compiling a code that will not work in run time).

The problem is that, because of type erasure, both patterns are exactly the same in run time. Thus, the first execution is fine but the second is not because the type cast fails. This is one of the reasons why type casts are discouraged, because they might look like they are safe but they are not and we will not know until we hit the code path that makes it fail.

## Type classes to the rescue

Haskell is another regular source of inspiration when it comes to solve design problems, specially if we want to do it in a type safe way. In this case, we can define a type class for all the Connection combinations and see what happens:


```scala
object ConnectionAggregator {
  trait Closeable[T] {
    def create(): T
    def close(c: T): Unit
  }
  implicit object oneConnectionCloseable extends Closeable[OneConnection] {
    def create() = new OneConnection(new Connection())
  
    def close(c: OneConnection) = {
      c.conn.close()
    }
  }
  
  implicit object twoConnectionsCloseable extends Closeable[TwoConnections] {
    def create() = new TwoConnections(new Connection(), new Connection())
  
    def close(c: TwoConnections) = {
      c.conn1.close()
      c.conn2.close()
    }
  }
}

case class OneConnection(conn: Connection) 
case class TwoConnections(conn1: Connection, conn2: Connection) 

def closing[T,Q](f: Q => T)(implicit ev: ConnectionAggregator.Closeable[Q]): T = {
  val conn: Q = ev.create()
  val res = f(conn)
  ev.close(conn)
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

If you [run it](https://scalafiddle.io/sf/60vTlmn/5) the result is what we expected:

```
c1 is true
c2 is true
c3 is true
```

This solution does not force us to make the two connection aggregators inherit from the same trait (this is specially interesting if we cannot modify the source of the connection aggregator)

In this case, we are not overloading the method (there is only one method) and, thus, we are safe from the type erasure problem. The solution is general because, as there is only one method, the structure of the method is the same for all the combinations and it is very easy to abstract the parts of behavior that may change and put them in the type class.

The biggest downside of this solution is that it is a bit verbose if we only have two cases but it is easy to see how it will be less verbose than the 'obvious' solution as the number of connection aggregators grow.