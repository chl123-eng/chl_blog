---
title: Vue-2-6源码分析之diff算法
date: 2024-07-24 15:45:20
tags:
  - Vue源码
cover: https://blog-1300014307.cos.ap-guangzhou.myqcloud.com/vue-01.jpg
categories:
  - Vue源码
---

### 虚拟DOM算法

 <span style="background-color: #fff5f5;color: #ff502c;">虚拟DOM算法 = 虚拟DOM + Diff算法</span>

  <span style="color: #036aca;">虚拟DOM算法操作真实DOM，性能高于直接操作真实DOM</span>

### 虚拟DOM

```
<ul id="list">
    <li class="item">哈哈</li>
    <li class="item">呵呵</li>
    <li class="item">林三心哈哈哈哈哈</li> // 修改
</ul>

```
生成的新虚拟DOM为：
```
let newVDOM = { // 新虚拟DOM
    tagName: 'ul', // 标签名
    props: { // 标签属性
        id: 'list'
    },
    children: [ // 标签子节点
        {
            tagName: 'li', props: { class: 'item' }, children: ['哈哈']
        },
        {
            tagName: 'li', props: { class: 'item' }, children: ['呵呵']
        },
        {
            tagName: 'li', props: { class: 'item' }, children: ['林三心哈哈哈哈哈']
        },
    ]
}
```

### Diff算法

<span style="color: #036aca;">概念：</span>对比两者是旧虚拟DOM和新虚拟DOM，对比出是哪个虚拟节点更改了，找出这个虚拟节点，并只更新这个虚拟节点所对应的真实节点，而不用更新其他数据没发生改变的节点，实现精准地更新真实DOM，进而提高效率。

#### Diff同层对比

新旧虚拟DOM对比的时候，Diff算法比较只会在同层级进行, 不会跨层级比较。 所以Diff算法是:<span style="color: #ff502c;">深度优先算法</span>。
<div>
<img src="https://chlblog.oss-cn-guangzhou.aliyuncs.com/diff1.png" />
</div>

#### 对比流程

<div>
<img src="https://chlblog.oss-cn-guangzhou.aliyuncs.com/diff2.png" />
</div>

<span style="color: #036aca;">patch的核心原理代码</span>
```
function patch(oldVnode, newVnode) {
  // 比较是否为一个类型的节点
  if (sameVnode(oldVnode, newVnode)) {
    // 是：继续进行深层比较
    patchVnode(oldVnode, newVnode)
  } else {
    // 否
    const oldEl = oldVnode.el // 旧虚拟节点的真实DOM节点
    const parentEle = api.parentNode(oldEl) // 获取父节点
    createEle(newVnode) // 创建新虚拟节点对应的真实DOM节点
    if (parentEle !== null) {
      api.insertBefore(parentEle, vnode.el, api.nextSibling(oEl)) // 将新元素添加进父元素
      api.removeChild(parentEle, oldVnode.el)  // 移除以前的旧元素节点
      // 设置null，释放内存
      oldVnode = null
    }
  }

  return newVnode
}


```

```
function sameVnode(oldVnode, newVnode) {
  return (
    oldVnode.key === newVnode.key && // key值是否一样
    oldVnode.tagName === newVnode.tagName && // 标签名是否一样
    oldVnode.isComment === newVnode.isComment && // 是否都为注释节点
    isDef(oldVnode.data) === isDef(newVnode.data) && // 是否都定义了data
    sameInputType(oldVnode, newVnode) // 当标签为input时，type必须是否相同
  )
}
```

<span style="color: #036aca;">patchVnode方法</span>
<div>
<li>
    找到对应的真实DOM，称为el
</li>
<li>
    判断newVnode和oldVnode是否指向同一个对象，如果是，那么直接return
</li>
<li>
    如果他们都有文本节点并且不相等，那么将el的文本节点设置为newVnode的文本节点。
</li>
<li>
    如果oldVnode有子节点而newVnode没有，则删除el的子节点
</li>
<li>
    如果oldVnode没有子节点而newVnode有，则将newVnode的子节点真实化之后添加到el
</li>
<li>
    如果两者都有子节点，则执行updateChildren函数比较子节点
</li>
</div>

```
function patchVnode(oldVnode, newVnode) {
  const el = newVnode.el = oldVnode.el // 获取真实DOM对象
  // 获取新旧虚拟节点的子节点数组
  const oldCh = oldVnode.children, newCh = newVnode.children
  // 如果新旧虚拟节点是同一个对象，则终止
  if (oldVnode === newVnode) return
  // 如果新旧虚拟节点是文本节点，且文本不一样
  if (oldVnode.text !== null && newVnode.text !== null && oldVnode.text !== newVnode.text) {
    // 则直接将真实DOM中文本更新为新虚拟节点的文本
    api.setTextContent(el, newVnode.text)
  } else {
    // 否则

    if (oldCh && newCh && oldCh !== newCh) {
      // 新旧虚拟节点都有子节点，且子节点不一样

      // 对比子节点，并更新
      updateChildren(el, oldCh, newCh)
    } else if (newCh) {
      // 新虚拟节点有子节点，旧虚拟节点没有

      // 创建新虚拟节点的子节点，并更新到真实DOM上去
      createEle(newVnode)
    } else if (oldCh) {
      // 旧虚拟节点有子节点，新虚拟节点没有

      //直接删除真实DOM里对应的子节点
      api.removeChild(el)
    }
  }
}

```

<span style="color: #036aca;">updateChildren方法</span>

<div>
<li>
    oldS 和 newS 使用sameVnode方法进行比较，sameVnode(oldS, newS)
</li>
<li>
    oldS 和 newE 使用sameVnode方法进行比较，sameVnode(oldS, newE)
</li>
<li>
   oldE 和 newS 使用sameVnode方法进行比较，sameVnode(oldE, newS)
</li>
<li>
    oldE 和 newE 使用sameVnode方法进行比较，sameVnode(oldE, newE)
</li>
<li>
    如果以上逻辑都匹配不到，再把所有旧子节点的 key 做一个映射到旧节点下标的 key -> index 表，然后用新 vnode 的 key 去找出在旧节点中可以复用的位置。
</li>
</div>

<div>
<img src="https://chlblog.oss-cn-guangzhou.aliyuncs.com/diff3.png" />
</div>

```
oldS = a, oldE = c
newS = b, newE = a
```
oldS 和 newE 相等，需要把节点a移动到newE所对应的位置，也就是末尾，同时oldS++，newE--
<div>
<img src="https://chlblog.oss-cn-guangzhou.aliyuncs.com/diff4.png" />
</div>

```
oldS = b, oldE = c
newS = b, newE = e
```
oldS 和 newS相等，需要把节点b移动到newS所对应的位置，同时oldS++,newS++
<div>
<img src="https://chlblog.oss-cn-guangzhou.aliyuncs.com/diff5.png" />
</div>

```
oldS = c, oldE = c
newS = c, newE = e
```
oldS、oldE 和 newS相等，需要把节点c移动到newS所对应的位置，同时oldS++,newS++
<div>
<img src="https://chlblog.oss-cn-guangzhou.aliyuncs.com/diff6.png" />
</div>

oldS > oldE，则oldCh先遍历完成了，而newCh还没遍历完，说明newCh比oldCh多，所以需要将多出来的节点，插入到真实DOM上对应的位置上
<div>
<img src="https://chlblog.oss-cn-guangzhou.aliyuncs.com/diff7.png" />
</div>

<span style="color: #036aca;">updateChildren函数</span>
```
function updateChildren方法(oldChildren,newChildren,parent){
    //创建双指针
    let oldStartIndex = 0
    let oldStartVnode = oldChildren[oldStartIndex]
    let oldEndIndex = oldChildren.length - 1
    let oldEndVnode = oldChildren[oldEndIndex]

    let newStartIndex = 0
    let newStartVnode = newChildren[newStartIndex]
    let newEndIndex = newChildren.length - 1
    let newEndVnode = newChildren[newEndIndex]


    //创建旧元素的映射表
    function makeIndexBykey(child){
        let map = {}
        child.forEach((item,index) => {
            if(item.key){
                map[item.key] = index;
            }
        })
        return map
    }
    let map = makeIndexBykey(oldChildren)
    //判断是否是同一元素
    function isSameVnode(oldContext,newContext){
        return (oldContext.tag === newContext.tag && oldContext.key == newContext.key)
    }
    while(oldStartIndex <= oldEndIndex && newStartIndex <= newEndIndex){
        //判断头部是否是同一元素 1-4是交叉比对
        if(isSameVnode(oldStartVnode,newStartVnode)){//从头部开始比
            //递归
            patch(oldStartVnode,newStartVnode)
            //移动指针
            oldStartVnode = oldChildren[++oldStartIndex]
            newStartVnode = newChildren[++newStartIndex]
        }else if(isSameVnode(oldEndVnode,newEndVnode)){//从尾部开始比
            patch(oldEndVnode,newEndVnode)
            //移动指针
            oldEndVnode = oldChildren[--oldEndIndex]
            newEndVnode = newChildren[--newEndIndex]
        }else if(isSameVnode(oldStartVnode,newEndVnode)){//旧前新后比
            patch(oldStartVnode,newEndVnode)
            //移动指针
            oldStartVnode = oldChildren[++oldStartIndex]
            newEndVnode = newChildren[--newEndIndex]
        }
        else if(isSameVnode(oldEndVnode,newStartVnode)){//旧后新前比
            patch(oldEndVnode,newStartVnode)
            //移动指针
            oldEndVnode = oldChildren[--oldEndIndex]
            newStartVnode = newChildren[++newStartIndex]
        }else{//儿子之间没有任何关系 暴力比对
            //1、创建旧元素映射表
            //2、从老中寻找新的元素 map:{c:0,b:1,a:2}
            let moveIndex = map[newStartVnode.key]
            if(moveIndex == undefined){
                parent.insertBefore(createEl(newStartVnode),oldStartVnode.el)
            }else{
                let moveVnode = oldChildren[moveIndex] // 旧元素中匹配到的元素
                oldChildren[moveIndex] = null //防止数组塌陷
                //插入
                parent.insertBefore(moveVnode.el,oldStartVnode.el)
                //有儿子的情况 递归
                patch(moveVnode,newStartVnode)
            }
            //新的元素指针位移
            newStartVnode = newChildren[++newStartIndex]
        } 
    }
    //添加多余的儿子
    if(newStartIndex <= newEndIndex){
        for(let i = newStartIndex; i < newEndIndex;i++){
            parent.appendChild(createEl(newChildren[i]))
        }
    }
    //将老的多余的元素去掉
    if(oldStartIndex <= oldEndIndex){
        for(let i = oldStartIndex; i <= oldEndIndex; i++){
            let child = oldChildren[i]
            if(child != null){
                parent.removeChild(child.el)//删除元素
            }
        }
    }
}
```