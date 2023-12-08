渲染逻辑顺序
* [1.render阶段](#1)
  * [1.1.reconcile阶段](#1.1)
  * [1.2.render阶段](#1.2)
* [2.commit阶段](#2)

从render函数开始，将虚拟dom(element)挂载到真实dom节点(element)。
首先构建wipRoot如下：
```js
wipRoot = {
  dom: container,
  props: {
    children: [element]
  },
  alternate: currentRoot
};
```
然后将wipRoot赋给nextUnitOfWork，触发requestIdleCallback函数，开始运行workLoop函数

```js
function workLoop(deadline) {
  let shouldYield = false;
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    shouldYield = deadline.timeRemaining() < 1;
  }

  if (!nextUnitOfWork && wipRoot) {
    commitRoot();
  }

  requestIdleCallback(workLoop);
}
```

因为nextUnitOfWork刚刚被赋值，进入循环，把nextUnitOfWork传入performUnitWork中，计算出下一份nextUnitOfWork，如果这一帧剩下的时间不够就跳出来，并且执行requestIdleCallback(workLoop)，等之后有时间再计算剩余的nextUnitOfWork，直到计算出fiber树，没有剩余的nextOfWork，然后将生成的dom挂载到真实dom节点上。

```js
function performUnitOfWork(fiber) {
  const isFunctionComponent = fiber.type instanceof Function;
  if (isFunctionComponent) {
    updateFunctionComponent(fiber);
  } else {
    updateHostComponent(fiber);
  }
  if (fiber.child) {
    return fiber.child;
  }
  let nextFiber = fiber;
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling;
    }
    nextFiber = nextFiber.parent;
  }
}
```

performUnitOfWork主要做两件事。一、更新fiber。二、更新完当前节点，再返回子节点fiber作为nextUnitOfWork。如果没有子节点了，就开始遍历当前节点的兄弟节点作为nextUnitOfWork，遍历完兄弟节点再返回父节点作为nextUnitOfWork。

总的来说，performUnitOfWork主要是以fiber为单位更新fiber树。

我们第一个fiber如下：
```js
{
  dom: div#root,
  props: {
    children: [
      {
        props: {
          children: []
        },
        type: f Counter()
      }
    ]
  },
  alternate: null
}
```

此时走updateHostComponent(fiber)
```js
function updateHostComponent(fiber) {
  if (!fiber.dom) {
    fiber.dom = createDom(fiber);
  }
  reconcileChildren(fiber, fiber.props.children);
}
```
当前存在fiber.dom为div#root，走<span style="color: blue;">reconcileChildren</span>(fiber, fiber.props.children),fiber.props.children即为

```js
[
  {
    props: {
      children: []
    },
    type: f Counter()
  }
]
```

reconcileChildren遍历当前fiber的children，并将这些children通过prevSibling = newFiber和oldFiber = oldFiber.sibling链接成两个单链表，oldFiber和newFiber

通过比较oldFiber和newFiber同时指向的节点来执行以下操作：
- 如果类型一致，UPDATE fiber
- 如果类型不一致，新节点存在，PLACEMENT fiber
  - 如果存在老节点存在，oldFiber.effectTag赋值为DELETION，并推入deletions数组
- 如果老节点存在，老节点指针指向老节点的兄弟节点
- 如果index=0，新节点指针指向wipFiber
  - 如果新节点存在，prevSibling的sibling指针指向新节点
- prevSibling指向新节点

如此走完，最终wipFiber被成功构建为一棵新的fiber树，但是此时还不完整，因为只构造了

```js
{
  dom: div#root,
  props: {
    children: [
      {
        props: {
          children: []
        },
        type: f Counter()
      }
    ]
  },
  alternate: null,
  child: {
    alternate: null,
    dom: null,
    effectTag: "PLACEMENT",
    parent: (wipFiber),
    props: { children: [] },
    type: f Counter()
  }
}
```

的一级子fiber，子fiber的children还没构建

```js {9}
function performUnitOfWork(fiber) {
  const isFunctionComponent = fiber.type instanceof Function;
  if (isFunctionComponent) {
    updateFunctionComponent(fiber);
  } else {
    updateHostComponent(fiber);
  }
  if (fiber.child) {
    return fiber.child;
  }
  let nextFiber = fiber;
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling;
    }
    nextFiber = nextFiber.parent;
  }
}
```

之后来到第九行返回fiber.child作为nextUnitOfWork,返回到workLoop函数，设置requestIdleCallback(workLoop);在下次空闲时继续workLoop

到了下一次，将

```js
{
  alternate: null,
  dom: null,
  effectTag: "PLACEMENT",
  parent: (wipFiber),
  props: { children: [] },
  type: f Counter()
}
```

作为fiber传入updateFunctionComponent

```js {5}
function updateFunctionComponent(fiber) {
  wipFiber = fiber;
  hookIndex = 0;
  wipFiber.hooks = [];
  const children = [fiber.type(fiber.props)]; // children: [{ type: 'h1', props: {...} }]
  reconcileChildren(fiber, children);
}
```

和之前一样，经过一番reconcile，给wipFiber构建了child，wipFiber如下：

```js
{
  alternate: null,
  dom: null,
  effectTag: "PLACEMENT",
  parent: (wipFiber),
  props: { children: [] },
  type: f Counter()，
  child: {
    alternate: null,
    dom: null,
    effectTag: "PLACEMENT",
    parent: (wipFiber),
    type: "h1",
    props: {
      type: style: 'user-select: none',
      onClick: f onClick(),
      children: [
        {
            "type": "TEXT_ELEMENT",
            "props": {
                "nodeValue": "Count: ",
                "children": []
            }
        },
        {
            "type": "TEXT_ELEMENT",
            "props": {
                "nodeValue": 1,
                "children": []
            }
        }
      ]
    }
  }
}
```

同样返回fiber.child作为nextUnitOfWork

```js {9}
function performUnitOfWork(fiber) {
  const isFunctionComponent = fiber.type instanceof Function;
  if (isFunctionComponent) {
    updateFunctionComponent(fiber);
  } else {
    updateHostComponent(fiber);
  }
  if (fiber.child) {
    return fiber.child;
  }
  let nextFiber = fiber;
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling;
    }
    nextFiber = nextFiber.parent;
  }
}
```

接下来由于h1不是函数式组件，同时h1的fiber还没有dom，所以走performUnitOfWork -> updateHostComponent -> createDom -> updateDom

在updateDom中，将fiber上的旧属性删除，赋上新属性，同时附上事件监听器

这样好了之后再把创建好的dom节点复制给fiber，然后进入reconcileChildren，开始创建fiber的子fiber节点，此时elements有两个TEXT_ELEMENT,所以最后的wipFiber如下：

```js {3-24}
{
  alternate: null,
  child: {
    alternate: null,
    dom: null,
    effectTag: "PLACEMENT",
    parent: (h1 Fiber),
    props: {
      nodeValue: 'Count: ',
      children: []
    },
    sibling: {
      alternate: null,
      dom: null,
      effectTag: "PLACEMENT",
      parent: (h1 Fiber),
      props: {
        nodeValue: 1,
        children: []
      },
      type: "TEXT_ELEMENT"
    },
    type: "TEXT_ELEMENT"
  },
  dom: h1,
  effect: "PLACEMENT",
  parent: (Counter Fiber),
  props: {
    type: style: 'user-select: none',
      onClick: f onClick(),
      children: [
        {
            "type": "TEXT_ELEMENT",
            "props": {
                "nodeValue": "Count: ",
                "children": []
            }
        },
        {
            "type": "TEXT_ELEMENT",
            "props": {
                "nodeValue": 1,
                "children": []
            }
        }
      ]
  }
}
```

照例，在performUnitOfWork返回fiber.child作为下一个nextUnitOfWork

在两次requestIdleCallback触发之后，构建了两个TEXT_ELEMENT的dom节点，最终，wipFiber树构建完成

开始进入commitRoot -> commitWork

在commitWork里按照挂载子节点->挂在兄弟节点->挂载父节点的顺序，递归地把dom节点一个个挂到div#root上，返回commitRoot，把挂载好的wipRoot赋值给currentRoot，并将wipRoot置为null

至此，挂载完成


