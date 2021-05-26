---
- 为了避免混淆，使用 "element" 来表示React element，使用 "node" 来表示DOM element
- 类型为	`TEXT_ELEMENT` 的当作是原始值

---

# Build your own React（译文）

原文地址: https://pomb.us/build-your-own-react/

## 概述

> 以下译文仅是本人学习所用，如有错误还望谅解

- 我们将要一步一步的重头开始重写react。遵循真实的React代码中的架构，但是没有所有的优化和非必要的功能

- 如果您阅读过我以前的任何有关 "build your own React" 的文章，这个区别在于该文章是基于React16.8，因此我们现在能使用hooks和删除所有与类相关的代码

- 你可以在旧的博客文章和Didact仓库中的代码里面找到历史记录。而且还有一个演讲也是涉及相同的内容，但这是一篇独立的文章

- 让我们从头开始一步步将这些东西添加到我们的react版本中 

  ​	1 .  The `createElement` Function

  ​	2 .  The `render` Function

  ​	3 .  Concurrent Mode

  ​	4 .  Fibers

  ​	5 .  Render and Commit Phases

  ​	6 .  Reconciliation

  ​	7 .  Function Components

  ​	8 .  Hooks


## 常见的三行代码

- 但首先让我们来复习一些基础概念。如果你对React，JSX and DOM element如何工作有了清晰的了解，你可以跳过这部分

```javascript
  // 1. defines a React Element（定义虚拟DOM）
  const element = <h1 title="foo">Hello</h1>;
  // 2. gets a node from the DOM
  const container = document.getElementById("root");
  // 3. renders the React element into the container
  ReactDOM.render(element,container);
```

### JSX -> React.createElement

> vanilla JS 仅仅是一个俚语，就是纯 JS，🐕

- 接下来，来我们用原始的JavaScript代码来替换掉React特定的代码，来完成同样功能

  - 看第一行 `const element = <h1 title="foo">Hello</h1>` 

    > 它是通过JSX语法定义的，甚至不是一个有效JS代码，因此第一步我们需要用有效的JS替换它

  - 通常JSX语法是通过Babel来转化为有效的JS代码的

    > 将JSX语法转化为相应的React.createElement(tagNameString, props, children) 即可 

    ```javascript
    // jsx -> js
    // const element = <h1 title="foo">Hello</h1>;
    const element = React.createElement(
      "h1",
      { title: "foo" },
      "Hello"
    );
    ```

- React.createElement使用传递的参数创建并返回一个对象，以及处理一些验证之外，没有额外的功能。因此也可以将上述的函数调用转化为返回的对象。


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

### ReactDOM.render

- 下一步，我们需要用原生JS来替换掉ReactDOM.render

> render的作用是将 Virtual DOM 转化为 DOM Node，并且将其挂载到指定的DOM元素上

```javascript
  // 1. 通过type创建DOM node
  const node = document.createElement(element.type);
  // 2. 给DOM node添加props中除了children的key-value
  node["title"] = element.props.title;

  // 3. 给DOM node添加children属性,采用创建文本节点的方法,文本内容存储在nodeValue属性上
  const text = document.createTextNode("");
  text["nodeValue"] = element.props.children;

  // 4. 将children Node添加到parentElementNode,然后将其挂载到容器上
  node.appendChild(text);
  container.appendChild(node);

```



## The createElement Function

```javascript
  const element = (
    <div id="foo">
      <a>bar</a>
      <b />
    </div>
  );

  const container = document.getElementById("root");
  ReactDOM.render(element, container);
```

- 仅需要在文件中加入如下注释，那么jsx语法就会转化为指定的Didact.createElement调用了

```javascript
/** @jsx Didact.createElement */
```

- 为实现Didact.createElement方法，首先来看React.createElement方法是如何工作的

```javascript
React.createElement(
  type, // type可以是标签的字符名 | React component | React fragment
  props, // 收集的时候是一个object，返回虚拟DOM时用...扩展运算符展开
  ...children, // 收集后续参数并将其转为数组，返回时需要区分原始值和引用值（原始值调用createTextNode方法生成文本节点）
)
```

```java
function createTextNode(text) {
  return {
    type: "TEXT_ELEMENT",
    props: {
      nodeValue: text,
      children: []
    }
  };
}
function createElement(type, props, ...children) {
  return {
    type,
    ...props,
    children: children.map(child => (
      typeof child === 'object' 
          ? child 
          : createTextNode(child)
    ))
  };
}

const Didact = {
  createElement    
};

```

- 此时便可以通过如下代码对我们自己的createElement进行测试了

```javascript
/** @jsx Didact.createElement */  
const element = (
  <div id="foo">
    <a>bar</a>
    <b />
  </div>
);

console.log(element);

```



## The Render Function

```javascript
const container = document.getElementById("root");

ReactDOM.render(element, container);
```

- render函数接收element和container两个参数，然后将其挂载到container中

  > 采用深度递归构造元素树

```javascript
function render (element, container) {
  const dom = element.type === "TEXT_ELEMENT" 
  	? document.createTextNode("") // 简化原始值与标签名字的区分操作
    : document.createElement(element.type);
  
  // 将element的props全部分配到dom中
  Object.keys(props).map(key => {
    if (key !== "children") {
      dom[key] = element.props[key];
    }
  })
    
  // 递归添加子节点
  element.props.children(child => render(child, dom));
    
  container.appendChild(dom);
}
```



## *Concurrent Mode

- 当我们需要渲染的元素树非常深的时候，因为是同步模式，所以浏览器会等待该元素树执行完，才能去执行一些高优先级的操作（如：处理用户输入或保持动画效果流畅），那该如何处理呢？
- `react给出了一个答案：Concurrent Mode：这就是之前操作系统中学的并发呀，这666`

> 将一个工作划分为几个小单元，在每个单元完成后。如果浏览器需要执行其它的操作我们就让它中断元素树的渲染

- 我们将使用 `requestIdleCallback` 进行循环（React现在使用scheduler package来代替它了，但效果是一样的）

> requestIdleCallback是在浏览器主进程空闲时，运行指定的回调函数

```javascript
let nextUnitOfWork = null;

function workLoop (IdleDeadline) {
  let shouldYield = false;
  while (nextUnitOfWork && !shouldYield) {
    // 执行当前工作单元，返回下一个工作单元
    nextUnitOfWork = performUnitOfWork(
      nextUnitOfWork
    );
    // timeRemaining函数能获取距离浏览器主进程需要工作的时间
    shouldYield = IdleDeadline.timeRemaining() < 1;
  }
  requestIdleCallback(workLoop);
}

requestIdleCallback(workLoop);

function performUnitOfWork (nextUnitOfWork) {
  // TODO
}
```



## Fibers

- 既然将一个工作划分为了几个小单元，那么应该使用什么样的数据结构去组织它们呢 ？

  > react给出的答案是，a fiber tree

### Fiber Tree

```javascript
    Didact.render({
      <div>
        <h1>
          <p></p>
          <a></a>
        </h1>
        <h2></h2>
      </div>
    }, container)
```


  ![image-20210516222031201](https://user-images.githubusercontent.com/48879887/119619059-30225580-be36-11eb-97f9-560cfba7edf6.png)



- 每个元素都有一个fiber，并且每一个fiber都将成为一个工作单元

  > fiber === unit of work，且包含 dom

- Fiber Tree数据结构设计的目的如下

  > firstChild和nextSibling是为了更好的寻找到nextUnitOfWork
  >
  > 而parent是为了fiber.dom能添加到fiber.parent.dom上而准备的！

  ```javascript
  // []代表不主动在fiber中设置该属性，就没有它 
  {
    dom,
    props: {
      [otherProps],
      children: []
    },
    [firstChild],
    [nextSibling],
    [parent]
  }
  ```

### performUnitOfWork

- 在render中，我们将创建root fiber作为nextUnitOfWork，然后其余工作交付给performUnitOfWork函数，对于每一个fiber，都会做如下三件事

  - 1 .  添加DOM到fiber中（fiber.dom and fiber.parent）

  - 2 .  根据 `fiber.children` 创建 `fibers`

    > 建立parent 、nextSibling 、firstChild的链接

  - 3 .  返回下一个工作单元

    > fiber寻找工作单元的规则如下优先级

    - fiber.firstChild > fiber.nextSibling > fiber.parent.nextSibling（uncle）> fiber.parent.parent.nextSibling > ... > root fiber（render 函数工作完成）

- 完善performUnitOfWork

```javascript
let nextUnitOfWork = null;
function render (element,container) {
  nextUnitOfWork = {
    dom: container,
    props: {
      children: [element]
    },
  };
}

// conCurrent Mode
function workLoop (IdleDeadline) {
  let shouldYield = false;
  while (!shouldYield && nextUnitOfWork) {
    nextUnitOfWork = performUnitOfWork(
      nextUnitOfWork
    );   
    shouldYield = IdleDeadline.timeRemaining() < 1;
  }
    
  requestIdleCallback(workLoop);
}

// a fiber tree
function performUnitOfWork (fiber) {
  // 1. add dom to fiber
  if (!fiber.dom) {
    // 根据fiber创建DOM
    fiber.dom = createDOM(fiber);
  }    
  if (fiber.parent) {
    fiber.parent.dom.appenChild(fiber.dom);
  }
    
  // 2. create fibers from the element's children
  let prevfiber = null;
  fiber.props.children.forEach((child,index) => {
    const newfiber = {
      dom: null,
      type: child.type,
      props: child.props,
      parent: fiber,
    };
    
    if (index === 0) {
      fiber.firstChild = newfiber;
    } else {
      prevfiber.nextSibling = newfiber;
    }
      
    prevfiber = newfiber;  
      
  })
    
  // 3. return next unit of work
  if (fiber.firstChild) {
    return fiber.firstChild;
  } else {
    let nextfiber = fiber;
    while (nextfiber) {
      if (nextfiber.nextSibling) {
        return nextfiber.nextSibling;
      }  
      nextfiber = nextfiber.parent;
    }
  }
}

requestIdleCallback(workLoop);
```



## *Render and Commit Phases

- 每执行一个unitofwork就会添加一个元素到DOM上，而在整棵树渲染完成前，浏览器会中断渲染工作，这种情况下用户看到  `不完整的UI展示`

  > 因此才需要修改performUnitOfWork方法中添加现有的dom到父节点上

  ```javascript
  function performUnitOfWork (fiber) {
      if (!fiber.dom) {
          fiber.dom = createDOM(fiber);
      }
      
      // 此处就需要想办法最后再统一添加到【容器节点】上，因此要移除
      /*
      	if (fiber.parent) {
          	fiber.parent.dom.appendChild(fiber.dom);
      	}
      */
      
     	...
  }
  ```

- 需要一个全局变量wipRoot来记录根fiber

  ```javascript
  function render(element, container) {
      wipRoot = {
          dom: container,
          props: {
              children: [element]
          },
      }
      nextUnitOfWork = wipRoot;
  }
  
  let wipRoot = null;
  ```

- commitRoot调度commitWork来提交每一个工作单元

  ```javascript
  // 从根fiber的firstChild开始
  function commitRoot () {
      commitWork(wipRoot.firstChild);
      wipRoot = null;
  }
  // 将fiber.dom提交到fiber.parent.dom中
  function commitWork (fiber) {
      if (!fiber) {
          return;
      }
      
      const domParent = fiber.parent.dom;
      domParent.appendChild(fiber.dom);
      commitWork(fiber.firstChild);
      commitWork(fiber.nextSibling);
  }
  ```

- 当全部工作单元执行完后，再提交整个dom树

  ```javascript
  function workLoop (IdleDeadline) {
      let shouldYield = false;
      
      while (nextUnitOfWork && !shouldYield) {
          nextUnitOfWork = performUnitOfWork(
              nextUnitOfWork
          );
          shouldYield = IdleDeadline.timeRemaining() < 1; 
      }
      
   	if (!nextUnitOfWork && wipRoot) {
          CommitRoot();
      }
      
      requestIdleCallback(workLoop);
  }
  ```



## *Reconciliation

> 如何尽可能高效的实现dom的更新 、删除操作，不可能每次都全部【推倒重来】，这是不高效的，因此可以通过比较新旧fiber tree从而复用已渲染的dom
>
> - 通过为每一个fiber提供一个【alternate属性】来【记录旧的fiber对象】（在fiber tree的位置上是相同的）
> - 通过一个全局变量【currentRoot】来【记录当前的根fiber】

```javascript
function commitRoot () {
    commitWork(wipRoot);
    currentRoot = wipRoot; // 记录当前的根fiber
    wipRoot = null;
}

function render (element,container) {
	wipRoot = {
        dom: container,
        props: {
            children: [element],
        },
        alternate: currentRoot, // 记录旧的根fiber对象
    }
    nextUnitOfWork = wipRoot;
}

let currentRoot = null;
```

- reconcileChildren(wipFiber, elements)

```markdown
  1. effectTag: "UPDATE"，如果type相同，保留以前的dom，仅仅更新dom属性
  2. effectTag: "PLACEMENT"，如果type不同，存在element，新建dom
  3. effectTag: "DELETION"，如果type不同，存在oldFiber，删除oldFiber的dom
  4. 建立fiber tree的firstChild 和 nextSibling
```

- 在commit phase步骤中必须针对effectTag标志进行相关的处理



## Function Components

- 当jsx标签用函数式组件App来表示时，babel将其转为createElement调用时，传入的type参数为它的构造函数App

  > function App(props) {
  >
  > ​	...
  >
  > }
  >
  > => App

- 若用jsx语法表示标签时，babel将其转为createElement调用时，传入的type参数用字符串来表示标签的名字

  > h1标签 => "h1"



## Hooks

- 调用setState方法后

  ```javascript
  wipRoot = currentRoot;
  nextUnitOfWork = wipRoot;
  deletions = [];
  ```

- 每个函数式组件的fiber都通过一个hooks数组来维护hooks的信息（state, queue）

