---
title: JavaScript学习笔记
date: 2017-02-15 12:50:12
tags:
- 学习笔记

categories:
- 前端
    - JavaScript
---


JavaScript 学习笔记

<!-- more -->


#### 变量作用域

var 申明的变量的作用域

如果一个变量在函数体内部申明，则该变量的作用域为整个函数体，在函数体外不可引用该变量：

```javascript
'use strict';

function foo() {
    var x = 1;
    x = x + 1;
}

x = x + 2; // ReferenceError! 无法在函数体外引用变量x
```

由于JavaScript的函数可以嵌套，此时，内部函数可以访问外部函数定义的变量，反过来则不行。

JavaScript的函数在查找变量时从自身函数定义开始，从“内”向“外”查找。如果内部函数定义了与外部函数重名的变量，则内部函数的变量将“屏蔽”外部函数的变量。

```javascript
'use strict';

function foo() {
    var x = 1;
    function bar() {
        var y = x + 1; // bar可以访问foo的变量x!
    }
    var z = y + 1; // ReferenceError! foo不可以访问bar的变量y!
}
```

#### 变量提升

JavaScript的函数定义有个特点，会先扫描整个函数体的语句，把所有申明的变量“提升”到函数顶部，但是只会自动提升变量的声明，但不会提升变量的赋值。

我们在函数内部定义变量时，**请严格遵守“在函数内部首先申明所有变量”这一规则**

#### 全局作用域

不在任何函数内定义的变量就具有全局作用域。JavaScript默认有一个全局对象`window`，全局作用域的变量实际上被绑定到`window`的一个属性：

#### 名字空间

全局变量会绑定到`window`上，不同的JavaScript文件如果使用了相同的全局变量，或者定义了相同名字的顶层函数，都会造成命名冲突，并且很难被发现。

减少冲突的一个方法是把自己的所有变量和函数全部绑定到一个全局变量中。

#### 局部作用域

由于JavaScript的变量作用域实际上是函数内部，我们在`for`循环等语句块中是无法定义具有局部作用域的变量的：

```javascript
'use strict';

function foo() {
    for (var i=0; i<100; i++) {
        //
    }
    i += 100; // 仍然可以引用变量i
}
```

为了解决块级作用域，ES6引入了新的关键字`let`，用`let`替代`var`可以申明一个块级作用域的变量：

```javascript
'use strict';

function foo() {
    var sum = 0;
    for (let i=0; i<100; i++) {
        sum += i;
    }
    // SyntaxError:
    i += 1;
}
```

#### 常量

由于`var`和`let`申明的是变量，如果要申明一个常量，在ES6之前是不行的，我们通常用全部大写的变量来表示“这是一个常量，不要修改它的值”：

```
var PI = 3.14;
```

ES6标准引入了新的关键字`const`来定义常量，`const`与`let`都具有块级作用域：

```javascript
'use strict';

const PI = 3.14;
PI = 3; // 某些浏览器不报错，但是无效果！
PI; // 3.14
```

### 方法

在一个对象中绑定函数，称为这个对象的方法

```javascript
var xiaoming = {
    name: '小明',
    birth: 1990,
    age: function () {
        var y = new Date().getFullYear();
        return y - this.birth;
    }
};

xiaoming.age; // function xiaoming.age()
xiaoming.age(); // 今年调用是25,明年调用就变成26了
```

在一个方法内部，`this`是一个特殊变量，它始终指向当前对象，也就是`xiaoming`这个变量。所以，`this.birth`可以拿到`xiaoming`的`birth`属性。

> JavaScript的函数内部如果调用了`this`，那么这个`this`到底指向谁？

答案是，视情况而定！

如果以对象的方法形式调用，比如`xiaoming.age()`，该函数的`this`指向被调用的对象，也就是`xiaoming`，这是符合我们预期的。

如果单独调用函数，比如`getAge()`，此时，该函数的`this`指向全局对象，也就是`window`。

在strict模式下函数的`this`指向`undefined`，因此，在strict模式下，你会得到一个错误：

```javascript
'use strict';

var xiaoming = {
    name: '小明',
    birth: 1990,
    age: function () {
        var y = new Date().getFullYear();
        return y - this.birth;
    }
};

var fn = xiaoming.age;
fn(); // Uncaught TypeError: Cannot read property 'birth' of undefined
```

#### apply

虽然在一个独立的函数调用中，根据是否是strict模式，`this`指向`undefined`或`window`，不过，我们还是可以控制`this`的指向的！

要指定函数的`this`指向哪个对象，可以用函数本身的`apply`方法，它接收两个参数，第一个参数就是需要绑定的`this`变量，第二个参数是`Array`，表示函数本身的参数。

用`apply`修复`getAge()`调用：

```javascript
function getAge() {
    var y = new Date().getFullYear();
    return y - this.birth;
}

var xiaoming = {
    name: '小明',
    birth: 1990,
    age: getAge
};

xiaoming.age(); // 25
getAge.apply(xiaoming, []); // 25, this指向xiaoming, 参数为空
```

另一个与`apply()`类似的方法是`call()`，唯一区别是：

- `apply()`把参数打包成`Array`再传入；
- `call()`把参数按顺序传入。

比如调用`Math.max(3, 5, 4)`，分别用`apply()`和`call()`实现如下：

```javascript
Math.max.apply(null, [3, 5, 4]); // 5
Math.max.call(null, 3, 5, 4); // 5
```

对普通函数调用，我们通常把`this`绑定为`null`。

#### 装饰器

利用`apply()`，我们还可以动态改变函数的行为。

JavaScript的所有对象都是动态的，即使内置的函数，我们也可以重新指向新的函数。

现在假定我们想统计一下代码一共调用了多少次`parseInt()`，可以把所有的调用都找出来，然后手动加上`count += 1`，不过这样做太傻了。最佳方案是用我们自己的函数替换掉默认的`parseInt()`：

```javascript
'use strict';

var count = 0;
var oldParseInt = parseInt; // 保存原函数

window.parseInt = function () {
    count += 1;
    return oldParseInt.apply(null, arguments); // 调用原函数
};
```

### 高阶函数

高阶函数英文叫Higher-order function。那么什么是高阶函数？

JavaScript的函数其实都指向某个变量。既然变量可以指向函数，函数的参数能接收变量，那么一个函数就可以接收另一个函数作为参数，这种函数就称之为高阶函数。

一个最简单的高阶函数：

```javascript
function add(x, y, f) {
    return f(x) + f(y);
}
```

#### reduce

再看reduce的用法。Array的`reduce()`把一个函数作用在这个`Array`的`[x1, x2, x3...]`上，这个函数必须接收两个参数，`reduce()`把结果继续和序列的下一个元素做累积计算，其效果就是：

```mathematica
[x1, x2, x3, x4].reduce(f) = f(f(f(x1, x2), x3), x4)
```

#### filter

filter也是一个常用的操作，它用于把`Array`的某些元素过滤掉，然后返回剩下的元素。

`filter()`把传入的函数依次作用于每个元素，然后根据返回值是`true`还是`false`决定保留还是丢弃该元素。

例如，在一个`Array`中，删掉偶数，只保留奇数，可以这么写：

```javascript
var arr = [1, 2, 4, 5, 6, 9, 10, 15];
var r = arr.filter(function (x) {
    return x % 2 !== 0;
});
r; // [1, 5, 9, 15]
```

#### sort

```javascript
// 看上去正常的结果:
['Google', 'Apple', 'Microsoft'].sort(); // ['Apple', 'Google', 'Microsoft'];

// apple排在了最后:
['Google', 'apple', 'Microsoft'].sort(); // ['Google', 'Microsoft", 'apple']

// 无法理解的结果:
[10, 20, 1, 2].sort(); // [1, 10, 2, 20]
```

`Array`的`sort()`方法默认把所有元素先转换为String再排序，结果`'10'`排在了`'2'`的前面，因为字符`'1'`比字符`'2'`的ASCII码小。

`sort()`方法会直接对`Array`进行修改，它返回的结果仍是当前`Array`

### every

`every()`方法可以判断数组的所有元素是否满足测试条件。

例如，给定一个包含若干字符串的数组，判断所有字符串是否满足指定的测试条件

```javascript
var arr = ['Apple', 'pear', 'orange'];
console.log(arr.every(function (s) {
    return s.length > 0;
})); // true, 因为每个元素都满足s.length>0

console.log(arr.every(function (s) {
    return s.toLowerCase() === s;
})); // false, 因为不是每个元素都全部是小写

```

### find

`find()`方法用于查找符合条件的第一个元素，如果找到了，返回这个元素，否则，返回`undefined`

### findIndex

`findIndex()`和`find()`类似，也是查找符合条件的第一个元素，不同之处在于`findIndex()`会返回这个元素的索引，如果没有找到，返回`-1`

### forEach

`forEach()`和`map()`类似，它也把每个元素依次作用于传入的函数，但不会返回新的数组。`forEach()`常用于遍历数组，因此，传入的函数不需要返回值：

### 闭包

函数作为返回值，高阶函数除了可以接受函数作为参数外，还可以把函数作为结果值返回。

闭包就是携带状态的函数，并且它的状态可以完全对外隐藏起来。

### 箭头函数

ES6标准新增的一种新的函数：箭头函数相当于匿名函数，并且简化了函数定义。

箭头函数有两种格式，一种像 ```x => x * x``` 的，只包含一个表达式，连`{ ... }`和`return`都省略掉了。还有一种可以包含多条语句，这时候就不能省略`{ ... }`和`return`：如果参数不是一个，就需要用括号`()`括起来：

```javascript
x => {
    if (x > 0) {
        return x * x;
    }
    else {
        return - x * x;
    }
}
// 两个参数:
(x, y) => x * x + y * y

// 无参数:
() => 3.14

// 可变参数:
(x, y, ...rest) => {
    var i, sum = x + y;
    for (i=0; i<rest.length; i++) {
        sum += rest[i];
    }
    return sum;
}
```

箭头函数看上去是匿名函数的一种简写，但实际上，箭头函数和匿名函数有个明显的区别：箭头函数内部的`this`是词法作用域，由上下文确定 。箭头函数不会创建自己的`this,它只会从自己的作用域链的上一层继承this`。

#### generator

generator（生成器）是ES6标准引入的新的数据类型。一个generator看上去像一个函数，但可以返回多次。

### 标准对象

#### Date

在JavaScript中，`Date`对象用来表示日期和时间。

要获取系统当前时间，用：

```javascript
var now = new Date();
now; // Wed Jun 24 2015 19:49:22 GMT+0800 (CST)
now.getFullYear(); // 2015, 年份
now.getMonth(); // 5, 月份，注意月份范围是0~11，5表示六月
now.getDate(); // 24, 表示24号
now.getDay(); // 3, 表示星期三
now.getHours(); // 19, 24小时制
now.getMinutes(); // 49, 分钟
now.getSeconds(); // 22, 秒
now.getMilliseconds(); // 875, 毫秒数
now.getTime(); // 1435146562875, 以number形式表示的时间戳
```

**JavaScript的月份范围用整数表示是0~11**

如果要创建一个指定日期和时间的`Date`对象，可以用：

```javascript
var d = new Date(2015, 5, 19, 20, 15, 30, 123);
d; // Fri Jun 19 2015 20:15:30 GMT+0800 (CST)
```

##### 时区

`Date`对象表示的时间总是按浏览器所在时区显示的

#### RegExp

正则表达式是一种用来匹配字符串的强有力的武器。它的设计思想是用一种描述性的语言来给字符串定义一个规则，凡是符合规则的字符串，我们就认为它“匹配”了，否则，该字符串就是不合法的。

JavaScript有两种方式创建一个正则表达式：

第一种方式是直接通过`/正则表达式/`写出来，第二种方式是通过`new RegExp('正则表达式')`创建一个RegExp对象。

```javascript
var re1 = /ABC\-001/;
var re2 = new RegExp('ABC\\-001');

re1; // /ABC\-001/
re2; // /ABC\-001/
```

> 切分字符串

用正则表达式切分字符串比用固定的字符更灵活

```javascript
'a b   c'.split(/\s+/); // ['a', 'b', 'c']
```

> 分组

除了简单地判断是否匹配之外，正则表达式还有提取子串的强大功能。用`()`表示的就是要提取的分组（Group）。比如：

```javascript
var re = /^(\d{3})-(\d{3,8})$/;
re.exec('010-12345'); // ['010-12345', '010', '12345']
re.exec('010 12345'); // null
```

如果正则表达式中定义了组，就可以在`RegExp`对象上用`exec()`方法提取出子串来。

`exec()`方法在匹配成功后，会返回一个`Array`，第一个元素是正则表达式匹配到的整个字符串，后面的字符串表示匹配成功的子串。

`exec()`方法在匹配失败时返回`null`。

> 贪婪匹配

正则匹配默认是贪婪匹配，也就是匹配尽可能多的字符，如：

```javascript
var re = /^(\d+)(0*)$/;
re.exec('102300'); // ['102300', '102300', '']
```

由于`\d+`采用贪婪匹配，直接把后面的`0`全部匹配了，结果`0*`只能匹配空字符串了。

必须让`\d+`采用非贪婪匹配（也就是尽可能少匹配），才能把后面的`0`匹配出来，加个`?`就可以让`\d+`采用非贪婪匹配：

```javascript
var re = /^(\d+?)(0*)$/;
re.exec('102300'); // ['102300', '1023', '00']
```

> 全局搜索

JavaScript的正则表达式还有几个特殊的标志，最常用的是`g`，表示全局匹配：

```
var r1 = /test/g;
// 等价于:
var r2 = new RegExp('test', 'g');
```

全局匹配可以多次执行`exec()`方法来搜索一个匹配的字符串。当我们指定`g`标志后，每次运行`exec()`，正则表达式本身会更新`lastIndex`属性，表示上次匹配到的最后索引：

```javascript
var s = 'JavaScript, VBScript, JScript and ECMAScript';
var re=/[a-zA-Z]+Script/g;

// 使用全局匹配:
re.exec(s); // ['JavaScript']
re.lastIndex; // 10

re.exec(s); // ['VBScript']
re.lastIndex; // 20

re.exec(s); // ['JScript']
re.lastIndex; // 29

re.exec(s); // ['ECMAScript']
re.lastIndex; // 44

re.exec(s); // null，直到结束仍没有匹配到
```

全局匹配类似搜索，因此不能使用`/^...$/`，那样只会最多匹配一次。

正则表达式还可以指定`i`标志，表示忽略大小写，`m`标志，表示执行多行匹配

### 面向对象编程

JavaScript的面向对象编程和大多数其他语言如Java、C#的面向对象编程都不太一样。

JavaScript不区分类和实例的概念，而是通过原型（prototype）来实现面向对象编程。

JavaScript的原型链和Java的Class区别就在，它没有“Class”的概念，所有对象都是实例，所谓继承关系不过是把一个对象的原型指向另一个对象而已。

#### 创建对象

JavaScript对每个创建的对象都会设置一个原型，指向它的原型对象。

当我们用`obj.xxx`访问一个对象的属性时，JavaScript引擎先在当前对象上查找该属性，如果没有找到，就到其原型对象上找，如果还没有找到，就一直上溯到`Object.prototype`对象，最后，如果还没有找到，就只能返回`undefined`。

> 构造函数

除了直接用`{ ... }`创建一个对象外，JavaScript还可以用一种构造函数的方法来创建对象。它的用法是，先定义一个构造函数，用关键字`new`来调用这个函数，并返回一个对象：

```javascript
function Student(name) {
    this.name = name;
    this.hello = function () {
        alert('Hello, ' + this.name + '!');
    }
}
```

为了区分普通函数和构造函数，按照约定，构造函数首字母应当大写，而普通函数首字母应当小写，这样，一些语法检查工具如[jslint](http://www.jslint.com/)将可以帮你检测到漏写的`new`

### Symbol

ES5 的对象属性名都是字符串，这容易造成属性名的冲突。比如，你使用了一个他人提供的对象，但又想为这个对象添加新的方法（mixin 模式），新方法的名字就有可能与现有方法产生冲突。如果有一种机制，保证每个属性的名字都是独一无二的就好了，这样就从根本上防止属性名的冲突。

ES6 引入了一种新的原始数据类型`Symbol`，表示独一无二的值。它是 JavaScript 语言的第七种数据类型，前六种是：`undefined`、`null`、布尔值（Boolean）、字符串（String）、数值（Number）、对象（Object）。

`Symbol`函数前不能使用`new`命令，否则会报错。这是因为生成的 Symbol 是一个原始类型的值，不是对象。

Symbol 值也可以转为布尔值，但是不能转为数值。
