---
layout: post
title: On "Optimizing Node.js"
date: 2015-10-13 20:24:06
categories: [cn, dev]
tags: [javascript, node.js, v8]
---

前两天我在HN十大上看到了一篇[Node.js调优的小tips](https://medium.com/@c2c/nodejs-a-quick-optimization-advice-7353b820c92e)，大抵内容是“如果你有一个经常被调用的函数或者回调，你应该尝试把它缩短到600字符（包括空格）以下，这样v8就会尝试把该函数inline作为优化”。这个小技巧我之前其实在dotJS的一个[talk](https://youtu.be/FXyM1yrtloc)里也听说过，虽然听起来很美，但是我觉得使用之前还是要明白这背后的一些细节。

首先，文章或者talk里的观点都是“v8会自动inline源代码字符数在600以下的函数”，然而实际上更准确的说法应该是“v8绝对不会inline源代码字符数达到600的函数”。而且，源代码字符数并不是唯一的拒绝条件；即使在600字符以下，v8也有可能选择不将其inline。例如说，v8并不会inline超过196个AST节点的函数；而且在一个单独函数里，总计只能有196个AST节点能够被inlined[[1][A tour of V8: Crankshaft, the optimizing compiler]]。具体例子可见[代码](https://chromium.googlesource.com/v8/v8.git/+/master/src/hydrogen.cc#8280)。在实际场景中应用这个技巧的话，效果可能并没有上面的hello world例子来得显著。

不过既然v8提供了优化可能性，我们也是可以尽量尝试利用的。前端的JS代码一般发布前都会minified，所以已经利用了v8的这个优化可能性。需要注意的主要是用JS写的后端，也就是Node.js的代码。因为v8计算函数源代码时是包含了注释和空格的，所以把部分长注释移出函数之外也许是个不错的主意。这里插个题外话，也许会有同学想为什么v8不能直接把注释都去掉呢，这里有两个原因：其一是600字符数这个heuristic原本就是为了避免parse/tokenize过长的函数造成太大开销；其二是不少JS的代码都用了一些通过注释和`Function.prototype.toString()`来实现多行字符串(multiline string literal)的诡异技巧[[2][HN comments]]，例如下面来自[[3][Multi-line strings in JavaScript and Node.js]]的例子。因为这样的用例，注释不是说想去掉就去掉的……

```javascript
var html = (function () {/*  
  <!DOCTYPE html>  
  <html>  
    <body>  
      <h1>Hello, world!</h1>  
    </body>  
  </html>          
*/}).toString().match(/[^]*\/\*([^]*)\*\/\}$/)[1];

```

从相关的这个[issue](https://code.google.com/p/v8/issues/detail?id=3354)来看，这个600字符数的heuristic是不会在`Crankshaft`版本被修复了，但是下一代`TurboFan`应该会使用更准确的AST节点信息来计算是否inline的heuristic。另外文章和talk都提到了使用`max_inlined_source_size`来改变默认的600字符数限制，个人觉得进行这样的hacky调优时，还是需要结合benchmark。而且从[issue](https://code.google.com/p/v8/issues/detail?id=3354)来看，随意调大字符数限制并不一定会获得性能的提升。

其实除了在v8上面调优，我们还有很多别的优化机会，例如说mraleph提到`for (let x ...)`一般会比`for (var x ...)`慢上三倍[[3][mraleph的吐槽]]……如果你非要用`let`不可的话，你至少应该把`let`移到`for()`之外。

所以说，JS大法好！

[Multi-line strings in JavaScript and Node.js]: http://tomasz.janczuk.org/2013/05/multi-line-strings-in-javascript-and.html
[A tour of V8: Crankshaft, the optimizing compiler]: http://jayconrod.com/posts/54/a-tour-of-v8-crankshaft-the-optimizing-compiler
[v8: a tale of two compilers]: http://wingolog.org/archives/2011/07/05/v8-a-tale-of-two-compilers
[HN comments]: https://news.ycombinator.com/item?id=10375297
[mraleph的吐槽]: https://twitter.com/mraleph/status/653708835320922112


