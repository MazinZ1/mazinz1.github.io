---
layout: post
title: "Variance in Scala"
date: 2014/06/06
categories: [dev]
tags: [en, scala, pl]
---

This is my note on the variance section in [Coursera Functional Programming in Scala](https://class.coursera.org/progfun-004/lecture/83).

Scala supports two forms of polymorphism: *sub-typing* and *generics*. Not only does the type system needs to deal with both of them independently, it also needs to correctly resolve their interactions. 

## Type Bounds

In order to express the sub-typing relationship between type parameters, we need to introduce the concept of *type bounds*:

- Upper bound: S <: T means S is a *sub-type* of T,
- Lower bound: S >: T means S is a *super-type* of T, and 
- Mixed bound: S >: T1 <: T2 means S is any type between T1 and T2.

## Variance

Both functions and classes can be parameterized by type parameters, and *Variance* refers how the sub-typing relationship between them relates to the sub-typing relationship between their type parameters. 

Say `C[T]` is a parametric type and `A <: B`, there are three possible relationships between `C[A]` and `C[B]`:

- C is *covariant* if `C[A] <: C[B]`,
- C is *contravariant* if `C[A] >: C[B]`, and
- C is *nonvariant* otherwise.

There is a syntax in Scala to declare the variance of a type by annotating the type parameter:

```scala
class C[+A] { ... } // C is covariant
class C[-A] { ... } // C is contravariant
class C[A]  { ... } // C is nonvariant
```

## Functions

Assume `IntSet` is the base class for `Empty` and `NonEmpty`, i.e. `IntSet >: NonEmpty` and `IntSet >: Empty`. Consider two function types `type F1 = IntSet => NonEmpty` and `type F2 = NonEmpty => IntSet`, what's the sub-typing relationship between `F1` and `F2`?

Let's try to explain it intuitively using *the Liskov Substitution Principle*:

> If `A :< B`, then everything one can do with a value of type `B`, one should also be able to do with a value of type `A`.

For functions, it means that it is safe to replace a function with the type `F_super` with another one with the type `F_sub`. The function arguments of `F_super` should also be accepted by `F_sub`, which means `F_sub` should accept a more general type of arguments. For any expression that takes the result of `F_super` as an parameter, it should also accept the result of `F_sub`, which means `F_sub` should return a more specific type of result. Therefore, we have `F1 <: F2`. 

> Functions are contravariant in their argument type(s) and covariant in their result type.

In general, If `A1 >: A2` and `B1 <: B2` then `A1 => B1 <: A2 => B2`. The same goes for the functions that take functions as arguments. For example, `(A2 => B2) => B1 <: (A1 => B1) => B2` if `A1 >: A2` and `B1 <: B2`. And there is a nice trick to determine the variance of a type: *a position is covariant if it is on the left side of an even number of arrows*. 

## Classes

We know that functions are objects with `apply` method in Scala. The function type `T => U` is defined in term of the `Function1` trait as:

```scala
package scala
trait Function1[-T, +U] {
    def apply(x: T): U
}
```

The Scala compiler will check the variance of a class with similar rules as the variance rule of function types:

- *Covariant* type parameters can only appear in method results.
- *Contravariant* type params can only appear in method args.
- *Invariant* type params can appear everywhere

## Making a Class Covariant

> Roughly speaking, a type that accepts mutations of its elements should not be covariant.

```java
NonEmpty[] a = new NonEmpty[]{new NonEmpty(1, Empty, Empty)};
IntSet [] b = a;
b[0] = Empty;        // throw an exception
NonEmpty s = a[0]
```

In Java, Arrays are covariant, which means `NonEmpty[] <: IntSet[]`. Therefore, it is possible assign a ref of the subtype to a ref of the super-type, which is clearly problematic (see `IntSet [] b = a;`). Line 3 would throw an exception, which has runtime cost. 

```scala
val a: Array[NonEmpty] = Array(new NonEmpty(1, Empty, Empty))
val b: Array[IntSet] = a
b(0) = Empty
val s: NonEmpty = a(0)
```

In Scala, Arrays are not covariant, so the compiler would complain on Line 2 instead.
However, immutable types can be covariant, if some conditions on methods are met. 

- Covariant type params may appear in lower bounds of method type params
- Contravariant type params may appear in upper bounds of method

------

## Reference

- [Coursera Functional Programming in Scala](https://class.coursera.org/progfun-004/lecture/83)
- [Wikipedia](http://zh.wikipedia.org/wiki/%E5%8D%8F%E5%8F%98%E4%B8%8E%E9%80%86%E5%8F%98)

## History

- Jun 6, 2014 04:00:56 First commit

