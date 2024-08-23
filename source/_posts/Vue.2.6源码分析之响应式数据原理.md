---
title: Vue.2.6源码分析之响应式数据原理
date: 2024-06-50
tags:
  - Vue源码
cover: https://blog-1300014307.cos.ap-guangzhou.myqcloud.com/vue-01.jpg
categories:
  - Vue源码
---

### 前言

响应式 是 Vue 最独特的特性之一。当修改实例的 data 的属性时，视图会进行更新。vue2 的数据劫持是利用 Object.defineProperty 的 getter 和 setter 来监听到属性的变化。

### 实现思路

1、定义 observer 函数
2、判断监听的值,如果是对象，则创建一个 Observer 类

```
 if(typeof data != 'object' || data == null){
        return data;
    }

    return new Observer(data)
```

3、判断是对象还是数组

```
if(Array.isArray(value))
```

### 对象的数据劫持

1、Observer 定义一个方法 walk,遍历对象的每个属性

```
    walk(data){
        let keys = Object.keys(data);
        for(let i = 0; i < keys.length; i++){
            let key = keys[i];
            let value = data[key];
            defineReactive(data,key,value);
        }
    }
```

2、定义 defineReactive，Object.defineProperty 对遍历的每个属性进行劫持

tip:
1、初始化的数据中对象的属性依然是一个对象时，要进行深度劫持，即递归；
2、对某个对象的属性赋值为一个对象时，在 set 中，对新赋值的对象进行劫持

```
function defineReactive(data,key,value){
    observer(value);//递归，劫持对象的某个对象属性
    Object.defineProperty(data,key,{
        get(){
            return value
        },
        set(newValue){
            if(newValue == value) return;
            observer(newValue);//如果赋值是一个对象，则要对对象进行遍历劫持
            value = newValue;
        }
    })
}
```

### 数组的数据劫持

如果属性是一个数组，则通过重写 Array 的方法进行函数劫持

```
if(Array.isArray(value)){
    value.__proto__ = ArrayMethods;

    //如果是数组对象
    this.observeArray(value);
}


observeArray(value){
    value.forEach(itemValue => {
        observer(itemValue)
    })
}
```

arr.js

```
//监听到data的属性是数组的时候重写Array的原型方法，即劫持函数

//获取原来的数组方法
let oldArrayProtoMethods = Array.prototype;

//继承: 现有的对象(oldArrayProtoMethods)来作为新创建对象(ArrayMethods)的原型
export let ArrayMethods = Object.create(oldArrayProtoMethods);

//劫持
let methods = [
    'push',
    'pop',
    'unshift',
    'splice'
]

methods.forEach(item => {
    ArrayMethods[item] = function(...args){
        console.log('劫持数组');
        let result = oldArrayProtoMethods[item].apply(this,args)

        //数组追加对象的情况

        let inserted
        switch(item){
            case 'push':
            case 'unshift':
                inserted = args
                break;
            case 'splice':
                inserted = args.splice(2);
                break;
        }
        //this指向数组中新增的对象{e:1},故有_ob_属性，指向Observer
        let ob = this._ob_;

        if(inserted){
            ob.observeArray(inserted);
        }
        return result
    }
})

```

tips:
1、对于一个对象数组，eg:[{a:1}], 遍历数组后，调用 observer 对对象进行劫持
2、数组中添加一个对象
