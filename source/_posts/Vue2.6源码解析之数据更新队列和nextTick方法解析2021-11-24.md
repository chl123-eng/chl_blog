---
title: Vue2.6源码解析之数据更新队列和nextTick方法解析
date: 2021-11-24
tags:
  - Vue
cover: https://blog-1300014307.cos.ap-guangzhou.myqcloud.com/vue-01.jpg
categories:
  - Vue
  - 总结
  - Vue源码
---

用了这么久的 Vue，我们都可以从官网上面知道，**Vue 在更新 DOM 时是异步执行的,Vue 将开启一个队列，缓冲在同一事件循环中发生的所有数据变更,如果同一个 watcher 被多次触发，只会被推入到队列中一次**，注: 首先要了解下 js 的事件循环和异步任务队列问题

那么内部究竟是如何实现的呢，我们对源代码进行细纠，下面会省略一些不相关的代码。

## 提供第一个 template 范例

如果看了我的事件循环的文章，那么就可以知道任务队列中的异步任务分 task 任务和 microTask 微任务，每一次执行 task 的时候都会执行清空该 task 下的同级 microtask，理解这个原理对源码理解有巨大作用

```html
<div id="app" @click="setData">
  <span>{{ les }}/ {{time}}</span>
  <uptime-day :day="day" :les="les" :time="time" />
</div>

<script>
  const uptimeDay = {
    props: ["time", "les"],
    data() {
      return {
        isTrue: true,
      };
    },
    methods: {
      handleClick() {
        this.isTrue = false;
        this.$nextTick(() => {
          console.log(this.uptimeDay.innerText);
        });
      },
    },
    template: `<div @click="handleClick" ref="uptimeDay">{{ les }} {{ time }}</div>`,
  };
  var app = new Vue({
    el: "#app",
    components: {
      uptimeDay,
    },
    data() {
      return {
        les: "谢小谢",
        time: "now",
      };
    },
    methods: {
      setData() {
        this.les = "谢谢谢";
        this.time = "before";
      },
    },
  });
</script>
```

这里进行一步点击操作 调用 setData 函数

## 初始渲染逻辑粗略解读

> 看过源码的大概都知道，new Vue 之后的最后一步是调用 mountComponent 方法生成 Watcher 订阅者，watcher 传入了一个 updateComponent 函数（内部包含\\\_update,render 方法）作为 getter,初始化时 Watcher 会调用其 getter 方法，进入内部逻辑调用 render 生成 vnode，调用\\\_update 方法对 vnode 进行 patch 操作,具体请看下图。

![](http://www.xiesmallxie.cn/20211103201626.png)

当前模板在\\\_createElement 时，遇到子组件，判断是不是原生标签之后，会调用 Vue.extend 方法重头调用一次 Vue.prorotype.init 方法，所以当前会生成 2 个 Watcher。

## 着重看一下 queueWatcher 执行过程

```js
// src/core/observer/scheduler.js

// 最大更新数量
export const MAX_UPDATE_COUNT = 100

const queue: Array<Watcher> = []
const activatedChildren: Array<Component> = []
let has: { [key: number]: ?true } = {}
let circular: { [key: number]: number } = {}
let waiting = false
let flushing = false
let index = 0
// 重置队列和刷新状态
function resetSchedulerState() {
  index = queue.length = activatedChildren.length = 0
  has = {}
  waiting = flushing = false
}

export let currentFlushTimestamp = 0
let getNow: () => number = Date.now

function flushSchedulerQueue() {
  // 设置当前的时间戳
  currentFlushTimestamp = getNow()
  flushing = true
  let watcher, id
  // 所有watcher根据id升序排列
  // 疑问: 点解要排序？
  // 回答: 1. 组件是从父级更新到子级，组件的渲染顺序是优于父级的，如果某个组件在父组件的观察程序运行期间被销毁，则可以跳过
  queue.sort((a, b) => a.id - b.id)

  // queue长度随时变化
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before()
    }
    id = watcher.id
    has[id] = null
    watcher.run()
  }

  // 保留上次缓存过和更新后的状态实例 为下次触发生命周期做准备
  const activatedQueue = activatedChildren.slice()
  const updatedQueue = queue.slice()
  // 清空状态
  resetSchedulerState()

  // 设置当前组件更activated状态
  callActivatedHooks(activatedQueue)
  // 设置当前组件为已更新状态
  callUpdatedHooks(updatedQueue)
}

// 触发生命周期
function callUpdatedHooks(queue) {
  let i = queue.length
  while (i--) {
    const watcher = queue[i]
    const vm = watcher.vm
    if (vm._watcher === watcher && vm._isMounted && !vm._isDestroyed) {
      callHook(vm, 'updated')
    }
  }
}

export function queueActivatedComponent(vm: Component) {router-view)
  vm._inactive = false
  activatedChildren.push(vm)
}

function callActivatedHooks(queue) {
  for (let i = 0; i < queue.length; i++) {
    queue[i]._inactive = true
    activateChildComponent(queue[i], true /* true */)
  }
}

// 推watcher入栈
export function queueWatcher(watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    // 当前队列没有冲刷的时候
    if (!flushing) {
      // 把当前watcher加入队列
      queue.push(watcher)
    } else {
      // 已冲刷的和已经通过的就删除掉
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // 不是在等待中
    if (!waiting) {
      nextTick(flushSchedulerQueue)
    }
  }
}
```

```js
// src/core/util/next-tick.js

export let isUsingMicroTask = false;
const callbacks = [];
let pending = false;

function flushCallbacks() {
  pending = false;
  const copies = callbacks.slice(0);
  callbacks.length = 0;
  for (let i = 0; i < copies.length; i++) {
    copies[i]();
  }
}

// timeFunc根据环境兼容处理使用任务或者微任务
let timerFunc;

if (typeof Promise !== "undefined" && isNative(Promise)) {
  const p = Promise.resolve();
  timerFunc = () => {
    p.then(flushCallbacks);
    if (isIOS) setTimeout(noop);
  };
  isUsingMicroTask = true;
} else if (
  !isIE &&
  typeof MutationObserver !== "undefined" &&
  (isNative(MutationObserver) ||
    MutationObserver.toString() === "[object MutationObserverConstructor]")
) {
  let counter = 1;
  const observer = new MutationObserver(flushCallbacks);
  const textNode = document.createTextNode(String(counter));
  observer.observe(textNode, {
    characterData: true,
  });
  timerFunc = () => {
    counter = (counter + 1) % 2;
    textNode.data = String(counter);
  };
  isUsingMicroTask = true;
} else if (typeof setImmediate !== "undefined" && isNative(setImmediate)) {
  timerFunc = () => {
    setImmediate(flushCallbacks);
  };
} else {
  // Fallback to setTimeout.
  timerFunc = () => {
    setTimeout(flushCallbacks, 0);
  };
}

// 主处理函数
export function nextTick(cb?: Function, ctx?: Object) {
  let _resolve;
  // 入栈
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx);
      } catch (e) {
        handleError(e, ctx, "nextTick");
      }
    } else if (_resolve) {
      _resolve(ctx);
    }
  });
  // 非等待情况下
  if (!pending) {
    pending = true;
    timerFunc();
  }
}
```

## template 范例解读

1. 根据入口遍历，根组件和子组件会生成两个 watcher 实例，从父到子我们成为 watcher1， watcher2
2. 由上面的 template 可知，根组件#app 和子组件 uptimeDay 都绑定了事件，点击 uptimeDay 组件时，子组件 handleClick 方法已经进入整体 task 队列，内部 isTrue 为 true 时 setter 触发，进入 queueWatcher(watcher2),has[id]为 false,flushing 为 false 进入推入 queue 队列，调用 nextTick(flushSchedulerQueue: 简称该方法为 Fn1) ，nextTick 入栈后 callBack 推入 f1，默认 pending 为 false，调用 timeFunc，此时该函数进入 handleClick 这个 task 下的微任务队列里面，pending 变 true。
3. 随后，下方又调用了 this.$nextTick(() => { console.log(this.$refs.ss.innerText); 简称该方法为 Fn2 }), Fn2 进入 nextTick 之后发现 pending = true，所以被合并进入 callBacks 更新，pending = false 子组件 patch 更新完成 queueWatcher。 初始化 重点: **此时，props 传进来的值并没有更新影响到 dom，打印出来的对象是有滞后性的，必须打印普通类型的值才能正确显示**
4. 接着，事件冒泡到父级，setData 触发进入另外一个 task 队列，接着更改了 les,time 的值。触发 les 的 setter 更新进入 queueWatcher(watcher1)。首先 has[id]为 null, flushing 为 false, watcher1 被推入 queue 队列调用 nextTick，nextTick 内部 pending 为 false，执行 timeFunc，推入微任务队列挂起。随后 time 的 setter 被触发，后进入 queueWatcher，发现 has[id]为 true，waiting 为 true, 不推入队列。重点: **此时，这里就是上文说的 watcher 被多次触发，只推入队列一次**，接着 timeFunc 的微任务开始调用，进入 flushSchedulerQueue 完成更新

## 结论

1. 同 watcher 内更新只更新一次，因为多个 data 改变时，第一次的 reRender 就可以拿到当前实例上的最新值了，无需耗费更多计算资源。
2. 如果外部组件修改，子组件没有绑定当前修改的属性，并且没有应用于 dom，那么也是不会更新到子组件的(这个得看一下 Observe 数据劫持和 patch 的过程才能了解)
3. scheduler.js 中有一段代码是很值得玩味的

```js
// src/core/observer/scheduler.js

// queue长度随时变化
for (index = 0; index < queue.length; index++) {
  watcher = queue[index];
  if (watcher.before) {
    watcher.before();
  }
  id = watcher.id;
  has[id] = null;
  watcher.run();
}
```

这里算是对上面第 4 步的补充，当根组件的实例更新后，子组件也会更新重新进入 queueWatcher，watcher2 会被推入 queue 队列，此时循环体内的 queue 长度增加, 会一层一层往下调用子组件的 watcher 的 run 方法不断地重新 render 来进行 patch 操作，直到没有子组件为止(为什么？因为 vnode 为组件类型时没有 children，无法 patch，必须得重新 render 解析组件)
