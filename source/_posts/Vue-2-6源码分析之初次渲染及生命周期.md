---
title: Vue-2-6源码分析之初次渲染及生命周期
date: 2024-06-27 11:50:12
tags:
  - Vue
  - Vue源码
cover: https://blog-1300014307.cos.ap-guangzhou.myqcloud.com/vue-01.jpg
categories:
  - Vue
  - 总结
  - Vue源码
---

<div style="display: flex;justify-content: center;">
  <img src='https://chlblog.oss-cn-guangzhou.aliyuncs.com/shengmingzhouqi1.png' style="width: 100%" />
</div>

#### 渲染流程：

<div>
  <li>
    依次判断是否有render、template；
  </li>
  <li>
    在没有template且存在el,就直接拿el.outerHTML；
  </li>
   <li>
    有template且存在el：
    <ul>
      <li>生成ast语法树</li>
      <li>生成render函数</li>
    </ul>
  </li>
   <li>
    生成虚拟节点
  </li>
  <li>
    真实节点挂载到页面上
  </li>
</div>

#### 生命周期

<a href="https://blog.csdn.net/zhenghuishengq/article/details/128581110?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-4-128581110-blog-128253661.235^v43^pc_blog_bottom_relevance_base3&spm=1001.2101.3001.4242.3&utm_relevant_index=5">生命周期详解</a>

<div>
  <div style="font-weight: 700"> beforeCreate：</div>
  <ul>
    <li>
      在创建完这个Vue实例之后，会进入一个初始化阶段，初始化生命周期和事件，但数据代理还并没有开始，因此这个阶段是拿不到vm对象以及_data数据的。
    </li>
  </ul>

  <div style="font-weight: 700"> created：</div>
  <ul>
    <li>
      进行了数据监测和数据代理的，获取到代理对象。
    </li>
  </ul>
  

  <div style="font-weight: 700"> beforeMount：</div>
  <ul>
    <li>
      Vue已经解析完成的真实DOM，但是此时的vue还没来的及将这些DOM向页面中存放。
    </li>
  </ul>
 

  <div style="font-weight: 700"> mounted：</div>
  <ul>
    <li>
      将内存中的虚拟DOM转化为真实DOM存在页面中,在这个函数中对DOM的操作均有效;
    </li>
    <li>
      一般会进行这些操作：开启定时器，发送网络请求，订阅消息，绑定自定义事件等操作
    </li>
  </ul>


  <div style="font-weight: 700"> beforeUpdate</div>
  <ul>
    <li>
      只要data中的数据发生改变，那么就会触发这个钩子函数;
    </li>
    <li>
      页面上渲染的数据和被改变的数据没有保持同步，就是页面还没有来得及展示被修改的值。
    </li>
  </ul>
  

  <div style="font-weight: 700"> updated</div>
  <ul>
    <li>
      在这个阶段，就会生成新的虚拟DOM，随后通过diff算法，实现新旧的虚拟DOM的比较，最终完成页面的更新。
    </li>
  </ul>


  <div style="font-weight: 700"> beforeDestory</div>
  <ul>
    <li>
      销毁的实例中的所有的data，methods还有指令等等都是处于可用状态，但是不能对实例中的具体的数据和方法进行一个操作;
    </li>
    <li>
      清除定时器、解绑自定义事件、取消消息订阅等。
    </li>
  </ul>
 

  <div style="font-weight: 700"> destoryed</div>
  <ul>
    <li>
      vue实例算是彻底的被销毁了，但是页面上的数据是还在的，但是不能操作上面的数据了。
    </li>
  </ul>
</div>


