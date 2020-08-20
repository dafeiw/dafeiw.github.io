---
title: Typeclasses
date: 2020-08-05 23:47:22
tags: Scala
---

## Type class and instance

In scala, a type class is represented by a trait with at least one type parameter. When we implement the type class, we provides a concret type we care about. The type can be from the Scala standard library and types from our domain model.

```scala
trait JsonWriter[T] {
    def write(value: T): Json
}


final case class Person(name: String, email: String)

object JsonWriterInstances {
    implicit val stringWriter: JsonWriter[String] =
        new JsonWriter[String] {
            def write(value: String): Json = JsString(value)
        }

    implicit val personWriter: JsonWriter[Person] =
        new JsonWriter[Person] {
            def write(value: Person): Json =
                JsObject(Map(
                    "name" -> JsString(value.name),
                    "email" -> JsString(value.email)
                ))
        }
// etc...
}
```
<!--more-->
## Use of type class instance

The implicit type class instance is used by the method which accepts them as an implicit parameter. The implicit type class instance provide a specific functionality.

### Interface objects

Place methods in a singleton object to use the type class:

```scala
object Json {
    def toJson[A](value: A)(implicit w: JsonWriter[A]): Json = w.write(value)
}
```

we import any type class instances we care about and call the relevant method:

```scala
import JsonWriterInstances._
Json.toJson(Person("Dave", "dave@example.com"))
```

### Interface Syntax

We enahace the value type:

```scala
object JsonSyntax {
    implicit class JsonWriterOps[A](value: A) {
        def toJson(implicit w: JsonWriter[A]): Json =
            w.write(value)
        }
}
```

We import it alongside the type class instance
```scala
import JsonWriterInstances._
import JsonSyntax._
Person("Dave", "dave@example.com").toJson
```

### The implicitly Method

## Where to place type class instance
1. by placing them in an object such as JsonWriterInstances;
2. by placing them in a trait;
3. by placing them in the companion object of the type class;
4. by placing them in the companion object of the parameter type.

### Two ways to define a type class instance

1. by defining concrete instances as implicit vals of the required type;
2. by defining implicit methods to construct instances from other type class instances.

The parameter needs to be implicit.
```scala
implicit def optionWriter[A](implicit writer: JsonWriter[A]): JsonWriter[Option[A]] =
    new JsonWriter[Option[A]] {
        def write(option: Option[A]): Json =
            option match {
                case Some(aValue) => writer.write(aValue)
                case None => JsNull
            }
}
```

### Variance

Variance is all about substitue one value for another. Covariance is generally for output. List[+T] means you can subsitute List[Apple] for List[Fruit]. Contravariance is generally for input. Consier a method `def format(value: T, writer: JsonWriter[T]): Json`, which combination of value and writer can you pass to format?

When the compiler searches for an implicit it looks for one matching the type or subtype. 

### Summary

* Define a type class - a trait takes at least one generic parameter
* Creat a type class instance - implicit
* Define a method to use the implicit type class instance

We can add new behaviour to closed data type without using inheritance, and without accessing to original source code of those types. 