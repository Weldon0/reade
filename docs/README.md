# 解析React性能利器—Fiber

> 武晓慧，医药技术部前端开发工程师，喜欢健身，旅游



<a name="20Wha"></a>
#### 什么是刷新率？
大部分显示器屏幕都有固定的刷新率（比如最新的一般在 60Hz），所以浏览器更新最好是在60fps。如果在两次硬件刷新之间浏览器进行两次重绘是没有意义的只会消耗性能。 浏览器会利用这个间隔 16ms(一帧)适当地对绘制进行节流，如果在16ms内做了太多事情，会阻塞渲染，造成页面卡顿， 因此 16ms 就成为页面渲染优化的一个关键时间

---

<a name="6uVE6"></a>
#### 一帧做了哪些事情
![image.png](https://cdn.nlark.com/yuque/0/2020/png/558831/1590477750359-9d95307b-77a6-4fc0-a485-a7ac15b87eba.png#align=left&display=inline&height=279&margin=%5Bobject%20Object%5D&name=image.png&originHeight=377&originWidth=800&size=76893&status=done&style=none&width=591)

- **events**: 点击事件、键盘事件、滚动事件等
- **macro**: 宏任务，如 `setTimeout`
- **micro**: 微任务，如 `Promise`
- **rAF**: **requestAnimationFrame**

window.requestAnimationFrame() 告诉浏览器——你希望执行一个动画，并且要求浏览器在下次重绘之前调用指定的回调函数更新动画。该方法需要传入一个回调函数作为参数，该回调函数会在浏览器下一次重绘之前执行。

- **Layout**: CSS 计算，页面布局
- **Paint**: 页面绘制
- **rIC: requestIdleCallback**

window.requestIdleCallback()方法将在浏览器的空闲时段内调用的函数排队。这使开发者能够在主事件循环上执行后台和低优先级工作，而不会影响延迟关键事件，如动画和输入响应。函数一般会按先进先调用的顺序执行，然而，**如果回调函数指定了执行超时时间timeout，则有可能为了在超时前执行函数而打乱执行顺序**。<br />

> 一个帧内要做这么多事情……如果js执行时间过长超过16ms，就会block住，那么就会丢掉一次帧的绘制
> 宏任务的执行总在微任务之后，但是与其他的顺序不太确定
> TIPS：协调的概念：比较虚拟DOM树，找出需要变更的节点，更新，称为协调(Reconcliation)




---

<a name="RblL9"></a>
#### React16之前的协调

<br />![190406_5gkdlca7k824he218jca83109fb39_550x280.gif](https://cdn.nlark.com/yuque/0/2020/gif/558831/1590480083024-35efcb36-5bf7-4aba-b59b-17e81316e117.gif#align=left&display=inline&height=200&margin=%5Bobject%20Object%5D&name=190406_5gkdlca7k824he218jca83109fb39_550x280.gif&originHeight=280&originWidth=550&size=1304415&status=done&style=none&width=393)<br />

- 特点：
   - 递归调用，通过React DOM 树级关系构成的栈递归
   - 在virtualDOM的比对过程中，发现一个instance有更新，会立即执行DOM操作。
   - 同步更新，没发打断
- 代码示例

有下面这样一个Component组件，用他来模拟DOM Diff过程：<br />

```jsx
const Component = (
  <div id="A1">
    <div id="B1">
      <div id="C1"></div>
      <div id="C2"></div>
    </div>
    <div id="B2"></div>
  </div>
)
```
Diff过程：<br />上面定义的Component组件会首先通过Babel转成React.CreateElement生成ReactElement，也就是我们口中的虚拟DOM(virtualDOM)，如下类似root的结构(下面里面属性做了很多简化，只展示了结构)<br />
<br />

```jsx


let root = {
  key: 'A1',
  children: [
    {
      key: 'B1',
      children: [
        {
          key: 'C1',
          children: [],
        },
        {
          key: 'C2',
          children: [],
        }
      ],
    },
    {
      key: 'B2',
      children: [],
    }
  ],
};

// 深度优先遍历
function walk(vdom) {
  doWork(vdom);
  vdom.children.forEach(child => {
    walk(child);
  })
}
// 更新操作
function doWork(vdom) {
  console.log(vdom.key);
}

walk(root);
```
<br />

- 缺点：

根据上面代码会发现，如果有大量更新或者有很深的组件结构树，执行diff操作的执行栈会越来越深并不能及时释放，那么 js 将一直占用主线程，一直要等到整棵 virtualDOM 树计算完成之后，才能把执行权交给渲染引擎，这就会导致用户的交互操作以及页面动画得不到响应，就会有明显感觉卡顿(掉帧)，影响用户体验。

- 解决：

把一个耗时长的任务分成很多小片，每一个小片的运行时间很短，虽然总时间依然很长，但是在每个小片执行完之后，都给其他任务一个执行的机会，这样唯一的线程就不会被独占，其他任务依然有运行的机会，所以React在15版本更新16版本时候推出了Fiber协调的概念。<br />


---

<a name="xyWkE"></a>
#### Fiber概念
**Fiber是对React核心算法的重构，2年重构的产物就是Fiber Reconciler**<br />核心目标：扩大其适用性，包括动画，布局和手势。<br />
<br />![fiber.gif](https://cdn.nlark.com/yuque/0/2020/gif/558831/1590481310158-67410095-5670-4910-a6b3-03db4a61a0bb.gif#align=left&display=inline&height=229&margin=%5Bobject%20Object%5D&name=fiber.gif&originHeight=280&originWidth=550&size=942073&status=done&style=none&width=449)<br />
<br />
<br />

- 把可中断的工作拆分成小任务
- 对正在做的工作调整优先次序、重做、复用上次（做了一半的）成果
- 在父子任务之间从容切换（yield back and forth），以支持React执行过程中的布局刷新
- 支持`render()`返回多个元素
- 更好地支持error boundary
> 每一个virtualDOM节点内部都会生成对应的Fiber


---



<a name="tQYWt"></a>
#### Fiber前置知识
怎么中断一个任务：实现一个类似于Fiber可中断的workLoop
```javascript

function sleep(delay) {
  for (let start = Date.now(); Date.now() - start <= delay;) {}
}

// 每一个子项可以认为是一个fiber
const works = [
  () => {
    console.log('第一个任务开始');
    sleep(20);
    console.log('第一个任务结束');
  },
  () => {
    console.log('第2个任务开始');
    sleep(20);
    console.log('第2个任务结束');
  },
  () => {
    console.log('第3个任务开始');
    sleep(20);
    console.log('第3个任务结束');
  },
];

window.requestIdleCallback(workLoop, { timeout: 1000});

function workLoop(deadLine) {
  console.log('本帧的剩余时间剩', parseInt(deadLine.timeRemaining()));
  /*
  * deadLine {
  *   timeRemaining(), 返回此帧还剩下多少ms供用户使用
  *   didTimeout 返回cb任务是否超时
  * }
  */
  while ((deadLine.timeRemaining() > 0 || deadLine.didTimeout) && works.length > 0) { // 对象 两个属性 timeRemaining()
    performUnitOfWord();
  }

  if (works.length > 0) {
    window.requestIdleCallback(workLoop, { timeout: 1000});
  }
}

function performUnitOfWord() {
  works.shift()(); // 取出第一个元素执行
}
```


- 单链表
   - 存储数据的数据结构
   - 数据以节点的形式表示，每个节点的构成：元素+指针(后续元素存储位置)，元素就是存储数据的存储单元
   - 单链表是Fiber中很重要的一个数据结构，很多异步更新逻辑都是通过单链表结构来实现的(setState中的UpdateQueue更新链表也是基于单链表结构)



模拟一个类似React中setState批量更新的逻辑
```javascript
/**
  Fiber很多地方用到链表（单链表），尾指针没有指向
 */

class Update {
  constructor(payload, nextUpdate) {
    this.payload = payload;
    this.nextUpdate = nextUpdate; // 下一个节点的指针
  }
}

class UpdateQueue {
  constructor(payload) {
    this.baseState = null; // 原状态
    this.firstUpdate = null; // 第一次更新
    this.lastUpdate = null; // 最后一次更新
  }

  enqueueUpdate(update) {
    if (this.firstUpdate === null) {
      this.firstUpdate = this.lastUpdate =update;
    } else {
      this.lastUpdate.nextUpdate = update;
      this.lastUpdate = update;
    }
  }
  // 获取老状态，遍历链表，进行更新
  forceUpdate() {
    let currentState = this.baseState || {}; // 初始状态
    let currentUpdate = this.firstUpdate;
    while (currentUpdate) {
      let nextState = typeof currentUpdate.payload === 'function'
                      ? currentUpdate.payload(currentState)
                      : currentUpdate.payload;
      currentState = {
        ...currentState,
        ...nextState,
      }; // 使用当前更新得到最新的状态
      currentUpdate = currentUpdate.nextUpdate; // 找下一个节点
    }
    this.firstUpdate = this.lastUpdate = null; // 更新结束清空链表
    this.baseState = currentState;
    return currentState;
  }
}
// 链表可以中断和恢复
// 每次setState都会通过一个链表保存起来，最后合并
// enqueueUpdate可以类比为setState操作
let queue = new UpdateQueue();
queue.enqueueUpdate(new Update({ name: '微医集团' }));
queue.enqueueUpdate(new Update({ number: 0 }));
queue.enqueueUpdate(new Update(state => ({ number: state.number + 1 })));
queue.enqueueUpdate(new Update(state => ({ number: state.number + 1 })));
console.log(queue)
queue.forceUpdate();
```
![image.png](https://cdn.nlark.com/yuque/0/2020/png/558831/1590481556734-be656af7-b662-4e67-aa0a-c124fee33b2b.png#align=left&display=inline&height=286&margin=%5Bobject%20Object%5D&name=image.png&originHeight=286&originWidth=746&size=34869&status=done&style=none&width=746)
> 思考：为什么setState在合成事件中会是异步去更新的？
> 解释：我们通过伪代码发现，每次的setState并没有对UpdataQueue中的state做任何更新，只是把每次需要更新的值(或函数)，放到了UpdataQueue的链表上面，在执行forceUpdate的时候再做统一处理，处理完之后更新state，所以没有执行forceUpdate之前，我们拿到的state都不是我们预期想要的state。


---



<a name="vmUny"></a>
#### React中的Fiber

1. **Fiber的两个执行阶段**
   - 协调Reconcile(render)：对virtualDOM操作阶段，对应到新的调度算法中，就是通过 Diff Fiber Tree **找出要做的更新工作**，**生成Fiber树**。这是一个js计算过程，计算结果可以被缓存，计算过程可以被打断，也可以恢复执行。 所以，React介绍 Fiber Reconciler 调度算法时，有提到新算法具有可拆分、可中断任务的新特性，就是因为这部分的工作是一个纯js计算过程，**所以是可以被缓存、被打断和恢复的**
   - 提交更新commit: 渲染阶段，拿到更新工作，提交更新并调用对应渲染模块（React-DOM）进行渲染。为了防止页面抖动，该**过程是同步且不能被打断**。
2. React中定义一个组件用来创建Fiber
```jsx
const Component = (
  <div id="A1">
    A1
    <div id="B1">
      B1
      <div id="C1">C1</div>
      <div id="C2">C2</div>
    </div>
    <div id="B2">B2</div>
  </div>
)
```

3. 上面定义的Component是一个组件，babel解析时候会默认调用React.createElement()方法，最终生成下面代码所示这样的virtualDOM结构并传给ReactDOM.render()方法进行调度
```json
{
  "type":"div",
  "key":null,
  "ref":null,
  "props": {
    "id":"A1",
    "children":[
      "A1",
      {
        "type":"div",
        "key":null,
        "ref":null,
        "props":{
          "id":"B1",
          "children":[
            "B1",
            {
              "type":"div",
              "key":null,
              "ref":null,
              "props":{
                  "id":"C1",
                  "children":"C1"
              },
              "_owner":null,
              "_store":{

              }
            },
            {
              "type":"div",
              "key":null,
              "ref":null,
              "props":{
                  "id":"C2",
                  "children":"C2"
              },
              "_owner":null,
              "_store":{

              }
            }
        ]
      },
      "_owner":null,
      "_store":{

      }
    },
    {
      "type":"div",
      "key":null,
      "ref":null,
      "props":{
          "id":"B2",
          "children":"B2"
      },
      "_owner":null,
      "_store":{

      }
    }
    ]
  },
  "_owner":null,
  "_store":{

  }
}
```

4. render方法会接受virtualDOM，为每个virtualDOM创建Fiber(render阶段)，并且按照一定关系连接接起来
4. fiber结构
```json
class FiberNode {
  constructor(tag, pendingProps, key, mode) {
    // 实例属性
    this.tag = tag; // 标记不同组件类型，如classComponent，functionComponent
    this.key = key; // react元素上的key 就是jsx上写的那个key，也就是最终ReactElement上的
    this.elementType = null; // createElement的第一个参数，ReactElement上的type
    this.type = null; // 表示fiber的真实类型 ，elementType基本一样
    this.stateNode = null; // 实例对象，比如class组件new完后就挂载在这个属性上面，如果是RootFiber，那么它上面挂的是FiberRoot

    // fiber
    this.return = null; // 父节点，指向上一个fiber
    this.child = null; // 子节点，指向自身下面的第一个fiber
    this.sibling = null; // 兄弟组件, 指向一个兄弟节点
    
    this.index = 0; //  一般如果没有兄弟节点的话是0 当某个父节点下的子节点是数组类型的时候会给每个子节点一个index，index和key要一起做diff

    this.ref = null; // reactElement上的ref属性

    this.pendingProps = pendingProps; // 新的props
    this.memoizedProps = null; // 旧的props
    this.updateQueue = null; // fiber上的更新队列 执行一次setState就会往这个属性上挂一个新的更新, 每条更新最终会形成一个链表结构，最后做批量更新
    this.memoizedState = null; // 对应memoizedProps，上次渲染的state，相当于当前的state，理解成prev和next的关系

    this.mode = mode; // 表示当前组件下的子组件的渲染方式

    // effects

    this.effectTag = NoEffect; // 表示当前fiber要进行何种更新
    this.nextEffect = null; // 指向下个需要更新的fiber
  
    this.firstEffect = null; // 指向所有子节点里，需要更新的fiber里的第一个
    this.lastEffect = null; // 指向所有子节点中需要更新的fiber的最后一个

    this.expirationTime = NoWork; // 过期时间，代表任务在未来的哪个时间点应该被完成
    this.childExpirationTime = NoWork; // child过期时间

    this.alternate = null; // current树和workInprogress树之间的相互引用
  }
}
```

<br />Fiber有很多属性，所有子节点Fiber的连接接是通过child，return，siblint链接起来，alternate连接的是每一次更新的状态，用来对比每次状态更新以及缓存，我们使用节点的id来标识每个Fiber组件，转换为Fiber最终会生成如下图所示的结构，也是类似于virtualDOM结构的，构建的顺序是先child => sibling => return，如果当前节点没有child了，这个节点就会完成。<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/558831/1592742939126-1fa8056c-ed6c-4184-8e85-494472d1eb92.png#align=left&display=inline&height=635&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1270&originWidth=2010&size=358714&status=done&style=stroke&width=1005)

Fiber树

- 收集依赖

收集依赖是在生成Fiber过程(render阶段)中同时完成的，按照每个节点完成的顺序来构建链表，每个有了Fiber的组件通过自己的nextEffect指向下一个需要更新的组件，每一个父节点都有firstEffect和lastEffect来连接自己子节点的第一次更新和最后一次更新，最终会生成下图这样的更新链表<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/558831/1592451943710-32fde591-362d-4ec1-955e-b1671c95be76.png#align=left&display=inline&height=530&margin=%5Bobject%20Object%5D&name=image.png&originHeight=530&originWidth=1008&size=99990&status=done&style=stroke&width=1008)<br />副作用链表(更新链表)

- 提交更新commit

全部节点创建完Fiber之后，会进入commit阶段，会从root的fistEffect（所有节点的第一个副作用阶段）开始更新，然后找firstEffect的nextEffect节点，以此类推，一气呵成全部更新完，然后清空更新链表，完成此次更新，这个过程不可打断。

---

<a name="oa2Js"></a>
#### 总结
以上是React大概工作流程，主要以首次更新全部节点需要创建Fiber来讨论，后续会更新：基于Fiber的diff、React中合成事件、各种类型组件(类组件，Function组件)、hooks、事件优先级(expirationTime)在内部如何调度相关。<br />

