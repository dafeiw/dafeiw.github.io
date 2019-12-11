---
title: Generic in Scala
date: 2019-11-27 23:36:01
tags: Scala
---

# Variance

```scala
class Animal
class Cat extends Animal
class Dog extends Animal
```

If Cat extends Animal, does List[Cat] extends List[Animal]?

1. Answer: yes ---> COVARIANCE

Define a covariant list:

```scala
class CovariantList[+A]
val animal: Animal = new Cat
val animalList: CovariantList[Animal] = new CovariantList[Cat]
```
<!--more-->
Now, a question: animalList.add(new Dog). Is it ok???

Answer: Adding a dog into a list of cat returns a list of animal

So

```scala
class MyList[+A] {
    def add[B >: A](elem: B): MyList[B] = ???
}
```


2. Answer: no ---> INVARIANCE

Define a invariant list:

```scala
class InvariantList[A]
val invariantAnimalList: InvariantList[Animal] = new InvariantList[Cat]  <-- error
```

3. Answer: no ---> CONTRAVARANCE

```scala
class ContravariantList[-A]
val contravariantList: ContravariantList[Cat] = new ContravariantList[Animal]
```

```scala
class Trainer[-A]
val trainer: Trainer[Cat] = new Trainer[Animal]
```

A trainer of animal is also a trainer of cat.

# Bounded Types

1. Upper bound
```scala
class Cage[A <: Animal](animal: A)
val cage = new Cage(new Dog)

class Car
val cage = new Cage(new Car)  <--- error !!
```

class cage only accepts type of A which is subtype of Animal

2. Lower bound

```scala
class Cage[A >: Animal](animal: A)
```

class cage only accepts type of A which is supertype of Animal


