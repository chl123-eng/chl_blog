---
title: 旧版本Vuex如何做持久化
date: 2024-02-11
tags:
  - Vue
  - Vuex
cover: https://blog-1300014307.cos.ap-guangzhou.myqcloud.com/20240607111025.png
abstracts: Vuex如何实现本地持久化？
---

## 怎么实现vuex的持久化缓存


Vuex持久化缓存通常指的是将Vuex中的state状态保存到本地存储中，这样即使在页面刷新或关闭后，重新打开页面仍然可以恢复之前的状态。实现Vuex持久化缓存有几种不同的方法，以下是一些常见的实现方式：

## 使用localStorage或sessionStorage

我们可以手动将Vuex的state序列化后保存到localStorage或sessionStorage中，并在应用启动时从存储中恢复状态。

```js
// 保存state到localStorage
function saveToStorage(state) {
  localStorage.setItem('vuex-state', JSON.stringify(state));
}
​
// 从localStorage恢复state
function loadFromStorage() {
  const storedState = localStorage.getItem('vuex-state');
  return storedState ? JSON.parse(storedState) : {};
}
​
// 在Vuex的mutations中使用
const mutations = {
  saveState(state) {
    saveToStorage(state);
  }
};
​
// 在Vuex的actions中恢复state
const actions = {
  restoreState({ commit }) {
    const savedState = loadFromStorage();
    if (Object.keys(savedState).length) {
      commit('hydrateState', savedState);
    }
  }
};
​
// 在组件中或在应用启动时调用恢复state的action
dispatch('restoreState');

```

### 快速使用：vuex-persist插件

**npm install vuex-persist，然后，在我们的Vuex store中使用这个插件：**

```js

javascript复制代码import Vue from 'vue';
import Vuex from 'vuex';
import vuexPersist from 'vuex-persist';
​
Vue.use(Vuex);
​
const store = new Vuex.Store({
  // ...我们的state, mutations, actions等
});
​
const vuexPersist = new vuexPersist({
  key: 'vuex', // 存储的名称
  storage: window.localStorage, // 存储方式，可以选择sessionStorage或localStorage
  // 其他配置...
});
​
// 使用插件
vuexPersist.plugin(store);
```

**注意**

1. 序列化: 当我们将state保存到本地存储时，确保state中的数据可以被序列化。这意味着state中不应该包含函数、undefined等不能被JSON序列化的数据。
2. 安全性: 存储在localStorage或sessionStorage中的数据可以被同源的任何JavaScript代码访问，因此不要存储敏感信息。
性能: 避免频繁地读写本地存储，因为这可能会影响性能。

## vuex-persist 插件扩展

vuex-persist是一个第三方Vuex插件，它提供了一种简便的方式来持久化Vuex的状态。这个插件的实现基于几个关键步骤：

1. 序列化状态: 插件首先会将Vuex store的状态（state）序列化成JSON字符串。这是通过JSON.stringify()实现的，确保状态对象中所有的数据都是可以被序列化的。
2. 存储状态: 序列化后的状态字符串被保存到浏览器的localStorage或sessionStorage中。vuex-persist允许我们通过配置来选择使用哪一种存储方式。
监听状态变化: 插件会监听Vuex store的状态变化。每当状态发生变化时，插件都会自动将新的状态序列化并更新到本地存储中。这通常是通过订阅Vuex store的mutation事件来实现的。
3. 恢复状态: 当应用启动或者需要从持久化存储中恢复状态时，插件会从localStorage或sessionStorage中读取状态字符串，并使用JSON.parse()将其解析回原始的对象结构。
4. 自动恢复: vuex-persist通常在Vuex store初始化时自动执行状态恢复的逻辑。这意味着在我们的应用启动时，如果本地存储中有保存的状态，vuex-persist会自动恢复这些状态。
5. 配置选项: vuex-persist提供了多种配置选项，允许我们定制化持久化的行为。例如，我们可以配置要持久化的state片段（通过paths选项），或者设置一个函数来在保存之前过滤状态（通过reducer选项）。
6. 版本控制: 插件还可以处理版本控制问题，如果我们的应用升级后state结构发生了变化，vuex-persist可以通过配置（如reducer函数）来确保向后兼容。
7. 集成Vuex严格模式: 当Vuex store配置为严格模式时，vuex-persist能够确保所有状态的变更都是通过mutations进行的，即使这些变更是由插件自身触发的。
