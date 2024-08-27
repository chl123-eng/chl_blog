---
title: canvas海报生成
date: 2024-08-27 17:44:24
tags:
categories:
  - 总结
cover: https://chlblog.oss-cn-guangzhou.aliyuncs.com/switchTab1.jpg
---

### 前言

生成商品海报进行分享或保存

#### 安装html2canvas，引入
```
npm install html2canvas

import html2canvas from 'html2canvas';
```

#### 生成海报方法

```
generateImage(e, ownerFun) {
    setTimeout(() => {
        const dom = document.getElementById('share2') // 需要生成图片内容的 dom 节点
        html2canvas(dom, {
            width: dom.clientWidth, //dom 原始宽度
            height: dom.clientHeight,
            scrollY: 0, // html2canvas默认绘制视图内的页面，需要把scrollY，scrollX设置为0
            scrollX: 0,
            useCORS: true, //支持跨域
            scale: 2, // 设置生成图片的像素比例，默认是1，如果生成的图片模糊的话可以开启该配置项
        }).then((canvas) => {
            console.log(canvas.toDataURL('image/png'));
            // 生成成功
            // html2canvas 生成成功的图片链接需要转成 base64位的url
            ownerFun.callMethod('receiveRenderData', canvas.toDataURL('image/png'))
        }).catch(err => {
            ownerFun.callMethod('_errAlert', `【生成图片失败，请重试】${err}`)
        })
    }, 300)
},
```