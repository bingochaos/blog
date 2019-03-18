# Hello World for Express

> 本文所指的 Express 为 Node 基金会下的一个项目，其所指开发高度包容、快速而极致的 Node.js Web 框架，介于项目本身后端复杂度不高，本次采用该框架进行快速迭代后端的开发工作。

## 安装及初始化 Express 项目
```
$ npm install -g express //全局安装 express
$ npm install -g express-generator //全局安装 express 生成器
$ express -e Server //初始化 Server 项目
$ cd Server
$ npm install //安装 express 项目
$ npm start //启动服务器
```
目录结构如下：

├─bin

├─node_modules

├─public

│  ├─images

│  ├─javascripts

│  └─stylesheets

├─routes

└─views

完成上述步骤后，一个 Express 项目就已经建好了，并且已经包含了"/"请求了，下面手写添加几个示例进行测试。

app.js 为程序入口，默认代码如下，载入的模块比较多，这里不做一一分析，只增加了部分关键注释：
```
var createError = require('http-errors');
var express = require('express');
var path = require('path');
var cookieParser = require('cookie-parser');
var logger = require('morgan');

//引入路由 js
var indexRouter = require('./routes/index');
var usersRouter = require('./routes/users');
var demandRouter = require('./routes/demand');
//生成 express 对象
var app = express();

// view engine setup
app.set('views', path.join(__dirname, 'views'));
app.set('view engine', 'ejs');

app.use(logger('dev'));
app.use(express.json());
app.use(express.urlencoded({ extended: false }));
app.use(cookieParser());
app.use(express.static(path.join(__dirname, 'public')));

//注册路由，注册之后可以使得路由隔离，比如 get /users/api,再书写时只需在 usersRouter 中增加/api 接口即可
app.use('/', indexRouter);
app.use('/users', usersRouter);

app.use('/demand', demandRouter);

// catch 404 and forward to error handler
app.use(function(req, res, next) {
  next(createError(404));
});

// error handler
app.use(function(err, req, res, next) {
  // set locals, only providing error in development
  res.locals.message = err.message;
  res.locals.error = req.app.get('env') === 'development' ? err : {};

  // render the error page
  res.status(err.status || 500);
  res.render('error');
});

module.exports = app;
```

## /HelloWorld
由于 index.js 已经注册为'/'路由，则只需在 index 中增加/HelloWorld 方法即可
```
var express = require('express');
var router = express.Router();

/* GET home page. */
router.get('/', function(req, res, next) {
  res.render('index', { title: 'Express' });
});

router.get('/HelloWorld', function(req, res, next){//添加/HelloWorld 方法
  res.send('Hello World');
});

module.exports = router;
```
请求“http://localhost:port/HelloWorld” ,正常返回 Hello World！

## /json  返回 json 数据
```
router.get('/json', function(req, res, next) {
  res.json(JSON.stringify({
    key: "key",
    value: "value",
  }))
});
```
请求“http://localhost:port/json" ,返回{key：key，value:value}

## 获取参数
express 获取 url 参数有四种方法

### 1.req.body 获取 post 参数
```
// => Post {"name":"bingoc"}
req.body.name //=> bingoc
```

### 2.req.params 获取 URL 参数
```
// => Get /user/bingoc
app.get('user/:name', function(req, res) {
    res.send(req.params.name);//=> bingoc
});
```

### 3.req.query 获取 URL 参数
```
// => Get /query?name=bingoc
res.query.name //=> bingoc
```

### 4.req.params 获取参数
```
//req.params 可以获取上述三种方法传递的参数
req.params['name'] //=>bingoc
```

## express 核心概念介绍

### 中间件（middleware)

> app.use([path], function)
Use the given middleware function, with optional mount path, defaulting to "/"

个人理解，每次调用 app.use，相当于把每个 path 加到了 pipeline 里面，每一个请求进来都会依次经过路径匹配的 function 进行处理,比如以下代码会经过三个"/", "/users", "/demand",直到匹配后对请求进行处理。但更多的中间件的作用应该是用来插入参数解析，cookie 解析等
```
app.use('/', indexRouter);
app.use('/users', usersRouter);
app.use('/demand', demandRouter);
```
中间件实现
```
router.get('/', function(req, res, next) {
  res.send('respond with a resource');
});
```
> get ： Http Method for which the middleware function applies
'/' : Path(route) for which the middleware function applies
function() : The middleware function.
req : Http request argument to the middleware function, called "req" by convention
res : Http response argument to the middleware function, called "res" by convention
next : Callback argument to the middleware function, called "next" by convertion

### 路由

路由即寻址，根据 url 分发 http 请求，感觉这里的路由和中间件的作用有些许重合，还是一切皆中间件。

Express 支持四种路由类型

1.普通字符串

/user => /user

2.字符串模式(感觉就是正则）

/user/man => /user/*man

/user/woman => /user/*man

3.正则表达式

/user => /users?$/

/users => /users?$/

4.参数类型

/user/bingoc => /user/:name


### 模板引擎
用于渲染 UI 的，本期不涉及不使用
