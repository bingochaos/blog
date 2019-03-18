# 关于初学 React 的一些 tips 总结

## Name your components

当你需要查找是哪个 Component 引起的 bug 时，Component 拥有一个名字就会变得很重要。

```
// Avoid thoses notations
export default () => {};
export default calss extends React.Component {};
```

React 支持自定义 Component 名称

```
export const Component = () => <h1>I'm a component </h1>;
export default Component;

// Define user custom component name
Component.displayName = "My Component';
```

ESLint rules 建议

```
"rules": {
    // check named import exists
    "import/named": 2,

    // Set to "on" in airbnb preset
    "import/prefer-default-export": "off"
}
```


## Prefer functional components

当你的 Component 只有展示数据作用时，使用 functional component 相比于 class component 代码更加简洁，也更加高效。同时仍然能够正常使用 props 等参数。

```
class Watch extends React.Component {
    render() {
        return <div>{this.props.hours}:{this.props.minutes}</div>
    }
}

// Equivalent functional component
const Watch = (props) => <div>{props.hours}:{props.minutes}</div>;
```


## Replace divs with fragments

由于任何一个 Component 在 render 时必须包含一个唯一的 root 节点，为了符合这条规则，我们一般会在最外层加一层<div>标签。在 React 16中引入了一个新特性 Fragments。用这个作为更节点包裹在 output 中不会包含任何 wrapper。

```
const Login = () => 
    <div><input name="login"/><input name="password"/><div>

const Login = () =>
    <React.Fragment><input name="login"/><input name="password"/></React.Fragment>;

const Login = () = // Short-hand syntax
    <><input name="login"/><input name="password"/></>;
```


## Be careful while setting state

在某些情况，在 setState 的时候可能需要使用之前的 state 数据，这时候不要直接读取 this.state，因为 setState 是异步函数，在这个时间内 state 可能已经产生变化。

```
// Very bad pratice: do not use this.state and this.props in setState !
this.setState({ answered: !this.state.answered, answer});

// With quite big states: the tempatation becomes bigger
// Here keep the current state and add answer property
this.setState({ ...this.state, answer });
```

应该采用提供的 function parameter 来正确利用原来的数据,props 也是一样

```
// Note the () notation around the object which makes the JS engine
// evaluate as an expression and not as the arrow function block
this.setState((prevState, props) 
    => ({ ...prevState, answer}));
```


## Binding component functions

有很多方法可以给当前 component 绑定事件，类似以下这样的

```
class DatePicker extends React.Component {
    handleDateSelected({target}) {
        // Do stuff
    }
    
    render() {
        return <input type="date" onChange={this.handleDateSelected}/>
    }
}
```

但是他却并不能正常工作，因为当你使用 JSX 时，this 并没有绑定到当前的 component instance 上，以下有几种方式可以让上面的代码正常工作。

```
// #1: use an arrow function
<input type="date" onChange={(event) => this.handleDateSelected(event)}/>

// OR #2: bind this to the function in component constructor
constructor() {
    this.handleDateSelected = this.handleDateSelected.bind(this);
}

// OR #3: declare the function as a class field (arrow function syntax)
handleDateSelected = ({target}) => {
    // Do stuff
}
```

这里不推荐使用第一种方法，第一种方法会导致代码在每一次 rerender 是重新 created，会影响渲染性能。第三种方法的使用需要 Babel 的支持，需要利用 Babel 转化代码，否则代码不能正常运行。


## Adopt container pattern (even with Redux)

the container design pattern。希望你将 React Component 尽可能的进行分离(follow the separation of concerns principle)。

```
export class DatePicker extends React.Component {
    state = { currentDate: null };
    
    handleDateSelected = ({target}) => 
        this.setState({ currentDate: target.value });

    render = () =>
        <input type="date" onChange={this.handleDateSelected}/>
```
一个 Component 既处理了 rendering 又处理了 user action，这里可以把他们拆成两个 Components
```
const DatePicker = (props) => 
    <input type="date" onChange={props.handleDateSelected}/>

export class DatePickerController extends React.Component {
    // ... No changes except render function ...
    render = () => 
        <DatePicker handleDateSelected={this.handleDateSelected}/>
```


## Fix props drilling

在书写页面是总是会出现很多孙子 Component 需要用到主 Component 的一些 props，但是这明显不能直接获取。有两个方法：

 1. 将他们包含在一个 Container 里面进行管理（wrapping the dumb component in a container
 2. 从父容器一层层传递 props

第二种方案往往会传递不是所有子容器都需要的 props 下去

```
const Page = props => <UserDetails fullName="John Doe"/>

const UserDetails = props => 
    <section>
        <h1>User details</h1>
        <CustomInput value={props.fullName}/> //<= No need fullName but pass it down
    </section>

const inputStyle = {
    height: '30px',
    width: '200px',
    fontSize: '19px',
    border: 'none',
    borderBottom: '1px solid black'
};
const CustomInput = props => // v Finally use fullName value from Page component
    <input style={inputStyle} type="text" defaultVlue={props.value}/>
```

这种现象叫做 props drilling，以下有一些解决方案主要利用了[Context API](https://reactjs.org/docs/context.html#before-you-use-context)中的 children prop 和 render prop


```
// #1: Use children prop
const UserDetailsWithChildren = props =>
    <section>
        <h1>User details (with children)</h1>
        {props.children /* <= use children */}
    </section>;

// #2: Render prop pattern
const UserDetailsWithRenderProp = props => 
    <section>
        <h1>User details (with render prop)</h1>
        <props.renderFullName() /* <= use passed render function */}
    </section>;

const Page = () => 
    <React.Fragment>
        {/* #1: children prop */}
        <UserDetailsWithChildren>
            <CustomInput value="John Doe"/> {/* Define props.children */}
        </UserDetailsWithChildren>

        {/* #2: Render prop pattern */}
        {/* Remember: passing arrow functions is a bad pratice, make it a method of Page class instead */}
        <UserDetailsWithRenderProp renderFullName={() => <CustomInput value="John Doe"/>}/>
    </React.Fragment>;
```

这样的解决方案看起来简单的多，children 看起来是更好的解决方案，因为在 render method 中他也能很好的运行。这样后再深的内联 Component 也可以解决了。

```
const Page = () =>
    <PageContent>
        <RightSection>
            <BoxContent>
                <UserDetailsWithChildren>
                    <CustomInput value="John Doe"/>
                </UserDetailsWithChildren>
            </BoxContent>
        </RightSection>
    </PageContent>
```

还有第三种解决思路，是利用 experimental context API

```
const UserFullNameContext = React.createContext('userFullName');

const Page = () => 
    <UserFullNameContext.Provider value="John Doe"> {/* Fill context with value */}
        <UserDetailsWithContext/>
    </UserFullNameContext.Provider>

const UserDetailsWithContext = () => // No props to provide
    <section>
        <h1>User details (with context)</h1>
        <UserFullNameContext.Consumer> {/* Get Context value */}
            { fullName => <CustomInput value={fullName}/>
        </UserFullNameContext.Consumer>
    </section>;
```

不推荐使用这种方法，其相当于存储在了全局变量中。


## 待补充
