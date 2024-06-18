---
title: Vue.2.6源码分析之事件挂载和双向绑定原理
date: 2021-12-06
tags:
  - Vue
cover: https://blog-1300014307.cos.ap-guangzhou.myqcloud.com/vue-01.jpg
categories:
  - Vue
  - 总结
  - Vue源码
---

对于用 vue 的小伙伴来说，v-model 是 vue 开发过程中使用非常频繁的一个指令，它实现了数据的双向绑定。那么现在，我们就来探究一下发生双向绑定的过程是如何实现的

## 设想一下绑定过程

我们都知道，传入 data 的数据会被 Object.defineProperty 转化成 getter,setter
进行监听，v-model 则是需要 input 框支持。当 input 输入时，view 层的数据也会随之动态改变，那么很明显是需要通过一个事件监听方法来触发的。当数据层被触发时，响应，那事件监听事件是在啥时候就开始挂载事件的呢?

## 看看源码中事件怎么监听的

```js
// src\\platforms\\web\\compiler\\directives\\model.js
function genDefaultModel(
  el: ASTElement,
  value: string,
  modifiers: ?ASTModifiers
): ?boolean {
  const type = el.attrsMap.type;

  const { lazy, number, trim } = modifiers || {};
  // 没有携带lazy修饰符并且input类型伟range时就需要进行compsition检测
  const needCompositionGuard = !lazy && type !== "range";
  // 1.如果v-model携带lazy修饰符，那么就自动转成change事件
  // change和input事件的区别: https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/change_event
  // 2.如果不是滑块类型的input 那么就默认是input事件
  const event = lazy ? "change" : type === "range" ? RANGE_TOKEN : "input";
  // 拼装执行语句
  let valueExpression = "$event.target.value";
  if (trim) {
    valueExpression = `$event.target.value.trim()`;
  }
  if (number) {
    valueExpression = `_n(${valueExpression})`;
  }
  // 调用genAssignmentCode语句重组
  // 如果value有值，实际上调用的是this.$set方法
  // if($event.target.composing)return;$set(day, key, $event.target.value)
  let code = genAssignmentCode(value, valueExpression);
  if (needCompositionGuard) {
    // composition事件 即输入法编辑器编辑时 就直接返回空
    code = `if($event.target.composing)return;${code}`;
  }
  addProp(el, "value", `(${value})`);
  // 增加监听事件
  addHandler(el, event, code, null, true);
  // 如果修饰符是trim或number那么应当立即强制更新
  if (trim || number) {
    addHandler(el, "blur", "$forceUpdate()");
  }
}

// src\\compiler\\helpers.js 塞进对象
export function addHandler(
  el: ASTElement,
  name: string,
  value: string,
  modifiers: ?ASTModifiers,
  important?: boolean,
  warn?: ?Function,
  range?: Range,
  dynamic?: boolean
) {
  modifiers = modifiers || emptyObject;

  let events;
  if (modifiers.native) {
    delete modifiers.native;
    events = el.nativeEvents || (el.nativeEvents = {});
  } else {
    events = el.events || (el.events = {});
  }
  // 普通input状态下 返回了{ value: value.trim()}
  const newHandler: any = rangeSetItem({ value: value.trim(), dynamic }, range);
  if (modifiers !== emptyObject) {
    newHandler.modifiers = modifiers;
  }
  // 对象塞入当前监听事件
  const handlers = events[name];
  if (Array.isArray(handlers)) {
    important ? handlers.unshift(newHandler) : handlers.push(newHandler);
  } else if (handlers) {
    events[name] = important ? [newHandler, handlers] : [handlers, newHandler];
  } else {
    events[name] = newHandler;
  }

  el.plain = false;
}

// src/platforms/web/runtime/modules/events.js 生成vnode后增加事件监听
function add(
  name: string,
  handler: Function,
  capture: boolean,
  passive: boolean
) {
  if (useMicrotaskFix) {
    const attachedTimestamp = currentFlushTimestamp;
    const original = handler;
    handler = original._wrapper = function (e) {
      if (
        e.target === e.currentTarget ||
        e.timeStamp >= attachedTimestamp ||
        e.timeStamp <= 0 ||
        e.target.ownerDocument !== document
      ) {
        return original.apply(this, arguments);
      }
    };
  }
  // 增加事件监听
  target.addEventListener(
    name,
    handler,
    supportsPassive ? { capture, passive } : capture
  );
}
```

由上面结合源码可知，在 compiler 编译模板之后生成 ast 语法，生成 ast 语法后会对当前 ast 调用 generate 方法进行一次格式化生成 render 函数，
在 patch 阶段调用 updateDOMListeners 挂载当前监听方法，下面是我总结出来的事件编译挂载的流程图

![](http://www.xiesmallxie.cn/20211206102405.png)

## 再来看看如何数据的更新监听

要想实现数据监听，实时变化。vue2.6 中的策略是这样的，通过 Observer 使用 Object.defineProperty 遍历劫持 data 对象的所有属性，为每个属性生成一个 Dep 消息订阅器，并且在后续挂载中生成一个 Watcher 观察者实例，当第一次调用 render 函数解析模板时，会扫描到模板里面绑定的每个属性，从而触发当前属性的 getter，把当前 Watcher 加入到每个对象的 Dep 的 sub 列表里面，当属性更新之后会就会调用当前属性的 Dep，调用其 sub 里面每个 watcher 的 update 函数来实现更新

```js
// 实现依赖收集器
// src/core/observer/dep.js
let uid = 0;

function remove(list, item) {
  const index = list.indexOf(item);
  list.splice(index, 1);
}

class Dep {
  constructor() {
    this.id = uid++;
    this.sub = [];
  }
  // 订阅所有依赖
  depend() {
    console.log(this);
    if (Dep.target) {
      console.log(Dep.target);
      Dep.target.addDep(this);
    }
  }

  addSub(watcher) {
    this.sub.push(watcher);
  }

  removeSub(watcher) {
    remove(this.sub, watcher);
  }

  notify() {
    console.log(this.sub);
    this.sub.forEach((watcher) => watcher.update());
  }
}

Dep.taget = null;

const targetStack = [];
// 赋值Dep.target主要是为了禁止某些getter依赖项触发到
// Dep.target有值时才能触发depend方法
function pushTarget(watcher) {
  Dep.target = watcher;
  targetStack.push(watcher);
}

function popTarget() {
  targetStack.pop();
  Dep.target = targetStack[targetStack.length - 1];
}

// 实现
// src/core/observer/index.js
function observe(data) {
  let ob;
  // 如果是对象，就对其进行监听
  if (isObject(data)) {
    ob = new Observer(data);
  }
  return ob;
}

class Observer {
  constructor(data) {
    // 初始化依赖收集器
    const dep = new Dep();
    if (Array.isArray(data)) {
    } else {
      this.walk(data);
    }
  }
  // 监听内部的每一个key
  walk(data) {
    const keyList = Object.keys(data);
    for (let i = 0; i < keyList.length; i++) {
      defineReactive(data, keyList[i]);
    }
  }
}

function defineReactive(data, key, value) {
  const dep = new Dep();
  // 没有传value时默认初始化value
  if (arguments.length === 2) {
    value = data[key];
  }
  // 遍历子项
  let child = observe(data[key]);
  // 对内部每个key进行一次代理
  Object.defineProperty(data, key, {
    enumerable: true, // 可枚举
    configurable: false, // 不能再define
    get() {
      // 依赖收集器和watcher互相绑定
      if (Dep.target) {
        dep.depend();
        if (child) {
          child.dep.depend();
        }
      }
      return value;
    },
    set(newValue) {
      // 新旧值不同的情况下就直接更新当前函数
      if (newValue !== value) {
        value = newValue;
        child = observe(data[key]);
        dep.notify();
      }
    },
  });
}

class Watcher {
  constructor(vm, expOrFn, cb, options) {
    this.id = ++id;
    this.vm = vm;
    this.expOrFn = expOrFn;
    this.cb = cb;
    this.lazy = options.lazy || false;
    this.dirty = options.lazy || false;
    // 新旧队列
    this.newDepIdList = [];
    this.depIdList = [];
    this.newDepList = new Set();
    this.depList = new Set();
    console.log(expOrFn);
    this.getter =
      typeof this.expOrFn === "function"
        ? this.expOrFn
        : createExpOrFn(this.vm, this.expOrFn);
    console.log(this.getter);
    this.value = this.lazy ? "" : this.get();
  }

  addDep(dep) {
    // 收集最新的依赖
    if (!this.newDepIdList.includes(dep.id)) {
      this.newDepIdList.push(dep.id);
      this.newDepList.add(dep);
      // 如果队列里面没有
      if (!this.depIdList.includes(dep.id)) {
        dep.addSub(this);
      }
    }
    console.log(dep);
  }
  // 单纯获取最新的值
  get() {
    let value;
    try {
      pushTarget(this);
      value = this.getter.call(this.vm);
    } catch (e) {
    } finally {
      this.cleanDepQueue();
      popTarget();
      console.log("清除了", value, this.getter);
    }
    return value;
  }
  // 更新
  update() {
    if (this.lazy) {
      this.dirty = true;
    } else {
      queueWatcher(this);
    }
  }

  run() {
    const value = this.get();
    if (value !== this.value) {
      const oldValue = this.value;
      this.cb.call(this.vm, value, oldValue);
      this.value = value;
    }
  }

  cleanDepQueue() {
    // 收集了新的依赖 如果新的依赖里面没有旧的 那就从旧的依赖里面去掉
    // 试想一下 页面的绑定了一个值,并且设置了一个v-if指令，在下一次渲染之后，
    // v-if=\"false\"不渲染了 但是上一个的依赖却已经被追踪了,这样就
    // 会追踪额外的依赖项了，所以必须要清除
    for (const dep of this.depList) {
      if (!this.newDepList.has(dep)) {
        dep.removeSub(this);
      }
    }
    // 新的赋值给旧的  清空新依赖列表 新旧对比
    let current = this.depList;
    this.depList = this.newDepList;
    this.newDepList = current;
    this.newDepList.clear();

    current = this.depIdList;
    this.depIdList = this.newDepIdList;
    this.newDepIdList = current;
    this.newDepIdList.length = 0;
  }
}
```

如果看不太懂的话可以参考我自己写的这一份链接 [new Vue 双向绑定示例](https://github.com/ShuHongXie/vue2.6-mvvm-example.git)

## 流程图展示

下面是我自己绘制的流程图，可以通过这张图的调用顺序更简明地了解更新逻辑

![](http://www.xiesmallxie.cn/20211206150549.png)

## 额外的一些注意点

当前实例并没有关注到一些性能上的优化点，一个一个地说明会导致篇幅太长，相关的性能处理在上面的 github 示例链接里面，各位小伙伴有兴趣的话可以点击一下查看，帮我点个 star。

另外可以看看我的其他文章，有一些内容会和当前内容有辅助功能，涉及到了优化相关。

1. [Vue2.6 源码解析之数据更新队列和 nextTick 方法解析](https://xiesmallxie.cn/article/nextTick)
2. [Vue2.6 源码解析之 diff 算法更新过程及其相关问题](https://xiesmallxie.cn/article/vue-diff)
