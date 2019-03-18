> 本文主要梳理 babel 作用，以及相关的一些 ES6语法

# Babel
Babel 是一个通用的多用途 JavaScript 编译器（没错，一个解释型语言的编译器！），通过 babel 我们可以安心的写 ES6语法，而不用担心生产上因为浏览器不支持 ES6语法所带来的兼容性问题。由于 JavaScript 的不断发展，前端扮演着越来越重要的作用，不同语言标准也不断出现。ES5，ES6，ES7多个规范，甚至有 jsx 这种 react 的语言标准，很难保证其在所有浏览器中运行得当。
Babel 就是为了解决这一系列的问题，Babel 可以把用最新标准编写的 JavaScript 代码向下编译成在当下随处可用的版本。这种“源码到源码”的编译过程，也被称为转换编译（transpiling,转换+编译）。

## Babel 小试

[点我进入 babel 试用界面](http://www.babeljs.cn)

### 箭头函数
```
var odds = evens.map(v => v+1);
```
babel es2015转化后
```
var odds = evens.map(function(v) {
    return v+1;
});
```

### this 对象
```
var bob = {
    _name:'Bob',
    _friends: [],
    printFriends() {
        this._friends.forEach(f=>
            console.log(this._name + " knows " + f));
    }
};
```
babel es2015转化后
```
var bob = {
    _name:'Bob',
    _friends: [],
    printFriends: function printFriends() {
        var _this = this;//注意 this 指针在迭代其中产生的变化，而采用=>函数怎么不需要考虑这个问题
        this._friends.forEach(function(f) {
            return console.log(_this._name + " knows " + f);
        });
    }
};
```
由于箭头函数没有独立的 this 指针，也没有独立的 arguments,所以如果要取不定参的时候，要么需要改回 function,或者使用 ES6特性的 rest

### rest and spread
```
//rest
const getOptions = function(...args) {
    console.log(args.join);
}
//spread
const [opt1, ...opts]=['one', 'two', 'three', 'four'];
const opts = ['one','two','three'];
const config = ['other', ...opts];
```
babel es2015转化后
```
var getOptions = function getOptions() {
    for (var _len = arguments.length, args = Array(_len), _key=0;_key<_len;_key++) {
        args[_key] = arguments[_key];
    }
    console.log(args.join);
};

var opt1 = 'one',
opts = ['two','three','four'];

var opts = ['one', 'two', 'three'];
var config = ['other'].concat(_toConsumableArray(opts));
```

### class
ES2015的类其实只是一个语法糖，通过 class 关键字让 JavaScript 的语法更加接近传统的面向对象编程。
```
class MyClass {
    constructor() {
    }
}
```
babel es2015转化后
```
function _classCallCheck(instance, Constructor) {if(!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function");}}

var MyClass = function MyClass() {
    _classCallCheck(this, MyClass);
}
```
其本质还是基于原型的一个 function。

### const and let
```
const a = 1;
let b = 2;
//a = 3;  //babel 编译时报错（果然是编译。。）
b = 4;
```
babel es2015转化后
```
var a = 1;
var b = 2;

b = 4;
```
const 和 let 在转化后都会被用 var 代替，而在 es6语法中，const 表示常量，不能被再次赋值，而 let 则是替代了原有的 var。然而 const 其实可以替代的远比想象的要多，包括常量，配置项以及引用的组件、定义的“大部分”中间变量等，都应该是用 const 进行定义。let 的使用场景大多是在 loop（for,while)等循环中使用。
> 推测：一般来说 const 作为不可重新赋值的变量，应该在语法静态分析中，做出更多的优化。

### 字符串模板
```
const user = {
    name: 'bingo',
    id:'123'
}
const msg = `${user.name}-${user.id}`;
```
babel es2015转化后
```
var user = {
    name: 'bingo',
    id:'123'
}
const msg = user.name + '-' + user.id;
```
字符串模板功能非常实用，不用再书写大量的'+'号来拼接字符串，使得整个代码简洁很多，但要注意那个不是引号！！！！不是''或者"",而是``！！

### 属性简写
```
const config = {
    name,
    getName() {
        return this.name;
    }
}
```
babel es2015转化后
```
var config = {
    name: name,
    getName: function getName() {
        return this.name;
    }
}
```

### 关于 Promise，Array.from 等
关于这一类 ES2015+新增的功能，则需要利用另一个库 polyfill，来增加这部分功能的定义，polyfill 会在 babel-node 时自动加载。

## babel 简单补充

### 核心包
```
//babel-generator 暴露 babel.transform 方法来编译 source code
babel-core
//语法字符串解析 parser
babylon
//结合 plugins 和 presets(plugins)遍历 AST 语法树
babel-traverse
//生成最后的编译字符串
babel-generator
```

### 整个过程大概如下：
1. input string
2. babylon parser
3. AST
4. babel-traverse transformer[s] //结合 plugins 和 presets(plugins)遍历 AST 语法树
5. AST
6. babel-generator
7. output string

### babel-register 在 node 项目中的使用
babel-register 利用 require hook 来完成
通过修改 require function，先将所有 require 的代码 babel 一遍，然后在传给 runtime 执行




