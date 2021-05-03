# Build your own React（译文）

## 引言

> Rodrigo Pombo <br>November 13, 2019

- 原文地址: https://pomb.us/build-your-own-react/

## 正文

> 以下译文仅是本人学习所用，如有错误还望谅解

- 我们将要一步一步的重头开始重写react。遵循真实的React代码中的架构，但是没有所有的优化和非必要的功能。

- 如果您阅读过我以前的任何有关 "build your own React" 的文章，这个区别在于该文章是基于React16.8，因此我们现在能使用hooks和删除所有与类相关的代码。

- 你可以在旧的博客文章和Didact仓库中的代码里面找到历史记录。而且还有一个演讲也是涉及相同的内容，但这是一篇独立的文章。

- 让我们从头开始一步步将这些东西添加到我们的react版本中

  1 . The `createElement` Function
  2 . The `render` Function
  3 . Concurrent Mode
  4 . Fibers
  5 . Render and Commit Phases
  6 . Reconciliation
  7 . Function Components
  8 . Hooks
    
    
- 但首先让我们来复习一些基础概念。如果你对React，JSX and DOM element如何工作有了清晰的了解，你可以跳过这部分
  - 我们将使用此React app（仅仅三行代码）
```javascript
  // 1. defines a React Element（定义虚拟DOM）
  const element = <h1 title="foo">Hello</h1>;

  // 2. gets a node from the DOM
  const container = document.getElementById("root");

  // 3. renders the React element into the container
  ReactDOM.render(element,container);
```

> vanilla JS 仅仅是一个俚语，就是纯 JS，🐕

- 接下来，来我们用原始的JavaScript代码来替换掉React特定的代码，来完成同样功能
  - 看第一行 `const element = <h1 title="foo">Hello</h1>` ，它是通过JSX语法定义的，甚至不是一个有效JavaScript代码，因此第一步我们需要用有效的JS替换它
  - 通常JSX语法是通过Babel来转化为有效的JS代码的。而内部的操作其实很简单：通过调用createElement(tagNameString,props,children) 来进行转化

```javascript
  /* 
    通过React.createElement创建一个对象，除了一些验证外，它跟JSX标签的作用是一样的
    因此我们能安全的替换它
  */
  const element = React.createElement(
    "h1",
    { title: "foo" },
    "Hello"
  );
```

- 如下就是调用 `createElement` 返回的元素，它有type和props属性
  （当然它还有更多的属性，但我们只关心这两个属性）
  - `type是一个string`，用于指定我们需要创建的DOM节点类型。它也是我们创建HTML元素时传递给document.createElement的tagName。`type还可以是一个function`
  - `prop是一个object`，它有JSX中所有的属性，以及一个children
    - 在子节点仅有一个时，`children是一个string of TagName`
      其它情况则是`包含所有子节点TagName的数组`

```javascript
  // 调用React.createElement返回的object
  const element = {
      type: "h1", 
      props: {
        title: "foo",
      	children: "Hello"
      },
      ...
  };
      
```

- 下一步，我们需要用原生JS来替换掉ReactDOM.render
> render的作用是将 Virtual DOM 转化为 DOM Node，并且将其挂载到指定的DOM元素上

```javascript
  // 1. 通过type创建DOM node
  const node = document.createElement(element.type);
  // 2. 给DOM node添加props中除了children的key-value
  node["title"] = element.props.title;

  // 3. 给DOM node添加children属性,采用创建文本节点的方法，而不是innerText（猜测是递归）
  const text = document.createTextNode("");
  text["nodeValue"] = element.props.children;

  // 4. 将children Node添加到parentElementNode,然后将其挂载到容器上
  node.appendChild(text);
  container.appendChild(node);

```

> *为了避免混淆，使用 "element" 来表示React element，使用 "node" 来表示DOM element

- 疑问：添加children属性哪里，应该是要采取递归添加节点才对呀？
