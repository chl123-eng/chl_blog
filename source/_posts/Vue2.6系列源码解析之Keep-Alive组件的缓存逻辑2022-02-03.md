---
title: Vue2.6系列源码解析之Keep-Alive组件的缓存逻辑
date: 2022-02-03
tags:
  - Vue
cover: https://blog-1300014307.cos.ap-guangzhou.myqcloud.com/vue-01.jpg
categories:
  - Vue
  - 总结
  - Vue源码
---

对于很多用过 vue 这个框架的人来说，想必都用过 keep-alive 组件缓存功能。vue 内部使用了 LRU 缓存淘汰算法来实现组件的缓存更新问题，那 vue 是如何实现这个 keep-alive 组件的逻辑呢，我们来解析一下。

## 源码解读

```js
// 获取vnode实例名称
function getComponentName(opts: ?VNodeComponentOptions): ?string {
  return opts && (opts.Ctor.options.name || opts.tag);
}
// 匹配当前队列里面是否存在该vnode
function matches(
  pattern: string | RegExp | Array<string>,
  name: string
): boolean {
  if (Array.isArray(pattern)) {
    return pattern.indexOf(name) > -1;
  } else if (typeof pattern === "string") {
    return pattern.split(",").indexOf(name) > -1;
  } else if (isRegExp(pattern)) {
    return pattern.test(name);
  }
  return false;
}
// 实例筛选
function pruneCache(keepAliveInstance: any, filter: Function) {
  const { cache, keys, _vnode } = keepAliveInstance;
  for (const key in cache) {
    const entry: ?CacheEntry = cache[key];
    if (entry) {
      const name: ?string = entry.name;
      if (name && !filter(name)) {
        pruneCacheEntry(cache, key, keys, _vnode);
      }
    }
  }
}

function pruneCacheEntry(
  cache: CacheEntryMap,
  key: string,
  keys: Array<string>,
  current?: VNode
) {
  const entry: ?CacheEntry = cache[key];
  if (entry && (!current || entry.tag !== current.tag)) {
    entry.componentInstance.$destroy();
  }
  cache[key] = null;
  remove(keys, key);
}

export default {
  name: "keep-alive",
  abstract: true,

  props: {
    include: patternTypes,
    exclude: patternTypes,
    max: [String, Number],
  },

  methods: {
    // 虚拟dom缓存
    cacheVNode() {
      const { cache, keys, vnodeToCache, keyToCache } = this;
      // 如果存在等待被缓存的vnode 就要缓存起来
      if (vnodeToCache) {
        const { tag, componentInstance, componentOptions } = vnodeToCache;
        cache[keyToCache] = {
          name: getComponentName(componentOptions),
          tag,
          componentInstance,
        };
        // 推vnode的key进入缓存组
        keys.push(keyToCache);
        // 如果缓存组件有最大数量限制的情况下 并且超大最大缓存数量限制，那么就删除缓存队列的第一项
        // pruneCacheEntry 判断了当前的缓存队列的第一项是 如果跟新进来的最新vnode是否一致，
        // 不一致的情况下就直接卸载当前第一项的实例，一致就保存不进行卸载进行复用操作
        if (this.max && keys.length > parseInt(this.max)) {
          pruneCacheEntry(cache, keys[0], keys, this._vnode);
        }
        this.vnodeToCache = null;
      }
    },
  },

  created() {
    // 存储所有组件
    this.cache = Object.create(null);
    // 存储所有组件的cid值 源码里面是逐步递增的
    this.keys = [];
  },

  destroyed() {
    // 卸载时删除所有缓存的组件实例
    for (const key in this.cache) {
      pruneCacheEntry(this.cache, key, this.keys);
    }
  },

  mounted() {
    // 开始缓存当前实例
    this.cacheVNode();
    // 监听include和exclude队列，去除里面不匹配的的组件
    this.$watch("include", (val) => {
      pruneCache(this, (name) => matches(val, name));
    });
    this.$watch("exclude", (val) => {
      pruneCache(this, (name) => !matches(val, name));
    });
  },

  updated() {
    // 更新的时候同步更新当前vnode
    this.cacheVNode();
  },

  render() {
    const slot = this.$slots.default;
    const vnode: VNode = getFirstComponentChild(slot);
    const componentOptions: ?VNodeComponentOptions =
      vnode && vnode.componentOptions;
    // 如果存在组件实例
    if (componentOptions) {
      // check pattern
      const name: ?string = getComponentName(componentOptions);
      const { include, exclude } = this;
      // 如果不缓存的列表有当前vnode或者缓存列表没有当前vnode 那么就直接返回该节点
      if (
        (include && (!name || !matches(include, name))) ||
        (exclude && name && matches(exclude, name))
      ) {
        return vnode;
      }
      // 到这里就说明当前组件是需要缓存的
      const { cache, keys } = this;
      const key: ?string =
        vnode.key == null
          ? componentOptions.Ctor.cid +
            (componentOptions.tag ? `::${componentOptions.tag}` : "")
          : vnode.key;
      // 如果当前组件被缓存过 那么就更新当前组件 把组件推到缓存队列key的最后边，
      // 这样就能总是获取到最新的更新项 LRU算法
      if (cache[key]) {
        // 复用缓存
        vnode.componentInstance = cache[key].componentInstance;
        remove(keys, key);
        keys.push(key);
      } else {
        this.vnodeToCache = vnode;
        this.keyToCache = key;
      }

      vnode.data.keepAlive = true;
    }
    return vnode || (slot && slot[0]);
  },
};
```

## 说说 LRU 缓存淘汰算法

LRU 算法，即最近最久未使用，是一种非常常见的缓存淘汰算法。
算法的设计原则： 如果一个数据在最近一段时间没有被访问到，那么在将来它被访问的可能性也很小。也就是说，当限定的空间已存满数据时，应当把最久没有被访问到的数据淘汰。

下面是我自己画的 keep-alive 缓存淘汰算法的流程示意图例，辅助查看

![](http://www.xiesmallxie.cn/20220209112855.png)
