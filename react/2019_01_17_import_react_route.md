# React-Router 

> 为了给网站维护登录状态，必须通过路由的方式将未登录的访问转移至登录，注册等页面，在路由的管理方面必须借助 ReactRouter 才能实现

## Route

```
import { Route } from 'react-router-dom'
<Route path="/auth/login" component={LoginPage}></Route>
```
上述代码中，Route 其实也是一个 jsx 标签，在渲染的过程中，会拿当前请求的 url 和 path 属性进行匹配，如果匹配上了则渲染 Route 标签对应的 component 或者标签内的内容，也就是说还可以写成这样```<Route path="/auth/login"><LoginPage/></Route>```，结果是一样的。

#### 在使用过程中，我写成了 ```import Route from 'react-router-dom'```，然后一直报错，这里记一下， `import {} from` 和`import from`的区别。
1. {}中包含的是一个对象，里面的变量名必须与 from 对应的模块的对外接口名称相同，也就是说：使用 import {Route} 是因为 `'react-router-dom'`里写了`export Route`
2. `import from`的时候（即没有{}），是因为在 from 的模块中输出了`export default`， 也就是说 `import React from 'react'`,是因为`'react'`里面包含了`export default`,而我们把 default 重命名为了 React 而已

## Switch

在路径渲染的过程中，往往会出现优先级问题，比如登录有如下逻辑
```
if (path != '/auth/login' && !logged) //没有登录，且没有访问登录页面，转到登录页面
    redirect to /auth/login
else if ( path == '/auth/login')//渲染登录页面
    render LoginPage
else //已登录，渲染正常页面
    render PrimaryLayout
```

### js 辅助实现
```
if (this.props.location.pathname !== "/auth/login" && !logged)
{
    return (
    <Redirect to="/auth/login"/>
    )
} else if (this.props.location.pathname === "/auth/login"){
    return <UnauthorizedLayout {...props}/>
}else {
return <PrimaryLayout {...props} />
}
```

### Switch 实现
```
    <Switch>
      <Route path="/auth/login"><UnauthorizedLayout {...props} /></Route>
      <Route {...rest} render={props => {
        if (pending) return <div>Loading...</div>
        return logged
        ? <PrimaryLayout {...props} />
        : <Redirect to="/auth/login"/>
      }}/>
    </Switch>
```
可以看出 Switch 的实现更加优雅，没有繁琐的防死循环的额外代码，在每个 Switch 中，只有一个 Route 会被响应，越早匹配到的优先响应。

## Redirect
```
<Redirect from="message/:id" to="/message/:id"/>
```
从某一个 path 重定位至另一个 path，请求会重新处理

## <Route>能渲染什么

1. `<component>` 当一个带有`component` prop 的路由匹配的时候，route 将会返回 prop 提供的 component 类型的组件（通过`React.createElement`渲染）
2. `render` 与`component`类似，也是当路径匹配的时候会被调用。写成内联形式渲染和传递参数的时候非常方便。
3. `children` 这种情况下无论路由与当前路径是否匹配都会进行渲染

```
//1
<Route path='/page' component={Page} />
//2
const extraProps = { color: 'red'}
<Route path='/page' render={{props} => (
    <Page {...props} data={extraProps}/>
)}/>
//3
<Route path='/page' children={(props)=> (
    Props.match
    ? <Page {...props}/>
    : <EmptyPage {...props}/>
)}/>
```

## 路由嵌套

exact 表示路径需完全匹配，以下例子在没有传递 id 时会被第一个路由解析，传递了会被第二个路由解析
```
<Switch>
    <Route exact path='/demand' component={AllDemands}/>
    <Route path='/demand/:id' component={DemandDetail}/>
</Switch>
```

## Links
利用<Link/> 标签可以在不重新请求整个页面的情况下切换路由，切换视图
```
import { Link } from 'react-router-dom'
const Header = () => {
    <header>
        <nav>
            <ul>
                <li><Link to='/'>Home</Link></li>
                <li><Link to='demand'>Demand</Link></li>
            </ul>
        </nav>
    </header>
}
```

## Some Tips
1. locations 是包含 URL 不同部分参数的对象
```
{ pathname: '/', search: '', hash: '', key: 'key', state: {} }
```
2. 一个无路径的`<Route />`可以匹配所有路径，可以很方便的访问存储在 context 上的对象和方法
3. 当使用 `children prop` 时，即使路径不匹配的时候也会渲染
4. `component` 和 `render`的区别是，`component`会使用`React.createElement`来创建元素，`render`将组件视作一个函数。如果想创建一个内联函数并传递给`component`，那么`rende`r 会比 `component` 来的快得多。
```
<Route path='/one' component={One} />
// React.createElement(props.component);
<Route path='/two' render={() => <Two />}/>
// props.render();
```
5. `<Route>` 和 `<Switch>` 组件可以接受一个 `location prop`, 这可以让他们被一个不同的 location 匹配到，而不仅仅是他们实际的 location（当前的 URL）
6. props 也可以传递 staticContext 这个 prop， 但是只在使用服务器渲染的时候有效。

## 附更多用法及示例
https://reacttraining.com/react-router
