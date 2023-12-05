---
title: React入门
date: 2021-07-21 21:38:41
tags:
    React JavaScript
categories:
    React
description: React相关基础认识
cover: false
---

# React入门

### 典中典：什么是React？

React 是一个**声明式**，**高效**且**灵活**的用于构建用户界面的 **JavaScript 库**。

使用 React 可以将一些简短、独立的代码片段组合成复杂的 UI 界面，这些代码片段被称作“组件”。

### React与传统JavaScript的优势和劣势？

1. **组件化**，提高代码利用率，声明式编码
2. 避免dom频繁的渲染，低效率
3. React Native使用React，学习移动端开发的前置知识
4. **虚拟DOM**，优秀的Diffing算法，减少真实dom操作
5. 有上手难度，需要学习和掌握全新的技术栈

### JSX

```jsx
class ShoppingList extends React.Component {
    render() {
        return (
            <div className="shopping-list">
                <h1>Shopping List for {this.props.name}</h1>
                <ul>
                    <li>Instagram</li>
                    <li>WhatsApp</li>
                    <li>Oculus</li>
                </ul>
            </div>
        );
    }
}
```

初见很怪异，它既包含html的元素，但又出在一个括号中，而且需要通过return将它返回出去。一般，需要返回某个数据，都是一个方法，react就是在render()方法中返回一个html块，
进行页面的渲染，正如react的介绍所说。上面的语法形式是被称为JSX。

在 JSX 中你可以任意使用 JavaScript 表达式，只需要用一个大括号把表达式括起来。每一个 React 元素事实上都是一个 JavaScript 对象，你可以在你的程序中把它保存在变量中或者作为参数传递

### 为什么不可变性在 React 中非常重要

我们在操作某个对象或者数组的时候，一般会有两种方式：

1. 直接对其进行操作，比如：

```js
var player = {score: 1, name: 'Jeff'};
player.score = 2;
```

2. 新数据替换旧数据，比如：

```js
var player = {score: 1, name: 'Jeff'};
Object.assign({}, player, {score: 3})
```

在React中，推荐使用第二种，原因有三:

1. 简化复杂的功能：不可变性使得复杂的特性更容易实现
2. 跟踪数据的改变：直接修改，会难以追踪数据
3. 确定在 React 中何时重新渲染：不可变性最主要的优势在于它可以帮助我们在 React 中创建 pure components。我们可以很轻松的确定不可变数据是否发生了改变，从而确定何时对组件进行重新渲染

这也涉及到对象在内存的创建与存储方式，一般创建一个对象，会在堆开辟一个空间存放，在栈中，创建一个对象的引用(地址值)，堆的对象也就会分配栈中的引用对象(地址值)。
我们的赋值操作，仅仅是在栈中创建了一引用对象，并将这个地址值分配给了堆中的对象，并没有在堆中重新开辟一个空间， 因此，当其中一个引用对象发生变化，那么其他的指向同一个堆的引用对象也会发生变化， 因为，他们实际上是一个东西，一变皆变。

### React小游戏 -- 井字棋

### 虚拟DOM（Virtual Dom）

简单来说就是一个JavaScript对象，它包含三个基本的点：标签(tag)，属性(props)，子类(children)，通过这三个点，来描述出真实dom，并通过各种方法渲染出真实dom

真实是dom：

```html

<div id="app">
    hello
</div>
```

虚拟dom：

```js
const ojb = {
    tag: "div",
    props: {
        id: "app"
    },
    children: [
        'hello'
    ]
}
```

通过上面代码来看，有点类型于伪代码。

### 虚拟dom的渲染

一般，我们书写了虚拟dom，浏览器并不能直接进行读取，都需要通过这类的翻译工具，将其转换为能够正常识别的方式。
当使用React书写虚拟dom时，一般推荐我们使用JSX语法来创建虚拟dom，React会通过babel将其转换为上面所示的虚拟dom的js对象形式。 之后就可以通过React自带的函数将其渲染为真实dom节点了。流程如下:

1. 使用jsx创建虚拟dom

```jsx
function getDom() {
    return (
        <div id="app">
            Hello Imdream
        </div>
    )
}
```

2. h函数：返回一个js对象，React中是`React.createElement`，但实际上还是h函数

```js
function h(tag, props, ...children) {
    return {
        tag,
        props: props || {},
        children: children.flat()
    }
}
```

3. 使用babel转换jsx，并使用h函数返回虚拟dom

```js
function getDom() {
    return h('div', {id: "app"}, "Hello Imdream")
}
```

4. 渲染虚拟dom: 选择根节点，传入对应需要渲染的dom组件，之后便将虚拟dom转换为真实dom，插入对应的根节点

```jsx
ReactDOM.render(
    <Component/>, // 组件
    document.getElementById('root') // 根节点
);
```

所以，通过使用虚拟dom的优势，结合diff算法(通俗的讲就是只更新我们需要的部分)，我们就可以省去很多渲染上的功夫，也就是提高了性能，一般业务量小，或者业务简单的需求，可能察觉不到频繁更新dom的性能损失，但是，换个环境(复杂业务)
就很容易体现出传统dom更新的瓶颈，上千个节点频繁的更新，画美不看。

### 组件

与后端相比，组件可以说是HTML中的一种抽象概念。尽可能提取可以重复使用的部分，然后通过这些抽象出的组合和未抽象的html组合，完成一个页面。当然，在组件中，也是可以存在组件的。组件名一般采用大驼峰的命名规范

```jsx
// 函数式组件
function Square(props) {
    return (
        <button className="square" onClick={() => props.onPress()}>
            {props.value}
        </button>
    );
}

// class组件
class Board extends React.Component {
    render() {
        return (
            <Square
                value={this.props.squares[i]}
                onPress={() => this.props.onSquareClick(i)}/>
        );
    }
}
```

#### Props

自定义组件的时候，我们也需要将一下参数传递到组件内部，比如在上面的函数式组件`Square()`中，会接收到传入的参数，内部使用这些参数进行页面的渲染。
这些参数是通过作为组件的属性以及子组件转换为单个对象进行传递的。**所有 React 组件都必须像纯函数一样保护它们的 props 不被更改**。

#### State

在组件中，可以使用props接受外部的数据，也可以使用State对象，作为内部数据集合。功能上，它们是类似的，但是，State完全受控于当前组件，相当于一个局部变量，我们可以使用`setState`函数去实时的更新它。

#### Event: 事件

除了使用到变量，在组件属性上，我们还需要用过其相关的事件。最为常见的就是点击事件。一般这种事件都会在当前组件内，使用`this`去调用。事件命名一般会使用小驼峰。大体上和DOM元素事件用法类似。

用法区别：
1. 阻止默认行为：
```html
<!-- 普通的dom中使用return false就能阻止事件 -->
<form onsubmit="console.log('You clicked submit.'); return false">
  <button type="submit">Submit</button>
</form>
```

```jsx
function From() {
  function submit(e) {
    e.preventDefault(); // 在React中需要显示声明，来阻止
    console.log('You clicked submit.');
  }

  return (
    <form onSubmit={submit}>
      <button type="submit">Submit</button>
    </form>
  );
}
```

2. this绑定：如果要在事件内使用`this`，一般有两种回调函数的书写方法，如果没有如此使用，那么`this`就是`undefined`

```jsx
// 第一种
class Button extends React.Component {
    constructor(props) {
        super(props);
        this.state = {
            number: 1
        }
        // 显示绑定this，才能在回调中使用this
        this.clcikOn().bind(this)
    }

    clcikOn() {
        this.setState({number: 2})
    }

    render() {
        return (
            <button onClick={this.clcikOn} type="submit">Submit</button>
        );
    }
}
```

```jsx
// 第二种
class LoggingButton extends React.Component {
  handleClick() {
    console.log('this is:', this);
  }

  render() {
    // 此语法确保 `handleClick` 内的 `this` 已被绑定。
    return (
      <button onClick={() => this.handleClick()}>
        Click me
      </button>
    );
  }
}
```

#### 受控组件 & 非受控组件

##### 受控组件

在 HTML 中，表单元素（如`<input>`、 `<textarea>` 和 `<select>`）通常自己维护 state，并根据用户输入进行更新。而在 React 中，可变状态（mutable state）通常保存在组件的 state 属性中，并且只能通过使用 setState()来更新。

我们可以把两者结合起来，使 React 的 state 成为“唯一数据源”。渲染表单的 React 组件还控制着用户输入过程中表单发生的操作。被 React 以这种方式控制取值的表单输入元素就叫做“受控组件”。

##### 非受控组件

现取现用就是非受控组件。通过ref取用当前输入的变量，直接拿来用，不经过state状态更新

