# setTimeout 和 window.onload 解析

## 知识准备

### js 执行

重要规则：
1. js 主线程执行是单线程的，只有当前 task 执行完后，才能执行下一个。
2. 然而浏览器还有很多其他线程，包括处理 Http 请求的，键盘输入的各种 I/O，均不占用主线程时间，在其就绪时会将其事件添加至主线程的 EventQueue 中，等待主线程处理。

在浏览器接受到 html 之后，整个 js 引擎主要分为两部分，大致如下图所示：
1. 执行页面内的主线程 js，构造 DOM 树
2. 进入 EventLoop，处理 EventQueue 中的 Event 事件。

这两个块根据我的理解应该基本等于$(document).ready()执行之前与之后。而 onload 只是第二部分 EventLoop 的其中一个事件。

![浏览器 js 执行过程](images/2019_03_04_settimeout_and_onload/js 流程.png "js 流程.Jpg")

> ##### H5多线程
> 为了利用多核 CPU 的能力，HTML5提出了 Web Worker 标准，允许 JavaScript 脚本创建多个线程，但是线程完全受主线程控制，父子线程通过 postMessage()相互传递消息。子线程可以执行任何代码，但不包括直接操作 DOM 节点，也不能使用 window 对象的默认方法和属性。然而可以使用大量 window 对象之下的东西，包括 WebSockets,IndexedDB 以及 FireFox OS 专用的 Data Store API 等数据存储机制。

### EventLoop

顾名思义就是用来处理事件的循环体，在各个语言，设计中均有类似组件。其代码形式一般如下：
```
      while (queue.waitForMessage()) {
        queue.processNextMessage();
      }  
```
EventQueue 保存的只有已就绪的事件，比如已到达唤醒时间的 setTimeOut，需要响应的 Onload 事件等。

> 在 EventLoop 中实际还存在两个队列，macrotask 和 microtask。本文不做详解，有兴趣自行查阅，可加深理解。

![EventLoop 图解](images/2019_03_04_settimeout_and_onload/js_eventloop.png "js 执行.Jpg")

## setTimeOut
setTimeout(func, timeout)计时器的主要功能就是在倒计时结束时将 func，加入 EventQueue 中，所以 setTimeout(func,0)，并不意味着会立即执行。而且其等待是见取决事件队列中，在该事件之前的事件数量以及各自的执行时间。因此会产生类似下面代码的执行结果。
```
      setTimeout(() => {
        console.log("timeout 2000");
      }, 1000);
      for (var i = 0; i< 1000000000; i++) {

      }
      setTimeout(() => {
        console.log("timeout 0")
      }, 0);

      // =>
      // timeout 2000
      // timeout 0
      // 因为当 timeout 0 被加入到 EventQueue 时，timeout 2000已提前就绪并执行了。
```

## Window.onload

### onload 不会立马执行
在浏览器主线程处理机制中，onload 事件会在整个页面的图片，css 外链资源都被加载渲染完成后，被加入 EventQueue 中。因此他甚至不是在全部加载后立马执行。比如下面这种情况。

```
      window.onload = function() {
        console.log("onload");
      }
      setTimeout(() => {
        console.log("timeout 0")
      }, 0);
      // =>
      // timeout 0
      // onload
      // 很明显 timeout 0更早被加入到 EventQueue 中
```

### JS 执行影响 onLoad

由于 js 主线程是单线程一旦有排在之前的 Task 时间较长，势必会影响到 Onload 执行

### img 加载时间过长会影响 onload 执行

下述代码中，如果 image.png 加载时间过长，导致 ajax 返回早于 onload 的执行，很有可能 onload 就要等所有返回列表里的数据返回才能执行了。
```
    <img src="image.png"/>
    <script>
    window.onload = function() {
      console.log('onload');
    }
    $.ajax({
      url: imgList,
      success: function(arr) {
        arr.forEach(function() {
          $('body').append('<img src="' + arr.imgUrl + '">')
        })
      }
    })
    </script>
```

### window.onload 与 window.addEventListener('load',function() {})

这两个方法均用于注册 onload 事件的回调，但是会有略微的区别：
1. window.onload 使用的是赋值语句，反复赋值，后者会将前者替代，而 window.addEventListener 则可以不断进行添加，执行顺序同添加顺序
2. 这两个函数可混用，执行顺序为添加顺序
3. window.onload 兼容性更好，window.addEventListener 不支持低版本以及部分浏览器，使用前最好添加判断

### Hybrid 进度条
iOS 中判断 webview 加载完成的 webViewDidFinishLoad 方法，Android 中判断 webview 加载完成的 onPageFinished 方法本质触发机制都对应页面上的 window.onload，一般来说会稍晚与 window.onload（某些特殊情况会早于 window.onload，比如页面中有 iframe 等情况）。
因此，更早的让 onload 触发，对于用户体验来说可能相对较好。之前遇到的进度条一直不消失的情况，可能不是网页加载慢，而是对应机制引起的。（据说 ios11可能不依赖这个了）
