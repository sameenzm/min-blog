---
title: 前端面试题集
date: 2019-04-28 11:42:25
tags:
- javascript
---

前端面试题集，持续更新。
有不对的地方欢迎指出，感谢


<!-- more -->

# Javascript篇

关键词目录
* [原型和原型链](#原型和原型链)
* [作用域和作用域链](#作用域和作用域链)
* [闭包](#闭包)
* [事件委托](#事件委托)
* [事件循环](#事件循环)
* [同源策略](#同源策略)
* [数组去重](#数组去重)
* [web安全](#web安全)


# CSS篇

关键词目录
* [BFC](#BFC)
* [伪类和伪元素](#伪类和伪元素)
* [垂直居中](#垂直居中)
* [两列布局](#两列布局)
* [三列布局](#三列布局)
* [事件循环](#事件循环)
* [同源策略](#同源策略)
* [数组去重](#数组去重)
* [web安全](#web安全)



### 原型和原型链
> 说说你对原型和原型链的理解

在javascript中，所有对象都有一个 `_proto_` 属性（属性值是普通对象）。其中函数对象都有一个 `prototype` 属性（属性值也是普通对象）。
所有对象的 `_proto_` 属性指向它的构造函数的`prototype`属性。
这个构造函数的`prototype`就是原型。
原型的好处是 某对象 的所有实例对象 都可共享它所包含的属性和方法。

原型链：
当在对象上找某属性时，会先找对象本身。
若没有则找其 `_proto_` ，即其构造函数的原型（`prototype`）
若构造函数的`prototype`上也没有，则找其`_proto_`，即再上一级的prototype
最后的源头，构造函数是Object，找Object.prototype，
如果Object.prototype有就有，没有就是null
原型链的最上层：  Object.prototype.__proto__ === null
这样一直往上找，构成了一个链式结构，就是原型链了

>补充：
构造函数、原型和实例的关系：
每个构造函数都有一个原型对象（xxx.prototype），原型对象都包含一个指向构造函数的指针，
而实例都包含一个指向原型对象的内部指针。
那么，假如我们让原型对象等于另一个类型的实例，结果会怎么样呢？显然，此时的原型对象将包含一个指向另一个原型的指针，
相应地，另一个原型中也包含着一个指向另一个构造函数的指针。
假如另一个原型又是另一个类型的实例，那么上述关系依然成立，如此层层递进，就构成了实例与原型的链条。这就是所谓原型链的基本概念

### 作用域和作用域链
##### [回到顶部](#Javascript篇)

> 说说你对作用域和作用域链的理解

当代码在一个环境中执行时，会创建 变量对象的一个作用域。
作用域链是 多个上下级关系作用域 形成的链，方向是从内到外的。
函数内指向函数外的链式结构称为作用域链。
查找变量就是沿着作用域链来查找的。
用途是 保证 能够有序访问 执行环境中有权访问的 所有变量和函数。（有序访问变量和函数），访问到window对象就被终止。

### 闭包
##### [回到顶部](#JavaScript篇)

> 什么是闭包？有什么作用？

闭包是指有权访问 另一个函数作用域中的变量 的函数；
通过 函数执行 形成一个 **不销毁的私有作用域**，作用是保存和保护了里面的私有变量，防止污染全局。


### 事件委托
##### [回到顶部](#JavaScript篇)

> 解释一下事件委托

事件委托就是利用 冒泡的原理，只指定一个事件处理程序，把事件加到父元素或祖先元素上，委托父级代为执行事件，就可以管理某一类型的所有事件。
优点： 提高javascript性能。事件委托可以显著提高事件的处理速度，减少内存的占用。

### 事件循环
##### [回到顶部](#JavaScript篇)

*一个事件循环有着至少一个任务队列，每个任务队列中都是特定的任务源产生的任务，这些任务源通常都是：事件、回调、I/O操作，DOM操作等，同一个任务队列中的任务严格按照先进先出的顺序执行，不同任务队列的任务执行顺序不一定。任务队列有宏任务和微任务之分。
因为JavaScript在浏览器中特点导致了它只能是的单线程的，这样就导致了Javascript中的并发模型不能像传统语言那样多线程操作。
JavaScript 的并发模型基于"事件循环"。*

> 下面代码的输出结果是什么？

```
console.log(1);
setTimeout(() => {
  console.log(2);

  setTimeout(() => { console.log(3) });

  new Promise(function (resolve) {
    console.log(4);
    resolve();
  }).then(function () {
    console.log(5);
  }, 1000);
})

async function async1() {
  console.log(6);
  await async2();
  new Promise(function (resolve) {
    console.log(7);
    resolve();
  }).then(function () {
    console.log(8);
  });
  console.log(9);
}

new Promise(function (resolve) {
  console.log(10);
  resolve();
}).then(function () {
  console.log(11);
});

async function async2() {
  console.log(12);
}

setTimeout(() => {
  console.log(13);
  new Promise(function (resolve) {
    console.log(14);
    resolve();
  }).then(function () {
    console.log(15);
  });
})

async1();
console.log(16)
```

需要关注的几点：
1、创建promise对象里面的代码属于同步代码
2、promise的then/catch才是异步，需进入任务队列
3、setTimeout也会进入任务队列，但是和promise的then/catch似乎不是同一队列，优先级低于 then/catch 的
4、遇到await的，eg: await async2(); async2会立刻执行，但是，紧接着await会让出当前线程，将后面的代码加到任务队列中。



>答案：1 10 6 12 16 11 7 9 8 2 4 5 13 14 15 3

### 同源策略
##### [回到顶部](#JavaScript篇)

> 解释一下同源策略

同源策略是由Netscape提出的一个安全策略。指一个域下的页面中，无法通过ajax请求获取到其他域的接口数据。
同源指协议、域名、端口号相同。
目的是 防止某个文档或脚本从多个不同源装载。保证用户信息的安全，防止被恶意网站窃取敏感数据（例如，登录态的cookie）。

### 数组去重
##### [回到顶部](#JavaScript篇)
法一：
```
function unique(arr) {
    const temp = [];
    arr.forEach(item => {
        if (temp.indexOf(item) === -1) {
            temp.push(item);
        }
    })
    return temp
}
```
`知识点：遍历, indexOf`

法二：
```
function unique (arr) {
   return Array.from(new Set(arr))
}
```
`知识点：Set`

### web安全
> 说说常见web安全及防护原理

①. xss(跨站脚本攻击)，原理是 攻击者 往web页面里面 插入恶意可执行网页脚本代码，使得其他用户浏览该网页时受影响。
防御： 转义输入输出的内容。

②. csrf(跨站请求伪造)，是 一种攻击者盗用用户在当前已登录的web页面上的身份信息执行非本意的操作（模拟发送请求）的攻击方式


### BFC
##### [回到顶部](#CSS篇)
> 请阐述 BFC (块格式化上下文Block Formatting Context) 及其工作原理。


### 伪类和伪元素
##### [回到顶部](#CSS篇)


### 垂直居中
##### [回到顶部](#CSS篇)


### 两列布局
##### [回到顶部](#CSS篇)
> 写一个左侧固定200px，右侧自适应布局

1、左边左浮动，右边设置margin-left:200px；
2、左边绝对定位，右边设置margin-left:200px；
3、右侧元素设置顶线和右线的位置为0， 左线位置为200px，宽度为100%；

<!-- ### 三列布局
##### [回到顶部](#CSS篇) -->


<!-- 某模块的宽度自适应，高度是宽的的2倍； -->


