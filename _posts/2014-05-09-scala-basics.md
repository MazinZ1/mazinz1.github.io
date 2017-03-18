---
layout: post
title : Scala基本语法与概念笔记
date: 2014/05/09
categories : [cn, dev]
tags : [scala]
---

### Partial application 和 Currying

和大部分函数式语言一样，Scala也支持partial application和currying。在应用函数时，你可以只提供部分的实际参数，而另外一部分参数则使用`_`来代替，这样会得到另外一个函数。`_`实际上起到一个匿名通配符 (unnamed wildcard) 的作用。


```scala
def adder(m: Int, n: Int) = m + n               //> adder: (m: Int, n: Int)Int
val add2 = adder(2, _: Int)                     //> add2  : Int => Int = <function1>
val add3 = adder(_: Int, 3)                     //> add3  : Int => Int = <function1>
```


而Currying则是说一个函数能够接受多个函数参数列表。从本质上，这样的函数可以通过使一个函数将另外一个函数作为返回值来实现。以下面为例，最外层的函数`sum`接受一个函数作为参数，并返回`sumInternal`作为结果。而在`sumInternal`的实现是依赖于`sum`的函数参数的。实际调用`sum`时的例子为<code>sum((x: Int) =&gt; x * x)(0, 2)</code>


```scala
def sum(f: Int => Int): (Int, Int) => Int = {
  def sumInternal(a: Int, b: Int) => Int = {
	if (a > b) 0
	else f(a) + sumInternal(a + 1, b)
  }
  sumInternal
}
```


按照这个的模式，我们可以用过嵌套定义函数来实现任意多的函数列表，只是略显繁琐。Scala提供了语法糖来简化Currying函数的定义。


```scala
def productSum(m: Double, x: Double)(n: Double, y: Double) = (m * x) + (n * y)
```


Partial application和Curring也是可以混合使用的。在`multiplyTwoSum`里我们对`productSum`两个参数列表都只传入了部分参数，而且得到的结果是<code>(Double, Double) =&gt; Double</code>，即是一个接受两个`Double`类型参数的函数。注意这和`oneMultiplySum`的区别：`oneMultiplySum`接受一个`Int`类型的参数，并返回一个<code>(Double, Double) =&gt; Double`</code>的函数作为结果。使用Partial application和Curring时，必须仔细观察得到的函数签名。我们也可以对一个partially applied的函数使用`curried`方法，得到它的curried版本。


```scala
val multiplyTwoSum = productSum(_: Double, 2)(_: Double, 2)
                                              //> multiplyTwoSum  : (Double, Double) => Double = <function2>
val fourteen = multiplyTwoSum(3, 4)             //> fourteen  : Double = 14.0
val oneMultiplySum = productSum(1, _: Int) _    //> oneMultiplySum  : Int => ((Double, Double) => Double) = <function1>
val curriedAdder = (adder _).curried            //> curriedAdder  : Int => (Int => Int) = <function1>
val five = curriedAdder(2)(3)                   //> five  : Int = 5
```


### 变长参数 (Variable length parameters)

Scala支持某个类型的变长参数，只要在函数参数的类型后面加上`*`就可以了。变长参数的类型是`Seq[T]`，因此大部分数据结构都能作为实际参数传入这样的函数中。

```scala
def capitalizeAll(args: String*) = {
  args.map { arg => arg.capitalize }
}
```

### 类 (Classes)

和Java或者C#不同，Scala里类的构造函数 (constructor) 并不是一个专门的方法 (所以构造函数这个翻译对于Scala的语境来说还挺不合适的，构造器可能更加适合一点？)，而是指类定义里面除去方法定义 (method definitions) 的所有部分。

```scala
// classes
class Calculator(val brand: String) {
//define method with def
def add(m: Double, n: Double) = m + n
def getBrand = brand
// define fields with val (or var)
val color: String =
  if (brand == "HP") {
    "black"
  } else {
    "white"
  }
}

val hpCalculator = new Calculator("HP")         //> hpCalculator  : basics.Calculator = basics$$anonfun$main$1$Calculator$1@664
```


抽象类，继承和重载都很正常。对于抽象类里只声明但是没有实现的定义，在实现时直接定义就可以。如果在抽象类里已经实现了的定义，那在子类里重新实现时要求使用`override`关键字。

```scala
// classes
// Inheritance
class ScientificCalculator(brand: String) extends Calculator(brand) {
  def log(m: Double, base: Double) = math.log(m) / math.log(base)
}

// overloading methods
class AdvancedScientificCalculator(brand: String) extends ScientificCalculator(brand) {
  def log(m: Int): Double = log(m, math.exp(1))
  override val color: String = "Golden"
}

// abstract classes; no implementation; can't create instances from abstract class
abstract class Shape {
  def getArea(): Int
}
```


### Traits

Traits定义了一系列的数据和方法，并且可以用来扩展或者mixin在你的类定义中。Traits和抽象类的特点和使用范围都有很相似的地方，所以选择的时候需要考虑它们不同的特点。一个类能够扩展多个traits，但是只能扩展一个父类。而traits并不能接受构造参数，而抽象类可以。

```scala
  trait Car {
    val brand: String
  }

  trait Sport {
    val Engine: String
  }

  class BMW extends Car with Sport {
    val brand = "BWM"
    val Engine = "Turbo"
  }
```


### 类型参数
定义类，函数，和traits时，都可以通过方括号`[]`引入类型参数。

```scala
  trait Cache[K, V] {
    def get(key: K): V
    def put(key: K, value: V)
    def delete(key: K)
    def remove[K](key: K)
  }
```


### apply 方法

Scala可以让你为类或者对象定义`apply`方法，来重载`()`运算符。

```scala
  class Foo {}
  object FooMaker {
    def apply() = new Foo
  }
  val newFoo = FooMaker()

  class Bar {
    def apply() = 0
  }
  val bar = new Bar
  bar()                                           //> res0: Int = 0
```


Scala里面类和对象能够使用同一个名称。所以通常我们会为一个类定义一个同名的对象 (companion object)，作为该类型的factories。

```scala
  class DummyClass(val foo: String) {
  }
  object DummyClass { // Classes and Objects can have the same name.
    def apply(foo: String) = new DummyClass(foo)
  }
  val c = DummyClass("foo")                                          //> res0: Int = 0
```


### 函数即对象

Scala里面的函数其实是一系列traits的集合。以一个<code>Int =&gt; Int</code>类型的函数为例，它扩展了`Function1[Int, Int]`这个trait。而且这个trait定义了`apply`函数，因为你可以用调用函数的语法来使用这个对象。实际上，Scala的提供了语法糖，让用户可以用<code>Int =&gt; Int</code>来表示`Function1[Int, Int]`。

注意这里的函数并不包括类里面定义的方法。

```scala
  object addOne extends Function1[Int, Int] {  // can be class AddOne extends (Int => Int)
    def apply(m: Int): Int = m + 1
  }
```


EOF

-------------------------------------------------

## Reference
- [Twitter Scala School](http://twitter.github.io/scala_school/)
