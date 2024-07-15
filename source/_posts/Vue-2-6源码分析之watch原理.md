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


#### 实现数据响应式

<div style="background-color: #fff5f5;color:#666;padding: 10px 20px; line-height: 40px">
任何类型的Watcher都是基于数据响应式的，也就是说，要想实现Watcher，就需要先实现数据响应式，而数据响应式的原理就是通过Object.defineProperty去劫持变量的get和set属性。
<a href="Vue.2.6源码分析之响应式数据原理">请移步[Vue.2.6源码分析之响应式数据原理]</a>。
</div>


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