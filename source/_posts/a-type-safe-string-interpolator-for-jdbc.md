---
title: A type-safe string interpolator for JDBC
date: 2017-01-20
authorId: jose
tags:
  - scala
  - type_classes
---

There are plenty of frameworks for database access in Scala but for a side project I wanted something that pulled zero dependencies with it and, specially, something that could be used in any project no matter what library we were using in that project.

That meant, as there is nothing for JDBC in the Scala standard library, to hand code my database access against the JDBC APIs. As anyone who has programmed any JDBC can say, it only takes two or three hand-written queries before you start to feel the need to generalize all this boilerplate.

In my case, I was OK with hand-writing the SQL (no FRM or ORM) but, at the same time, I wanted my code to be checked at compile time as much as possible. I wanted a type safe SQL interpolator.

<!--more-->

# A very simple solution that just works

For our first iteration we will check the official [documentation](http://docs.scala-lang.org/overviews/core/string-interpolation.html) and write an interpolator based in the examples there:

```scala
import scala.collection.mutable.ListBuffer

case class SqlStatement(source:String, vars: Seq[Any])

object JdbcInterpolation {

  implicit class SqlInterpolation(val sc: StringContext) extends AnyVal {

    def sql(args:Any*): SqlStatement = {
      val strings = sc.parts.iterator
      val builder = new StringBuilder
      while (strings.hasNext) {
        builder.append(strings.next())
        if (strings.hasNext) {
        	//If there are more strings, a param goes here
          builder.append("?")
        } 
      }
      SqlStatement(builder.mkString.trim(), args)
    }
  }
}
```

This is very simple to use:

```scala
describe("Jdbc interpolation") {

  import JdbcInterpolation._
	
  it("should work if there are no substitutions") {
    val ps = sql"select 2+2 from dual"
    assert(ps.source == "select 2+2 from dual")
  }
	
  it("should replace a var with a ? and keep its value") {
    val id = 1
    val ps = sql"select name from users where id=$id"
	
    assert(ps.source == "select name from users where id=?")
    assert(ps.vars == Seq(id))
  }
	
  it("should keep the order of the variables") {
    val id = 1
    val city = "BCN"
    val ps = sql"select name from users where id=$id and city=$city"
	
    assert(ps.source == "select name from users where id=? and city=?")
    assert(ps.vars == Seq(id, city))
  }
}
```

As you can see, you don't need a fancy framework to get SQL interpolation. You can actually make this work with a plain old JDBC Prepared statement and call it a day:

```scala
   //ss is of type SqlStatement
   val ps = conn.prepareStatement(ss.source)
   ss.vars.zipWithIndex.foreach { case (v,i) =>
     ps.setObject(i,v)
   }
```

Of course, being Scala developers, we know we can do better and we expect something more from our compiler. Specifically, this solution can fail at run-time if the driver does not know what to do with our object because the interpolator accepts `Any` as parameter and we know that `setObject` only works for a given set of types.

What we need is a way to limit which types can be used in our interpolator so that, if we try to use a non-supported type in the interpolator we get a compile error. There are several solutions to this problem but we are going to use type classes because it is the most flexible one for this scenario.

# A type-safe solution with type classes

First of all, let's refresh our knowledge about type classes: 

A type class allows us to define classes (or groups, sets, or whatever you want to call them) of types that satisfy a given propiety. In our example, this property is "it can be used as a parameter in a statement". If we know how to set a parameter of type T then T belongs to our class, otherwise it doesn't.

In Scala, to define a type class, we define a trait with a type parameter (the type parameter is the type that belongs to the class) and, for every type that we want to belong to the class, we define an implicit object that acts as an evidence: If we can resolve an implicit that implements the trait for a type T then T belongs to our class, if we cannot resolve the implicit then it does not belong to the class.

In our case, we need to define a trait `JdbcParameterCompatible[T]` and, for every type `[T]` that can be used as a parameter, an evidence object.

```scala
import java.sql.PreparedStatement

trait JdbcParameterCompatible[T] {
  def set(ps:PreparedStatement, index:Int, value:T): Unit
}

object DefaultMappers {
  implicit val stringEvidence = new JdbcParameterCompatible[String] {
    override def set(ps: PreparedStatement, index: Int, value: String): Unit = {
      ps.setString(index, value)
    }
  }

  implicit val intEvidence = new JdbcParameterCompatible[Int] {
    override def set(ps: PreparedStatement, index: Int, value: Int): Unit = {
      ps.setInt(index, value)
    }
  }

}
```
The trait `JdbcParameterCompatible` defines a method (`set`) that sets a parameter in a statement. Now all we need is a way to make our interpolator require an evidence that a variable is "mappable" when we try to interpolate it.

Right now, our `sql` method cannot ask for the evidence because we are using `Any*` as the type of our parameter but there is nothing preventing us from defining a set of `sql` methods with different arity, each one of them asking for as many evidence parameters as arguments we have:

```scala
case class SetableParameter[T](value:T, mapper: JdbcParameterCompatible[T])

object JdbcInterpolation {

  implicit class SqlInterpolation(val sc: StringContext) extends AnyVal {

    def sql(): SqlStatement = {
      statement()
    }

    def sql[T](arg: T)(implicit mapper: JdbcParameterCompatible[T]): SqlStatement = {
      statement(param(arg))
    }

    def sql[T, U](arg1: T, arg2:U)(implicit mapper1: JdbcParameterCompatible[T], mapper2: JdbcParameterCompatible[U]): SqlStatement = {
      statement(param(arg1), param(arg2))
    }

    private def statement(args: SetableParameter[_]*): SqlStatement = {
      SqlStatement(statementWithPlaceHolders, args)
    }

    private def param[T](arg: T)(implicit mapper: JdbcParameterCompatible[T]): SetableParameter[T] = {
      SetableParameter(arg, mapper)
    }

    private def statementWithPlaceHolders: String = {
      val strings = sc.parts.iterator
      val builder = new StringBuilder
      while (strings.hasNext) {
        val str = strings.next()
        builder.append(str)
        if (strings.hasNext) {
          builder.append("?")
        }
      }
      builder.mkString.trim
    }
  }
}

```

Now, if we want our previous tests to compile, we need to bring into scope the DefaultMappers. If we don't, we will get an error like this one:

```
Error:(27, 16) could not find implicit value for parameter mapper2: com.agilogy.jdbc.interpolation. JdbcParameterCompatible[String]
      val ps = sql"select name from users where id=$id and city=$city"
```
And this is exactly what we wanted :)

# Summary
 
Of course, this solution is far from perfect as it requires a non-negligible amount of boilerplate (we need as many versions of the sql method as the maximum number of parameters we want to support) but we have a very powerful interpolator that is:

* Easy to use: No more writing '?' in the query and making sure that the args are in the right order
* Easy to extend: If we want to support new types all we need to do is bring a `JdbcParameterCompatible[T]` in scope
* Has no dependencies other than scala and jdbc.
* Type-safe: If we use the wrong type as parameter we will know it at compile-time

As an exercise, we can try to apply a similar type-class based approach to the result of the query (you can see a hint [here](https://gist.github.com/joseraya/0894cc79d1249bb0606306500e64699c)

You can find the code for this article here: [https://github.com/joseraya/jdbc-interpolator](https://github.com/joseraya/jdbc-interpolator)

