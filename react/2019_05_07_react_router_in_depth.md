# 深入理解 React Router 实现原理

## 锚点( Anchor )和 hash
锚点和 hash 都跟 # 直接相关。但到底啥叫锚点？啥又叫 hash？

### 锚点
锚点是一个 html 中的标记定义， `<a/>`就是锚元素。在使用的过程中，锚点往往用于标记元素在文档的位置，在使用的使用会设置锚点的链接，使用该链接可以跳转至对应的锚点位置。
```
// 定义一个锚点
<a name="content">内容</a>
// 使用 id 也可以替代 name ，产生相同作用
<a id="content">内容</a>

// 可以跳转至以上锚点位置的链接
<a href="#content">点我去内容</a>

// 所有元素其实均可作为锚点存在 只要设置 name 或者 id 属性
// 同时设置 name 和 id 解决一些兼容性问题
<div name="content" id="content">内容</div>
```

> 关于锚点触发滚动有一个重要条件：无滚动则无定位。换而言之就是1、当前元素或其父标签可滚动2、锚点元素在元素内部。

> 浏览器 F5 刷新不会触发锚点定位，在 Chrome 浏览器下，这个过程分为三步：首先，滚动高度为 0，其次，锚点定位高度，最后，还原成刷新之前滚动条的滚动高度。

### hash
`#` 代表了文档中的一个位置，虽然它会显示与浏览器的地址栏中，但在真正发起 http 请求时，并不会被包含进 url 中，而会在 `#` 处进行截断，之后的所有内容将会被忽略。
```
http://www.example.com/index.html#content
=>
GET /index.html HTTP/1.1
Host: www.example.com

http://www.example.com/?color=#fff
=>
GET /?color= HTTP/1.1
Host: www.example.com
```

改变 `#` 内容不触发页面重载。但是每次一改变均会在浏览器的访问历史中增加一个记录（IE6 IE7等除外），使用“后退”按钮可以回到上一个位置（修改#内容）。
而 `#` 的值可以从 `window.location.hash` 取到，取到的值包含 `#` （例如以上为 "#content"）。而这也就是我们常说的浏览器 hash 了。
```
window.location.hash  => ""
window.location.hash = "123" => "123"
window.location.hash => "#123"
```

H5 中增加了对 Hash 的监听事件，可以实时监听 hash 的变化，也为利用 hash 完成前端路由提供了可能性。
```
/** event 中包含了以下参数
    type: "hashchange"
    newURL: 新的完整URL
    oldURL: 旧的完整URL
**/
window.onhashchange = function(event) {
  console.log(event);
}
window.addEventListener('hashchange', function(event){
  console.log(event);
})
```

> 关于单页应用影响 SEO，现在已经有很多解决方案了，有条件最常见的是，将 SEO 引擎流量引导至单独应用，返回静态的页面。

## history 实现前端路由
Html5 的 History 接口允许操作浏览器的会话历史记录。

History 属性：
1. History.length：返回在会话历史中有多条记录，包含了当前会话页面。如果打开一个新的 Tab，那么 length 值为 1。
2. History.state：保存了会触发 popState 事件的方法，后续进行介绍。

History 方法：
1. History.back()：返回浏览器会话历史中的上一页，跟浏览器的回退按钮功能相同
2. History.forward()：前进至浏览器会话历史的洗衣液，跟浏览器的前进阿娜妞功能相同
3. History.go()：可以跳转到浏览器会话历史中的指定的某一个记录页（ex : history.go(-1)）
4. History.pushState()：pushState 可以将给定的数据压入到浏览器会话历史栈中，该方法接受 3 个参数，对象、title 和 url。pushState 后会改变当前的 url，但是不会伴随着刷新
5. History.replaceState()：replaceState 将当前的会话页面的 url 替换成指定的数据， replaceState 后也会改变当前页面的 url， 也不会伴随刷新，与 pushState 的区别是 replaceState 不会增加 History.length。 而前者会 +1。

```
history.pushState(stateObj, title, url);
history.replaceState(stateObj, title, url);

window.history.pushState({foo:'bar'}, "page", "bar.html")

// {foo: 'bar'}
console.log(window.history.state);
```
* 如果页面发生 back 或者 forward ，那么就会触发 popstate 事件。
* history.pushState 和 history.replaceState 可以在不刷新的情况下修改 url 的地址，并利用 history.state 保存参数，但不会触发 popState。

## React-Router 原理
React-Router 中有许多的 API，比如 BrowserRouter、Router、Link、Switch 等。这里只简单介绍一下 `<Route>` 和 `<Link>`。
> 这里重新巩固一个理解：
> 1. React 是一个 Component Based 的框架，所有用 JSX 书写的标签，都避免不了是一个 Component，因此一个 `<Route>` 只能控制自己被不被渲染，不能影响其他的。
> 2. 如果需要选择路由，请使用 `<Switch>` 去进行包裹。
> 3. 一个 `<Route>` 更像是一个 if else 结构，if match 则渲染该 `<Route>`, if not 则跳过。

### `<Route>`
`<Route>` 完成的事情就是匹配响应的 location 中的地址，匹配成功则渲染对应的组件。

其中匹配的唯一标识及 path 值，当 location 发生变化时，Route 标签会重新匹配 path 值，若匹配成功，则进行渲染。其中 exact、strict 可以辅助增强匹配
```
<Route path='/home' component={Home}/>
// /      失败
// /home 成功
// /home/ 成功
// /home/1 成功

<Route path='/home' exact component={Home}/>
// /      失败
// /home 成功
// /home/ 成功
// /home/1  失败

<Route path='/home' strict component={Home}/>
// /      失败
// /home 成功
// /home/  失败  严格控制'/'
// /home/1 失败
```
Route 的渲染可以由以下三个内容决定
* component：该属性接受一个 React 组件，当 url 匹配成功，就会渲染该组件
* render：func 该属性接受一个返回 React Element 的函数，当 url 匹配成功时，渲染该返回的元素
* children：与 render 相似，接受一个返回的 React Element 的函数， 但是不同点是，无论 url 与当前的 Route 的 path 匹配与否，children 的内容始终会被渲染出来。

而这三个都会包含 location、match 和 history 三个参数，如果是组件则会自动存在 props 中。

### `<Link>`
`<Link>` 的作用类似于 `<a>` 标签，利用 to 参数设定需要设置的 url，点击后修改至对应的 url，然后进行 `<Router>` 匹配，同时也会传递 history、location 和 match。

```
// to string
<Link to='/home'>Home</Link>

// to object
<Link to={{pathname:'/home', search:'?sort=name', hash:'#edit', state:{a:1}}}>Home</Link>
// => 改变后的 Url 为 '/home?sort=name#edit'
// 其中 ? 后的内容不做为匹配内容，只是作为参数传递至匹配到的组件中，同样 state={a:1} 也会同样作为参数传递至需要渲染的组件中
```

> 记录一个写法：
>```
> var p = {
>    f : function() {
>      console.log(this)
>    },
>    x : "foo"
> };
> p.f();    // { f: ... x: foo}
> (p.f)();  // { f: ... x: foo}
> (0, p.f)(); // implicit global this  将 this 指向 global this
>```

## Route 源码解析
React Route 的简单示例
```
<Router>
  <div>
    <ul>
      <li>
        <Link to="/">首页</Link>
      </li>
      <li>
        <Link to="/about">关于</Link>
      </li>
    </ul>
  
    <Switch>
      <Route exact path="/" component={Home} />
      <Route path="about" component={About} />
    </Switch>
  </div>
</Router>
```

### Router
Router 作为 Route 的根组件，负责监听 url 的变化和传递数据（props），这里使用了 history.listen 监听 url，使用 react context 的 Provider 和 Consumer 模式。

```
class Router extends React.Component {
  static computeRootMatch(pathname) {
    return { path: "/", url: "/", params: {}, isExact: pathname === "/" };
  }

  constructor(props) {
    super(props);

    this.state = {
      location: props.history.location // 当前路径为 history 中的 location 值
    };

    // This is a bit of a hack. We have to start listening for location
    // changes here in the constructor in case there are any <Redirect>s
    // on the initial render. If there are, they will replace/push when
    // they mount and since cDM fires in children before parents, we may
    // get a new location before the <Router> is mounted.
    this._isMounted = false;
    this._pendingLocation = null;

    if (!props.staticContext) { // staticContext == true 则为服务器端渲染，不监听变化
      this.unlisten = props.history.listen(location => { // 注册 history 的监听事件，并返回 unlisten
        if (this._isMounted) {
          this.setState({ location });
        } else {
          this._pendingLocation = location;
        }
      });
    }
  }

  componentDidMount() {
    this._isMounted = true;

    if (this._pendingLocation) { // 切换完成将 location 赋值至 state 中
      this.setState({ location: this._pendingLocation });
    }
  }

  componentWillUnmount() { // 取消监听
    if (this.unlisten) this.unlisten();
  }

  render() {
    return (
      <RouterContext.Provider
        children={this.props.children || null} // 向下传递 children
        value={{ // 构建 history 相关 context value 向下传递
          history: this.props.history,
          location: this.state.location,
          match: Router.computeRootMatch(this.state.location.pathname),
          staticContext: this.props.staticContext
        }}
      />
    );
  }
}
```

> 关于 Context 使用了 React 新API 的 React Context，简单来讲就是在 <Context.Provider> 内部的所有组件均可以用 <Context.Consumer> 去获取 Context 值。不用每一级均向下传递。

### Link
React Router 中的 Link 其实本质即为 `<a>` 标签，但是其阻止了 `<a>` 标签的默认事件。而是利用 History 库中的 push 和 replace 来切换 location，达到路由切换的目的。
```
class Link extends React.Component {
  handleClick(event, history) {
    if (this.props.onClick) this.props.onClick(event);// 若传递了 onClick 方法，则使用传递的

    if (
      !event.defaultPrevented && // onClick prevented default
      event.button === 0 && // ignore everything but left clicks
      (!this.props.target || this.props.target === "_self") && // let browser handle "target=_blank" etc.
      !isModifiedEvent(event) // ignore clicks with modifier keys
    ) {
      event.preventDefault(); // 阻止默认事件，防止页面刷新，重新请求

      const method = this.props.replace ? history.replace : history.push;// 利用 history的 replace 和 push 来处理 location 变更

      method(this.props.to);
    }
  }

  render() {
    const { innerRef, replace, to, ...rest } = this.props; // eslint-disable-line no-unused-vars

    return (
      <RouterContext.Consumer>
        {context => {
          invariant(context, "You should not use <Link> outside a <Router>");

          const location =
            typeof to === "string"
              ? createLocation(to, null, null, context.location) // 如果传递的是 to 为字符串，例如 "/"
              : to;
          const href = location ? context.history.createHref(location) : "";

          return ( // 生成 <a> 标签内容
            <a
              {...rest}
              onClick={event => this.handleClick(event, context.history)}
              href={href}
              ref={innerRef}
            />
          );
        }}
      </RouterContext.Consumer>
    );
  }
}
```

### Route
Route 是 React 路由内容的践行者，他决定了什么样的 location 应该展示什么样的 component。但是不管 location 是什么， Route 组件总会被渲染，作为一个组件无法控制自己是否被渲染，只能控制自己渲染什么。当 Route 内容判断 location 与自己的 path 不匹配时，则返回 null （会被 react 忽略，没有任何内容），当匹配成功时，渲染 Route 中的 children 或者 component 或者 render 中的子组件。
```
class Route extends React.Component {
  render() {
    return (
      <RouterContext.Consumer>//获取 Router 中 provider 的 react context
        {context => {
          invariant(context, "You should not use <Route> outside a <Router>");

          const location = this.props.location || context.location;// 未传 location 则使用 context 中的 location
          const match = this.props.computedMatch
            ? this.props.computedMatch // <Switch> already computed the match for us
            : this.props.path
              ? matchPath(location.pathname, this.props) // *重要方法：决定了当前是否展示*
              : context.match; // 默认为 Router 中的 { path: "/", url: "/", params: {}, isExact: pathname === "/" }

          const props = { ...context, location, match };

          let { children, component, render } = this.props;

          // Preact uses an empty array as children by
          // default, so use null if that's the case.
          if (Array.isArray(children) && children.length === 0) {
            children = null;
          }

          if (typeof children === "function") {
            children = children(props);

            if (children === undefined) {
              children = null;
            }
          }

          return (
            <RouterContext.Provider value={props}>// 新的 RouterContext 向下使用
              {children && !isEmptyChildren(children) // 依次根据 children component render 选择一种渲染
                ? children // 如果 children 不为空 直接展示 children，所以 children 一定会展示
                : props.match // 否则依据匹配结果
                  ? component
                    ? React.createElement(component, props)
                    : render
                      ? render(props)
                      : null
                  : null}
            </RouterContext.Provider>
          );
        }}
      </RouterContext.Consumer>
    );
  }
}
```

### History 
History 库用于抽象浏览器的历史记录有以下几种类型，用不同的实现方式进行。
1. createHashHistory : 利用 hash 来实现
2. createBrowserHistory : 利用 H5 中的 location.history
3. createMemoryHistory : node 环境下使用，存储于 memrory中

```
// History 抽象
function createHistory(options={} {
  ...
  return {
    listenBefore, // 内部的 hook 机制， 可以在 location 发生变化前执行某些行为，AOP 实现
    listen, // location 发生改变时触发回调
    transitionTo, // 执行 location 的改变
    push, // history 方法
    replace, 
    go,
    goBack,
    goForward,
    createKey, //创建location的key,用于唯一标示该 location，是随机生成的
    createPath,
    createHref,
    createLocation, // 创建关键的 location 对象
  }
})
// History 中的 location 和浏览器原生的 location 的最大区别是多了 key 字段， history 内部可以通过 key 来对 location 进行操作
// Location 抽象
function createLocation() {
  return {
    pathname, // url 的基本路径
    search, // 查询字段
    hash, // url 中的 hash 值
    state, // url 对应的 state 字段
    action, // 分为 push replace pop 三种
    key, // 生成方法为： Math.random().toString(36).substr(2, keyLength)
  }
}
```
内部解析：
1. createHashHistory : 利用 hash 来存储不同状态下的 history 信息，方法有 location.hash = "***" location.replace() 等，在执行回退时，会触发 hashchange 事件
2. createBrowserHistory : 利用 H5 中的 location.history，方法有 pushStat、replaceState 等，在执行回退时，会触发 popstate 事件。
3. createMemoryHistory : 在内存中进行记录存储。

以下为 Api 的大致执行流程。

![React Router Api 流程](./images/2019_05_07_react_router_in_depth/react-route-in-depth.Jpg)

## 总结
1. react router 实现原理不外乎使用 hash history memory 三种方法
2. 在 React 中所有均为 Component，如果用前端路由就是讲所有 Component 的实现先下载至本地，在用本地路由去决定展示。
3. url 更新完成之后才发生路由匹配
4. 在 Router 中，window.location.href 的改变会刷新页面，并重新发起请求。而 History api 所进行的 url 改变，并不会触发页面刷新。
