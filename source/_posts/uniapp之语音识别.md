---
title: uniapp之语音识别
date: 2024-05-23 11:42:24
tags:
  - uniapp
  - 语音识别
cover:
categories:
  - 经验
feature: true
---

### 前言

<p>在hbuilder中引入百度语音识别集成平台，实现&nbsp;&nbsp<span style="background:#e1c953; color:#000;padding:5px 5px">AI语音识别</span>&nbsp;&nbsp;</p>

<div style="display: flex;justify-content: center;">
    <img src='https://chlblog.oss-cn-guangzhou.aliyuncs.com/yuyinshibie1.jpg?Expires=1719138259&OSSAccessKeyId=TMP.3KkWxBhPT2PLwHUkAck2fqXke4mp3QgWaU8DAsKgo4VJto5WGVpbs5jgnM7LhbfoU5TThVrtXwfwk2dhSoAB31ufMi9c8R&Signature=ow5iLjLMloHo1EOuBl0iicX0uZw%3D' style="width: 200px" />
</div>
<!-- ![](成效图.jpg) -->

### 操作方式

#### 引入百度语音

<p>&nbsp;&nbsp<span style="background:#e1c953; color:#000;padding:5px 5px">manifest.json</span>&nbsp;&nbsp;文件</p>

<div><img src='https://chlblog.oss-cn-guangzhou.aliyuncs.com/yuyinshibie1.jpg?Expires=1719138259&OSSAccessKeyId=TMP.3KkWxBhPT2PLwHUkAck2fqXke4mp3QgWaU8DAsKgo4VJto5WGVpbs5jgnM7LhbfoU5TThVrtXwfwk2dhSoAB31ufMi9c8R&Signature=ow5iLjLMloHo1EOuBl0iicX0uZw%3D' style="width: 200px" /></dix>

#### 百度 AI 平台获取 key 值

https://blog.csdn.net/zcs2632008/article/details/123306370

#### 唤起百度语音

<p>&nbsp;&nbsp<span style="background:#e1c953; color:#000;padding:5px 5px">@longpress</span>&nbsp;&nbsp;长按屏幕</p><p>&nbsp;&nbsp<span style="background:#e1c953; color:#000;padding:5px 5px">@touchend</span>&nbsp;&nbsp;松开触碰</p>

```
 <view class="img"
    :class="longPressing ? 'longPressing' : ''"
    @longpress="startRecognize" @touchend="endRecognize" @touchmove="endRecognize">
    <image
        src="https://yizhu-new.oss-cn-shenzhen.aliyuncs.com/app_img/cImg/AIrecognition9.png"
    />
    </view>
    <view class="text" v-if="!longPressing">长按说话</view>
</view>
```

<p>&nbsp;&nbsp<span style="background:#e1c953; color:#000;padding:5px 5px">onLoad</span>&nbsp;&nbsp;钩子中进行监听</p>

```
// 监听语音识别事件
plus.speech.addEventListener('start', this.ontStart, false);
plus.speech.addEventListener('volumeChange', this.onVolumeChange, false);
plus.speech.addEventListener('recognizing', this.onRecognizing, false);
plus.speech.addEventListener('recognition', this.onRecognition, false);
plus.speech.addEventListener('end', this.onEnd, false);
```

<p>&nbsp;&nbsp<span style="background:#e1c953; color:#000;padding:5px 5px">@longpress="startRecognize"</span>&nbsp;&nbsp;开始识别</p>
```
startRecognize() {
    console.log('startRecognize');
    let options = {
    engine: 'baidu',
    lang: 'zh-cn',
    userInterface: true,
    continue: false,
    timeout:  60 * 1000,
    punctuation: false
    }
    plus.speech.startRecognize(options)
},
```

触发监听相关函数

```
ontStart() {
    this.title = '...倾听中...';
    this.text = '';
    console.log('Event: start');
},
onVolumeChange(e) {
    this.valueWidth = 100*e.volume+'px';
},
onRecognizing(e) {
    console.log('Event: recognizing');
},
onRecognition(e) {
    console.log(e.result,'e.resulte.resulte.result');
    this.text += e.result;
    this.text?(this.text+='\n'):this.text='';
    this.text_content = this.text;//将识别的到信息进行赋值
    console.log('Event: recognition');
    plus.speech.stopRecognize();
},
onEnd() {
    if(!this.text||this.text==''){
        plus.nativeUI.toast('没有识别到内容');
    }
    this.text_content = this.text;
    plus.speech.stopRecognize();
},
```

<p>&nbsp;&nbsp<span style="background:#e1c953; color:#000;padding:5px 5px"> @touchend="endRecognize"</span>&nbsp;&nbsp;结束识别</p>
```
endRecognize() {
    console.log('endRecognize');
    plus.speech.stopRecognize();
},
```
