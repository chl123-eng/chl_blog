---
title: Vue-2-6源码分析之watch原理
date: 2024-07-15 17:15:07
tags:
  - Vue
  - Vue源码
cover: https://blog-1300014307.cos.ap-guangzhou.myqcloud.com/vue-01.jpg
categories:
  - Vue
  - 总结
  - Vue源码
---

### Watcher的种类是什么

#### Watcher的种类

<div>
  <ul>
    <li>
      <span style="background-color: #fff5f5;color: #ff502c;">渲染Watcher</span> ：变量修改时，负责通知HTML里的重新渲染
    </li>
    <li>
       <span style="background-color: #fff5f5;
        color: #ff502c;">computed Watcher</span> ：变量修改时，负责通知computed里依赖此变量的computed属性变量的修改
    </li>
     <li>
       <span style="background-color: #fff5f5;
        color: #ff502c;">watch Watcher</span> ：变量修改时，负责通知watch属性里所对应的变量函数的执行
    </li>
  </ul>
</div>


### 实现数据响应式

<div style="background-color: #fff5f5;color:#666;padding: 10px 20px; line-height: 40px">
任何类型的Watcher都是基于数据响应式的，也就是说，要想实现Watcher，就需要先实现数据响应式，而数据响应式的原理就是通过Object.defineProperty去劫持变量的get和set属性。
<a href="Vue.2.6源码分析之响应式数据原理">请移步[Vue.2.6源码分析之响应式数据原理]</a>。
</div>

#### 什么是Dep？
```
// 例子代码，与本章代码无关

<div>{{name}}</div>

data() {
        return {
            name: '林三心'
        }
    },
    computed: {
        info () {
            return this.name
        }
    },
    watch: {
        name(newVal) {
            console.log(newVal)
        }
    }

```
这里name变量被三个地方所依赖，三个地方代表了三种Watcher，那么name会直接自己管这三个Watcher吗？答案是不会的，name会实例一个Dep，来帮自己管这几个Wacther，类似于管家，当name更改的时候，会通知dep，而dep则会带着主人的命令去通知这些Wacther去完成自己该做的事

![](https://chlblog.oss-cn-guangzhou.aliyuncs.com/watcher1.png)

#### Watcher为何也要反过来收集Dep？

上面说到了，dep是name的管家，他的职责是：name更新时，dep会带着主人的命令去通知subs里的Watcher去做该做的事，那么，dep收集Watcher很合理。那为什么watcher也需要反过来收集dep呢？这是因为computed属性里的变量没有自己的dep，也就是他没有自己的管家，看以下例子：

<div style="background-color: #fff5f5;color:#666;padding: 10px 20px; line-height: 40px">
这里先说一个知识点：如果html里不依赖name这个变量，那么无论name再怎么变，他都不会主动去刷新视图，因为html没引用他（说专业点就是：name的dep里没有渲染Watcher），注意，这里说的是不会主动，但这并不代表他不会被动去更新。什么情况下他会被动去更新呢？那就是computed有依赖他的属性变量。
</div>


```
// 例子代码，与本章代码无关

<div>{{person}}</div>

computed: {
    person {
        return `名称：${this.name}`
        }
    }

```
这里的person事依赖于name的，但是person是没有自己的dep的（因为他是computed属性变量），而name是有的。好了，继续看，请注意，此例子html里只有person的引用没有name的引用，所以name一改变，按理说虽然person跟着变了，但是html不会重新渲染，因为name虽然有dep，有更新视图的能力，但是奈何人家html不引用他啊！person想要自己去更新视图，但他却没这个能力啊，毕竟他没有dep这个管家！这个时候computed Watcher里收集的name的dep就派上用场了，可以借助这些dep去更新视图，达到更新html里的person的效果。具体会在下面computed里实现。


### Watcher的实现

```
import { nextTick } from "../utils/nextTick";
import { popTarget, pushTarget } from "./dep";

let id = 0
class watcher {
    constructor(vm, exprOrfn,cb,options){
        this.vm = vm;
        this.exprOrfn = exprOrfn
        this.cb = cb
        this.options = options
        this.id = id++
        this.user = !!options.user
        this.deps = [] //watcher存放dep
        this.depsId = new Set()

        if(typeof exprOrfn === "function"){
            this.getter = exprOrfn
        }else{//字符串变成函数
            this.getter = function(){//属性c.c.c
                let path = exprOrfn.split('.');
                let obj = vm;
                for(let i = 0; i < path.length; i++){
                    obj = obj[path[i]]
                }
                return obj;
            }
        }
        
        //第一次渲染页面
        this.value = this.get(); //获取watch旧值
    }
    addDep(dep){
        //1、去重
        let id = dep.id
        if(!this.depsId.has(id)){
            this.deps.push(dep)
            this.depsId.add(id)
            dep.addSub(this)
        }
    }
    //更新数据
    run(){
        let value = this.get() //watch新值
        this.oldValue  = this.value
        this.value = value
        if(this.user){
            this.cb.call(this.vm,value,this.oldValue);
        }
    }
    get(){
        pushTarget(this);
        const value = this.getter()//初次更新获取到原始的值
        popTarget();
        return value

        // queueWatcher(this)
    }
    //更新
    update(){
        //实现：不要数据更新一次就调用一次
        //方式：缓存
        // this.getter()
        queueWatcher(this)
    }
}

let queue = [];//将需要批量更新的watcher存放到一个队列中
let has = {};
let pending = false


function flushWatcher(){
    queue.forEach(item => {item.run(),item.cb()})
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

export default watcher

```

### 渲染Watcher：初始化渲染数据
```
  //lifecycle.js
  export function mountComponent(vm,el){
    //1、vm.render将render函数变成vnode
    //2、vm._update将vnode变成真实dom
    // vm._update(vm._render())

    let updateComponent = () => {
        vm._update(vm._render())
    }
    new watcher(vm,updateComponent, ()=>{
    }, true);
}
```

### watch Watcher: 更新数据
```
  //initState.js
  //watch Watcher关键：user: true
  vm.prototype.$watch = function(Vue,exprOfn,handler,options) {
      let watcher = new Watcher(Vue, exprOfn, handler, {...options,user:true});
      if(options.immediate){
          handler.call(vm);
      }
  }
  ```