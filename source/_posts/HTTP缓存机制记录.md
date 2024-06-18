---
title: HTTP缓存机制记录
date: 2019-12-04
tags:
  - http
cover: https://blog-1300014307.cos.ap-guangzhou.myqcloud.com/http01.png
categories:
  - http
  - 总结
---

http 缓存机制很容易就忘记了 这里大概记录一下

## 简单概括

http 缓存分为强缓存(Cache-Control > Expires),协商缓存(Etag / If-None-Match > Last-Modified / If-Modified-Since)强缓存和协商缓存的区别> 强缓存命中的情况下不和服务器做交互，协商缓存不管有没有命中都会和服务器做交互

## 第一次请求

1. 浏览器发起 http 请求到服务器。
2. http 请求通过浏览器缓存，浏览器缓存没有发现该请求的缓存标识和缓存结果，会直接放行，让 http 请求直接请求到服务器。
3. 服务器返回请求结果和缓存规则给浏览器
4. 浏览器将请求结果和缓存标识存入浏览器缓存中，作为下一次请求的标识

## 第二次请求命中强缓存的情况

- Expires 该字段作为缓存标志， 如果客户端的时间小于 Expires 的值时，就直接使用缓存结果，但是这只是存在于 HTTP/1.0 中，现今大部分已经被 Cache-Control 所替代
- Cache-Control HTTP1.1 所使用的标识，该字段有以下几种属性
- public：所有内容都将被缓存（客户端和代理服务器都可缓存）
- private：所有内容只有客户端可以缓存，Cache-Control 的默认取值
- no-cache：客户端缓存内容，但是是否使用缓存则需要经过协商缓存来验证决定
- no-store：所有内容都不会被缓存，即不使用强制缓存，也不使用协商缓存
- max-age=xxx (xxx is numeric)：缓存内容将在 xxx 秒后失效

## 第二次请求

- 强缓存未命中，协商缓存命中的情况当协商缓存命中时，服务器返回 304 状态码表示该资源没有更新协商缓存字段有 Last-Modified / If-Modified-Since 和 Etag / If-None-Match，其中 Etag / If-None-Match 的优先级比 Last-Modified / If-Modified-Since 高。
- If-None-Match/Etag => 客户端再次发起该请求时，携带上次请求返回的唯一标识 Etag 值，通过此字段值告诉服务器该资源上次请求返回的唯一标识值。服务器收到该请求后，发现该请求头中含有 If-None-Match，则会根据 If-None-Match 的字段值与该资源在服务器的 Etag 值做对比，一致则返回 304，代表资源无更新，继续使用缓存文件；不一致则重新返回资源文件，返回 200
- Last-Modified/If-Modified-Since => 则是客户端再次发起该请求时，携带上次请求返回的 Last-Modified 值，通过此字段值告诉服务器该资源上次请求返回的最后被修改时间。服务器收到该请求，发现请求头含有 If-Modified-Since 字段，则会根据 If-Modified-Since 的字段值与该资源在服务器的最后被修改时间做对比，若服务器的资源最后被修改时间大于 If-Modified-Since 的字段值，则重新返回资源，状态码为 200；否则则返回 304，代表资源无更新，可继续使用缓存文件
