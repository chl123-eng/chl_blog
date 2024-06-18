---
title: uni-app微信小程序实现axios请求拦截器和重复请求中断
date: 2022-12-08
tags:
  - uni-app
  - http
  - 微信小程序
cover: http://www.xiesmallxie.cn/20211208153631.jpeg
categories:
  - axios
  - 微信小程序
---

最近在写二手表微信小程序的时候，发现老是会有重复请求的情况，用了函数防抖和布尔拦截之后，又显得非常
臃肿，没能从根本上解决问题，刚好 leader 叫我做一下重复请求拦截。可是，当我翻开 flyio 文档之后......

## What the fuck

没想到 flyio 竟然没有预设重复请求功能，绝望

![](http://www.xiesmallxie.cn/20211208153631.jpeg)

## 当前目标和处理思路

那么我的目标就变成了兼容旧 flyio 的拦截器功能，并且还要实现重复请求拦截功能。对接重复请求拦截，因为我们当前小
程序只有微信端，那我就直接换成了微信小程序官方的请求，刚好也有请求中断功能。对于拦截器，基本都是通过 promise 来实现的
，那这里就直接自己写一个。

## 请求和响应拦截的思路

通过 Promise.then 实现链式的串行调用，因为前置有请求拦截器，后置有响应拦截器，中间插入请求结构体。整体结构如下图

![](http://www.xiesmallxie.cn/20211208153630.png?imageMogr2/thumbnail/!50p)

## 代码细究

整体工具库使用 lodash

```js
export const callFor = (arrayData, fn) => {
  arrayData.forEach(item => {
    fn.call(null, item)
  })
}
// 全局拦截器实例
export default class Interceptor {
  constructor() {
    this.handlers = []
  }
  // use方法传入Promise两个状态处理语句
  use(fulfilled, rejected) {
    this.handlers.push({
      fulfilled,
      rejected
    })
    return this.handlers.length - 1
  }
}

// 主体请求参数 作为promise调用链的入参之一
const requestFn = config => {
  const options = config
  // requestTaskKey 当前重复请求的url标识
  let lastRequestKey = config.hasOwnProperty('requestTaskKey') ? config.requestTaskKey : ''
  // 如果全局变量数组里面含有当前的请求，那么就直接中断上一次请求
  if (
    lastRequestKey &&
    getApp().hasOwnProperty('requestTasks') &&
    getApp().requestTasks.hasOwnProperty(lastRequestKey)
  ) {
    try {
      console.log('中断了上一次的请求----------------------')
      getApp().requestTasks[lastRequestKey].abort()
      // 中断后就应该清除掉
      delete getApp().requestTasks[lastRequestKey]
    } catch (e) {
      console.error(e)
    }
  }
  // 返回一个promise
  return new Promise((resolve, reject) => {
    const url = options.baseURL + options.url
    options.url = url
    const requestTask = uni.request({
      ...options,
      complete: res => {
        console.log(res)
        // 每一次成功之后 就清空当前url在全局变量数组里面的位置
        // request:faile abort为公司自定义的错误请求状态，
        if (res.errMsg !== 'request:faile abort' && lastRequestKey) {
          delete getApp().requestTasks[lastRequestKey]
        }
        if (res.errMsg === 'request:ok') {
          return resolve({ ...res, request: options })
        } else if (res.errMsg === 'request:fail') {
          return reject({ ...res, request: options })
        }
      }
    })
    // 如果当前请求需要支持重复请求的中断，
    if (lastRequestKey) {
      // 没有队列的情况下 全局变量存储队列
      if (!getApp().requestTasks) {
        getApp().requestTasks = {}
      }
      // 把当前请求塞入存储队列之中
      getApp().requestTasks[lastRequestKey] = requestTask
    }
  })
}

function HttpRequest(config) {
  // 深拷贝配置防止配置被上一个请求修改
  this.config = _.cloneDeep(
    Object.assign(
      {},
      {
        baseURL: '',
        url: '',
        data: {},
        header: {},
        method: 'GET',
        timeout: 60000
      },
      config
    )
  )

  // 初始化拦截器
  this.interceptors = {
    request: new Interceptor(),
    response: new Interceptor()
  }
}

HttpRequest.prototype.request = function (config = {}) {
  const options = Object.assign({}, this.config, config)
  console.log(options)

  let requestInterceptorChain = [],
    responseInterceptorChain = []
  // 推入请求拦截器
  this.interceptors.request.forEach(obj => {
    requestInterceptorChain.unshift(obj.fulfilled, obj.rejected)
  })
  // 推入响应拦截器
  this.interceptors.response.forEach(obj => {
    responseInterceptorChain.push(obj.fulfilled, obj.rejected)
  })

  let promise
  // 第二个值为undefined是因为要为后面的请求进行补位 这样可以防止流入错误请求
  let chain = [requestFn, undefined]
  // request请求拦截插入最前方
  Array.prototype.unshift.call(chain, ...requestInterceptorChain)
  // responese响应插入最后方
  chain = chain.concat(responseInterceptorChain)
  promise = Promise.resolve(options)
  // promise.then串行处理
  while (chain.length) {
    promise = promise.then(chain.shift(), chain.shift())
  }
  return promise
}

// 不同请求格式的差异化处理
callFor(['post', 'put', 'patch'], function (methodType) {
  HttpRequest.prototype[methodType] = function (url, data = {}, otherConfig = {}) {
    const config = Object.assign({}, { data, url, method: methodType }, otherConfig)
    return this.request(config)
  }
})

callFor(['delete', 'get', 'head', 'config'], function (methodType) {
  HttpRequest.prototype[methodType] = function (url, params = {}, otherConfig = {}) {
    const config = Object.assign({}, otherConfig, { url, params, method: methodType })
    return this.request(config)
  }
})

export default HttpRequest
```

这就是我基于 Prmiose 加小程序重复请求封装的核心代码，这里还未涉及到离开页面时的请求中断行为。不过大概思路的话就是收集所有请求的状态，封装成对象，调用时修改状态，后续有时间我会更加深究这部分的代码，更好地服务于业务。
