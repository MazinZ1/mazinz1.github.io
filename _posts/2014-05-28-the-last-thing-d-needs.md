---
layout: post
title : The Last Thing D Needs
date: 2014/05/28
categories : [cn, dev]
tags : [c++]
---


“I am not here to bash C++” — Scott Meyers, 2014

今年DConf把Scott Meyers (Effetive C++系列的作者) 请过来做[演讲](http://www.ustream.tv/recorded/47947981)。打开前我还疑惑地以为Meyers难道开始D语言的传道了，可是一开始看到他一脸假正经地宣称自己不是过来羞辱C++的，我就明白这是又一出挂羊头卖狗肉的C++鞭笞秀了XDDDD。抱着娱乐的心情看完，却发现感想远超预期。也许是因为场合不同，这次Meyers的演讲完全跳出了他之前”Meyers knows all”的老调，而是通过自己对C++的理解进行了相当有趣的吐槽，而且中间插的段子也是笑点十足。强烈推荐给关注近期C++发展的程序员。

下面是对这次演讲的一些笔记。

———

## C++ 的坑

C++鞭笞秀的标准起手式就是陈列语言大坑，这次也不例外。

### 初始化 Initialization

```cpp
int x1;                // unknown
int x2;                // (at global scope) 0
static int x3;         // 0

{
	int x4;            // unknown
	…
}

int a1[100];           // unknown
int a2[100];           // (at global scope) 0
```

变量的初始化是个很基础的话题。会得到上面的结果的原因也简单：未被初始化的全局变量和静态变量都会被放去BSS段(BSS segment)，而BSS段的数据都是0初始化的。背后的道理也很单纯：为这些变量的初始化并不会带来运行时的代价(runtime cost)。C++标准(准确地说，C标准)的原则是，只要不损害运行时的性能，编译器都很愿意为程序员提供方便。然后这样就带来了变量初始化行为不一致的问题。尽管现在所有的代码规范都强烈抵制全局变量和静态变量，并且一再强调定义变量时初始化，但是只要是人都会有失误的时候。而标准作出了取舍：它以程序员发生错误的可能性作为代价，换取了更好的运行性能。背后的经济学原理就是机器的时间要比程序员的时间更值钱。

这样的取舍也有一定的道理，而如果打之为止的话，这原则也很好地解释了一致性。可是我们C++有容器，而容器是默认初始化为0的，即使这带来了运行时的代价……

```cpp
std::vector<int> a(100);  // 0
```

### 类型推导 Type Deduction

```cpp
const int cx = 0;

auto my_cx1 = cx;                 // int
decltype(cx) my_cx2 = cx;         // const int
```

在这里可以解释为，`my_cx1`是另外一个独立的变量，它使用了`cx`作为初始值，并不代表它应该有完全一样的类型。使用`decltype`，就代表你想完全照用`cx`的类型。这两条是不同的规则，而符合逻辑，没有问题。

```cpp
template<typename T>
void f1(T param);

f1(cx);                           // T: int

template<typename T>
void f2(T& param);

f2(cx);                           // T: const int
```

这里`f1`的逻辑和`auto`类似。`T`在函数里是一个独立的新变量，因此我们只需要把`cx`的数据复制到`T`，然而不需要要求`T`也带有常量性(constness)。而`f2`中，我们引用了某块内存里的某个变量，因此`T`必须保留它原来的常量性，这也很符合逻辑

```cpp
template<typename T>
void f3(T&& param);

f3(cx);                           // T: const int &
```

注意<code>T&amp;&amp;</code>在C++ 11前的是非法的：你不能为一个引用再生成一个引用 (you are not allowed to take a reference to a reference)。

然而C++ 11引入了新的*reference collapsing rules*

- <code>A&amp; &amp; -&gt; A&amp;</code>
- <code>A&amp; &amp;&amp; -&gt; A&amp;</code>
- <code>A&amp;&amp; &amp; -&gt; A&amp;</code>
- <code>A&amp;&amp; &amp;&amp; -&gt; A&amp;&amp;</code>

根据模版参数推导规则，`cx`是类型为`A`的左值(lvalue)，因此`T`会根据上面的reference collapsing rules得到<code>A&amp;</code>的类型。这是为了实现*perfect forwarding*的一个hack，具体细节可以参考[Perfect Forwarding: The Solution](http://thbecker.net/articles/rvalue_references/section_08.html)。(Meyers: still not here to bash XD!)

```cpp
auto lam = [cx] { cx = 10; }    // error!

class UpToTheCompiler {
private:
    const int cx;
}
```

当考虑到lambda时，事情就变得更有乐趣了。当我们用`capture`时，编译器实际上会被背后自动生成一个类，并把`cx`从原来的scope里复制到这个类的同名数据成员中。尽管我们逻辑上觉得，这样的capture不过也是简单的复制数据，所以类里的数据成员`cx`的类型应该还是`int`。但是实际上数据成员的类型是`const int`，所以上面的例子是不是编译的……这也是C++唯一的地方(据Meyers宣称)，当你尝试把一个变量的内容复制到一个新的独立变量时，会保存原变量的常量性的。

```cpp
auto lam = [cx = cx] { cx = 10; }    // error!

class UpToTheCompiler {
private:
    int cx;    // but acts like const int
public:
    void operator()() const
    { cx = 0; }
}
```

而C++ 14新加入的*lambda captures expressions*使我们可以用`cx`初始化一个lambda内的`cx`变量。而生成类里的`cx`数据成员也会变成`int`。但是很遗憾，即使是这样，你也不能为lambda内的`cx`进行赋值。因为这个类还会自动生成一个`()`操作符函数，而这个函数是`const`的，所以你不能对数据变量`cx`进行改变……怎么解决这个问题？很简单，你只需要如下：(Meyers: still not here to bash XDDDD!!!!!)

```cpp
auto lam2 = [cx = cx]() mutable { cx = 10; }

class UpToTheCompiler {
private:
    int cx;
public:
    void operator()()
    { cx = 0; }
}
```

总结上面，对于`const int cx = 0`来说，根据上下文总共只有6种类型推导的方式:

- `auto` &amp; `lambda int capture` = `int`
- `decltype` = `const int`
- `template(T param)` = `int`
- <code>template(T&amp; param)` = `const int</code>
- <code>template(T&amp;&amp; param)` = `const int &amp;</code>
- `lambda (by-value capture)` = `const int`

```cpp
auto x1 = 0;            // int
auto x2(0);             // int
auto x3 = { 0 };        // initializer_list<int>
auto x4 { 0 };          // initializer_list<int>

template<typename T>
void f(T param);

f({0});                 // error! {0} has no type
```

最后这个例子说明了`auto`其实是有自己的一套推导法则的，只是Meyers也没办法解释这背后的逻辑了……

### Inheritance 继承

```cpp
template<typename T>
class Base {
public:
    void doBaseWork();
};

template<typename T>
class Derived: public Base<T> {
public:
    void doDerivedWork()
    {
        doBaseWork();
    }
};
```

C++对于模板内符号的*two-phase name lookup*应该是C++ gotchas里的经典场景了。
上面的代码是无法编译的，编译器会要求你显式地注明`doBaseWork()`依赖于类型参数`T`。<code>this-&gt;doBaseWork()</code>或者`Base::doBaseWork()`都能通过编译。但是之后出现了`Base`对于`int`类型的特化(specialization)的话，在实体化(instantiating)模版时还是一样会出错。

```cpp
template<>
class Base<int> {};

Derived<int> d;
d.doDerivedWork();    // fail
```

这个问题另外一个有趣的变体是，如果不幸打错字，將`doBaseWork`打成`doBaseWrk`的话，会怎么样。

```cpp
template<typename T>
class Base {
public:
    void doBaseWork();
};

template<typename T>
class Derived: public Base<T> {
public:
    void doDerivedWork()
    {
        doBaseWrk();
    }
};
```

这不是一个容易回答的问题。如果编译器认为这是笔误，第一个phase就诊断这是一个错误的话，之后就不能通过特化来提供`doBaseWrk`函数了。如果编译器选择推迟name lookup直到实体化，那么我就就把诊断库(library)的责任推给了用户。

C++的解决方案是：模板的作者拥有选择权：

- 只写`doBaseWrk()`的话，那么编译器在解释（parse）模板时就进行符号查找(lookup name)。
- <code>this-&gt;doBaseWrk()</code>或者`BasedoBaseWrk()`的话，编译器会推迟到实体化模板时才进行符号查找。

### Computational Complexity 计算复杂度

```cpp
auto it1 = std::binary_search(v.begin(), v.end(), 10);  // O(lg n)

auto it2 = std::binary_search(li.begin(), li.end(), 10); // O(n); standard: O(lg n)
```

为什么标准中二分查找链表的复杂度是`O(lg n)`？因为这个复杂度是基于比较的次数的……

### Specifications

C++标准中规定了，除了满足基本容器的要求外，连续容器(sequence container)还需要满足另外的16项要求。可是看一下STL里面的5种连续容器，它们是否都满足这16项要求呢？

- array: no
- deque: yes
- forward_list: 只满足了1条……
- list: yes
- vector: yes

-------------

## 总结

Fred Brooks提到软件设计中的复杂性可以分为两种:

- Essential complexity
- Accidental complexity

Meyers认为在编程语言设计中，essential complexity来自设计方向的冲突:

- Simplicity and regular vs. expressiveness.
- Abstraction and portability vs. efficiency.
- New approaches vs. compability with legacy systems.
- Expressiveness vs. ability to issue good diagnostics.

然而C++本身在设计时带来了很多不必要的accidental complexity，例如在演讲里提到的：

- `int` *有时*会被初始化为0
- By-value lambda capture *有时*会保持被捕捉变量的常量性
- `mutable lambdas` 必须声明参数列表，而non-mutable的不用
- 花括号初始化(braced initializer) *有时*会带有类型
- 计算复杂度的保证*有时*很微妙
- 容器的API命名不完全一致
- `sort` *有时*是stable的
- 容器的要求在*有时*是被要求的
- 很多很多

C++主要的问题在于：

- 太复杂以致难以被修复
- 需要兼容大量陈旧代码
- 用户和标准委员会并没有很大的兴趣/精力去修复这些问题

对于Meyers来说，他不了解D，不设计编程语言，不进行软件开发(很多年)，偶尔干干咨询(很多年前)，经常作为培训的教师，不停地给人解释概念。因此对于他来说，容易解释的东西就是好的 (easy to explain = good)。然而从*piecemeal design philosophy*来说，C++的符合要求的：每条语言规则单独来说都是容易被解释或者辩解的。然后从*holistic design pholosophy*的角度，每条语言规则除了能被单独解释外，它在别的规则的上下文里，也应该同样易于解释。

常见的C++设计的正当化理由有：
1. 与C兼容
2. 最大化效率
3. 不用就不花钱 (you don't pay for what you don't use)
4. 信任用户
5. 预防可能的用户错误
6. 让功能更泛用
7. 不要限制编译器
8. 相比起编译器开发者，更关注用户
9. 保持后向兼容性
10. 与其他功能保持一致

这些理由单独使用时，都能够让某个语言特性得到正当化。然而4、5、7和8实际上可以为所有语言特性进行辩解，而它们本身就是容易相互冲突的。

然而作为语言用户，使用C++很容易让用户把大量精力花费在`tool use`的层面，然而相对花费在`tool application`的时间就少了很多。Meyers用自己的经验来说，他写了很多那么多本C++系列的书籍，大部分的内容都关注`tool use`，而真正使用语言特性来实现high-level ideas的`tool application`实际上很少。因为他认为D语言最不需要的东西，就是他这样的人。

另外某个听众提问了最新的Effective Modern C++推出时间，Meyers预计会在今年的9月份推出。但是他不认为D会议的听众会需要这本书 XDDDD。

--------------

# Reference

- [Keynote: The Last Thing D Needs - Scott Meyers](http://www.ustream.tv/recorded/47947981)
- [Universal References in C++11](https://isocpp.org/blog/2012/11/universal-references-in-c11-scott-meyers)
- [C++ Rvalue References Explained](http://thbecker.net/articles/rvalue_references/section_01.html)
- [Dependent name lookup for C++ templates](http://eli.thegreenplace.net/2012/02/06/dependent-name-lookup-for-c-templates/)
