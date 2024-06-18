---
title: 记一次uni-app原生应用高德地图配置的坑
date: 2023-11-16
tags:
  - 原生小程序
  - 高德地图
cover: https://blog-1300014307.cos.ap-guangzhou.myqcloud.com/gddt.png
categories:
  - 经验
feature: true
---

### 前言

为配合打荷生鲜 2.0 小程序的发展，需要对配送人员增加一个类似于美团骑手版的安卓 App，为此需要引入高德地图的 sdk，折腾了我一天的时候，为此将遇到的问题记录下来

### 新开项目如何引用 uni-app 原生插件？

1. 创建新项目，记录下 appId
   ![](https://blog-1300014307.cos.ap-guangzhou.myqcloud.com/1.png)
2. 将新项目与第三方插件绑定
   ![](https://blog-1300014307.cos.ap-guangzhou.myqcloud.com/2.png)
3. 项目配置再次进行绑定
   ![](https://blog-1300014307.cos.ap-guangzhou.myqcloud.com/3.png)

至此，插件的初步使用已经完成

### 如何结合高德地图 sdk？

1. 登录 DCloud 开发者中心，我的应用，应用信息->各平台信息，记下**包名/appid**，应用信息->Android 云端证书->证书详情，记下**证书的别名， SHA1 码，密码，并把证书下载到本地**
   ![](https://blog-1300014307.cos.ap-guangzhou.myqcloud.com/80.png)
   ![](https://blog-1300014307.cos.ap-guangzhou.myqcloud.com/5.png)
2. 前往高德地图控制台->应用管理->我的应用->创建应用，配置应用的 packageName 和 SHA1 码一定要和 Dcloud 的证书一致，不一致的话，基座打包后调用 api 会报 ERROR_CODE:7 错误
   ![](https://blog-1300014307.cos.ap-guangzhou.myqcloud.com/01.png)
   ![](https://blog-1300014307.cos.ap-guangzhou.myqcloud.com/02.png)
3. 创建应用配置完，回到列表拿到应用的 key，回填到 uni-app 项目配置中
   ![](https://blog-1300014307.cos.ap-guangzhou.myqcloud.com/6.png)
4. 基座打包，填写包名，证书别名，密码，上传证书，开始进行打包
   ![](https://blog-1300014307.cos.ap-guangzhou.myqcloud.com/81.png)
5. 真机调试，打开 Hbuilder/Hbuilderx->运行->运行到手机或模拟器->运行到 Android App 基座-> 使用自定义基座运行
   ![](https://blog-1300014307.cos.ap-guangzhou.myqcloud.com/82.png)

### 总结

主要的坑点还是在 packageName 包名那块，官方也没说明怎么生成 SHA1 码，于是就自己用 keytool 自己生成了，结果显而易见，一直卡在那边，通过慢慢结合网上摸索才得出解决办法。
