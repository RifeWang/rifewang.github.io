+++
draft = false
date = 2019-09-18T19:15:57+08:00
title = "从 V8 优化看高效 JavaScript【译】"
description = "从 V8 优化看高效 JavaScript【译】"
slug = ""
authors = []
tags = []
categories = ["Node.js"]
externalLink = ""
series = []
disableComments = true
+++

*文本翻译自:
https://blog.logrocket.com/how-javascript-works-optimizing-the-v8-compiler-for-efficiency*

---

理解 JavaScript 是如何工作的对于编写高效的 JS 大有帮助。

V8 执行 JS 分为三个阶段：

- 源代码转换为 AST 抽象语法树。
- 语法树转换为字节码：这个过程由 V8 的 Ignition 完成，2017年之前是没有的。
- 字节码编译成机器码：由 V8 的编译器 TurboFan 来完成。


第一个阶段并不是文本的讨论范围，第二三阶段对于编写优化 JS 有直接影响。

实际上第二三阶段是紧耦合的，它们都在 just-in-time（ JIT ）内运作。为了理解 JIT ，我们先回顾下源代码转换为机器码的两种方法：

1、解释器
解释器逐行转换和执行代码，其优点是易于实现和理解、及时反馈、更宽泛的编程环境，缺点也非常明显，那就是速度慢，慢的原因在于（1）反复解释的开销和（2）无法优化程序的各个部分。

换句话说，解释器在处理不同的代码段时无法识别重复的工作量。如果你通过解释器运行相同的代码 100 次，那么解释器将会翻译并执行相同的代码 100 次，其中不必要的重新翻译了 99 次。

解释器很简单、启动快速，但执行速度慢。

2、编译器
编译器在执行之前翻译所有的源代码。编译器更加复杂，但是可以进行全局优化（例如，共享重复代码），其执行速度也更快。

编译器更复杂、启动慢，但执行速度更快。

JIT 的作用就是尽可能结合解释器和编译器的优点，以使翻译代码和执行都能快速。

基本思想是尽可能避免重新翻译。首先，探测器通过解释器运行代码，在执行期间，探测器会追踪代码段并将其会被划分为 warm（运行少数几次） 和 hot（运行重复多次）。

JIT 把 warm 代码段直接丢给基准编译器，尽可能重用已编译的代码。

JIT 把 hot 代码段丢给优化编译器，其根据解释器收集来的信息（1）作出假设，（2）基于假设（比如，对象属性始终以特定顺序出现）进行优化。
然而，一旦假设不成立，优化编译器就会进行 deoptimization 去优化，就是丢弃优化的代码。


优化和去优化的周期是昂贵的。由于需要存储优化过的机器码和探测器的信息，JIT 引入了额外的内存成本。这种成本激发了 V8 的解释器 Ignition 。


Ignition 将 AST 转换为字节码，字节码序列被执行，其反馈信息被 inline caches 内联高速缓存。 反馈信息被用于（1）Ignition 随后的解释，和（2）TurboFan 推测性优化。
TurboFan 基于反馈推测性的优化将字节码转换为机器码。


---

## 如何优化你的 JavaScript


### 1、在构造函数中声明对象属性

改变对象的属性将会导致新的隐藏类：

```
class Point {
  constructor(x, y) {
    this.x = x;
    this.y = y;
  }
}

var p1 = new Point(11, 22);  // hidden class Point created
var p2 = new Point(33, 44);

p1.z = 55;  // another hidden class Point created
```

本来 p1 和 p2  应该使用的是同一个隐藏类，但是由于 p1.z 的原因将会导致它们使用不同的隐藏类，这将导致 TurboFan 的去优化，这是应该避免的。


### 2、保持对象属性排序不变

改变对象属性的排序也将会导致新的隐藏类：

```
const a1 = { a: 1 };  # hidden class a1 created
a1.b = 3;

const a2 = { b: 3 };  # different hidden class a2 created
a2.a = 1;
```

保持对象属性的排序有利于重用相同的隐藏类，效率更高。


### 3、注意函数的参数类型

函数参数类型的更改也将会导致去优化和重新优化：

```
function add(x, y) {
  return x + y
}

add(1, 2);  # monomorphic
add("a", "b");  # polymorphic
add(true, false);
add([], []);
add({}, {});  # megamorphic
```

比如这个函数，由于参数类型的易变将会导致编译器无法优化。


### 4、在 script 域声明类

不要在函数范围内定义类：

```
function createPoint(x, y) {
  class Point {
    constructor(x, y) {
      this.x = x;
      this.y = y;
    }
  }
  return new Point(x, y);
}

function length(point) {
  ...
}
```

这个函数每被调用一次，一个新的原型就被会创建，每个新的原型都会对应一个新的对象 shape ，这也是无法优化的。


### 5、使用 `for ... in`

`for ... in` 循环是 V8 引擎特别优化过的，可以快 4 到 6 倍。


### 6、不相关的字符不会影响性能

早期使用的是函数的字节计数来确定是否内联函数，但是现在使用的是 AST 的节点数量来确定函数的大小。这就是说，诸如空格、注释、变量名称长度、函数签名之类的不相关字符不会影响函数的性能。


### 7、Try / catch / finally 并不是毁灭性的

Try 以前会导致昂贵的优化和去优化循环，但是现在并不会导致明显的性能影响。



文本翻译有部分删减，全部内容可查看原始文章。