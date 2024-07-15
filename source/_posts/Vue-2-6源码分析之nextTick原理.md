---
title: Vue.2.6源码分析之nextTick原理
date: 2024-07-10 16:03:25
tags:
  - Vue
  - Vue源码
cover: https://blog-1300014307.cos.ap-guangzhou.myqcloud.com/vue-01.jpg
categories:
  - Vue
  - 总结
  - Vue源码
---

### 前言

Vue.js 中的 $nextTick 函数是一个非常重要的 API，它用于延迟回调的执行直到下次 DOM 更新循环之后。简单来说，$nextTick 会在 DOM 完全更新后执行其回调函数。这是 Vue.js 异步更新队列的一个具体应用。

### 源码实现

##### watcher监听数据

实现： 数据更新多次，vm._update(vm._render())只执行一次，即DOM只更新一次

watcher.js文件
```
class watcher {
    constructor(vm, updateComponent,cb,options){
        .....
    }
    run(){
        this.get()
    }
    get(){
        this.getter()//外部传进来的vm._update(vm._render())
    }
    //更新
    update(){
        queueWatcher(this)
    }
}

let queue = [];//将需要批量更新的watcher存放到一个队列中
let has = {};
let pending = false

function flushWatcher(){
    queue.forEach(item => {item.run()})
    queue = [];
    has = {}
    pending = false
}
function queueWatcher(watcher){
    let id = watcher.id //每个组件共用同一个watcher
    if(!has[id]){//去重
        queue.push(watcher);
        has[id] = true;
        //防抖
        if(!pending){
            nextTick(flushWatcher) //相当于定时器,异步
        }
        pending = true
    }
}
```

##### 将操作放入异步队列中，进行同步执行

nextTick.js文件
```
let callBack = [] //列队：1、是vue自己的nextTick, 2、$nextTick回调函数
let pending = false

function flush(){
    callBack.forEach(cb => {
        cb();
    })
    pending = false
}
let timerFunc
//处理兼容问题
if(Promise){
    timerFunc = () => {
        Promise.resolve().then(flush)//使所有函数同步操作
    }
}else if(MutationObserver){//h5异步方法，监听dom变化，监控完毕后进行异步更新
    let observe = new MutationObserver(flush)
    let texNode = document.createTextNode(1)
    observe.observe(texNode,{characterData: true});
    timerFunc = () =>{
        texNode.textContent = 2
    }
}else if(setImmediate){//ie
    timerFunc = () => {
        setImmediate(flush)
    }
}

export function nextTick(cb){

    callBack.push(cb);
    if(!pending){
        timerFunc();
        pending = true
    }
}
```

##### 定义 $nextTick API
initState.js，初始化定义一个 $nextTick API，将回调函数放入 $nextTick 中
```
export function stateMixin(vm){
    vm.prototype.$nextTick = function(cb) {
        nextTick(cb)
    }
}
```

### $nextTick的使用场景

<div>
  <ul>
    <li>
        数据变化后立即获取 DOM 元素：例如，你可能需要在数据变化后立即获取 DOM 元素的尺寸或属性。
    </li>
    <li>
        父子组件通信：在某些情况下，子组件需要在父组件更新后执行某些操作。
    </li>
  </ul>
</div>