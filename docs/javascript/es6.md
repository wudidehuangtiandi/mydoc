# ES6常用语法

#### 1.`let`和`const`

用以替代`var`，我们看下`var`的问题

```javascript
for(var i=0;i<=5;i++){
        console.log(i)
    }
console.log("我在循环外"+i)
```

假如使用`var`则输出结果为

![avatar](https://picture.zhanghong110.top/docsify/16450593426249.png)

可见其虽然在循环外但是也获得了循环内定义的参数，发生了溢出现象

```javascript
for(let i=0;i<=5;i++){
        console.log(i)
    }
console.log("我在循环外"+i)
```

> 在 `ES6` 之前，`JavaScript` 中变量默认是全局性的，只存在函数级作用域，声明函数曾经是创造作用域的唯一方法。这点和其他编程语言存在差异，其他语言大多数都存在块级作用域。所以在 `ES6` 中，新提出的 `let` 和 `const` 关键字使这个缺陷得到了修复。

![avatar](https://picture.zhanghong110.top/docsify/16450595002259.png)

> 同时还引入的概念 `const`，用来定义一个常量，一旦定义以后不可以修改，如果是引用类型，那么可以修改其引用属性，不可以修改其引用。

![avatar](https://picture.zhanghong110.top/docsify/16450597304811.png)

#### 2.字符串扩展

增加模板字符，之前多行拼接字符串比较麻烦现在可以用模板字符 ``

![avatar](https://picture.zhanghong110.top/docsify/16450614885234.png)

ES6 之前判断字符串是否包含子串，用` indexO`f 方法，ES6 新增了子串的识别方法。

- **includes()**：返回布尔值，判断是否找到参数字符串。
- **startsWith()**：返回布尔值，判断参数字符串是否在原字符串的头部。
- **endsWith()**：返回布尔值，判断参数字符串是否在原字符串的尾部。

```javascript
let string = "apple,banana,orange";
string.includes("banana");     // true
string.startsWith("apple");    // true
string.endsWith("apple");      // false
string.startsWith("banana",6)  // true
```

- 这三个方法只返回布尔值，如果需要知道子串的位置，还是得用 indexOf 和 lastIndexOf 。
- 这三个方法如果传入了正则表达式而不是字符串，会抛出错误。而 indexOf 和 lastIndexOf 这两个方法，它们会将正则表达式转换为字符串并搜索它。

#### 3.解构赋值

解构语法可以快速从数组或者对象中提取变量，可以用一个表达式读取整个结构。

解构数组

![avatar](https://picture.zhanghong110.top/docsify/16450618991064.png)

解构对象

![avatar](https://picture.zhanghong110.top/docsify/16450620043480.png)

> 解构赋值可以看做一个语法糖，它受 `Python` 语言的启发，提高效率之神器。

#### 4.函数优化

方法默认值的设置

![avatar](https://picture.zhanghong110.top/docsify/16450624054638.png)

箭头函数

![avatar](https://picture.zhanghong110.top/docsify/16450626484193.png)

不定参数

![avatar](https://picture.zhanghong110.top/docsify/16450758355305.png)

#### 5.数组的扩展

`map`,将数组中的每个元素处理后返回一个新的数组

![avatar](https://picture.zhanghong110.top/docsify/16450784873333.png)

传入function可以看见b为对应的次数

![avatar](https://picture.zhanghong110.top/docsify/16450797273008.png)

`reduce`的用法,求和，这里例子是求和，a初始值为10之后为计算出的值，b为m1,和是70

![avatar](https://picture.zhanghong110.top/docsify/16450801581092.png)

#### 6.对象的扩展

直接获取对象的k，v或者对象组成数组

![avatar](https://picture.zhanghong110.top/docsify/16450808428965.png)

对象拷贝

![avatar](https://picture.zhanghong110.top/docsify/16450813233208.png)

对象查询

![avatar](https://picture.zhanghong110.top/docsify/1645081618505.png)

#### 7.class类

>在ES6中，class (类)作为对象的模板被引入，可以通过 class 关键字定义类。
>
>class 的本质是 function。
>
>它可以看作一个语法糖，让对象原型的写法更加清晰、更像面向对象编程的语法。



类定义

```javascript
// 匿名类
let Example = class {
    constructor(a) {
        this.a = a;
    }
}
// 命名类
let Example = class Example {
    constructor(a) {
        this.a = a;
    }
}
```

类声明

```javascript
class Example {
    constructor(a) {
        this.a = a;
    }
}
```

> 类的使用比较多，这边就不扩展了，有兴趣的可以去下面这个链接去学习学习

[跳转](https://www.runoob.com/w3cnote/es6-class.html)

#### 8.Promise类

> 是异步编程的一种解决方案。从语法上说，Promise 是一个对象，从它可以获取异步操作的消息。



*Promise 异步操作有三种状态：pending（进行中）、fulfilled（已成功）和 rejected（已失败）。除了异步操作的结果，任何其他操作都无法改变这个状态。*

*Promise 对象只有：从 pending 变为 fulfilled 和从 pending 变为 rejected 的状态改变。只要处于 fulfilled 和 rejected ，状态就不会再变了即 resolved（已定型）*。

![avatar](https://picture.zhanghong110.top/docsify/16450830321478.png)

> then 方法接收两个函数作为参数，第一个参数是 Promise 执行成功时的回调，第二个参数是 Promise 执行失败时的回调，两个函数只会有一个被调用。
>
> 在 JavaScript 事件队列的当前运行完成之前，回调函数永远不会被调用。

![avatar](https://picture.zhanghong110.top/docsify/16450833124923.png)



then 方法将返回一个 resolved 或 rejected 状态的 Promise 对象用于链式调用，且 Promise 对象的值就是这个返回值。

![avatar](https://picture.zhanghong110.top/docsify/16450834045520.png)

!>简便的 Promise 链式编程最好保持扁平化，不要嵌套 Promise。注意总是返回或终止 Promise 链。大多数浏览器中不能终止的 Promise 链里的 rejection，建议后面都跟上 **.catch(error => console.log(error));**

#### 9.async函数

> async 是 ES7 才有的与异步操作有关的关键字，和 Promise  有很大关联的。

语法如下

```javascript
async function name([param[, param[, ... param]]]) { statements }
```

- name: 函数名称。
- param: 要传递给函数的参数的名称。
- statements: 函数体语句。

`async` 函数返回一个 `Promise` 对象，可以使用` then` 方法添加回调函数。

![avatar](https://picture.zhanghong110.top/docsify/16450840543224.png)

`async` 函数中可能会有 `await `表达式，`async `函数执行时，如果遇到`await` 就会先暂停执行 ，等到触发的异步操作完成后，恢复 `async` 函数的执行并返回解析值。

`await` 关键字仅在 `async function` 中有效。如果在 `async function` 函数体外使用 `await` ，你只会得到一个语法错误。

语法如下

```javascript
[return_value] = await expression;
```

- `expression`: 一个 `Promise` 对象或者任何要等待的值。



返回 Promise 对象的处理结果。如果等待的不是 Promise 对象，则返回该值本身。

如果一个 Promise 被传递给一个 await 操作符，await 将等待 Promise 正常处理完成并返回其处理结果。

![avatar](https://picture.zhanghong110.top/docsify/16450842253821.png)

#### 10.ES6 模块

> 在 ES6 前， 实现模块化使用的是 RequireJS 或者 seaJS（分别是基于 AMD 规范的模块化库， 和基于 CMD 规范的模块化库）。

*ES6 引入了模块化，其设计思想是在编译时就能确定模块的依赖关系，以及输入和输出的变量。*

*ES6 的模块化分为导出（export） @与导入（import）两个模块。*

*ES6 的模块自动开启严格模式，不管你有没有在模块头部加上 **use strict;**。*

*模块中可以导入和导出各种类型的变量，如函数，对象，字符串，数字，布尔值，类等。*

*每个模块都有自己的上下文，每一个模块内声明的变量都是局部变量，不会污染全局作用域。*

*每一个模块只加载一次（是单例的）， 若再去加载同目录下同文件，直接从内存中读取。*

举个栗子

```javascript
import { myName, myAge, myfn, myClass } from "./test.js";
export { myName, myAge, myfn, myClass }
```

