---
title: uniapp下小程序获取详细定位
date: 2024-04-15 15:53:36
tags:
  - uniapp小程序
  - 地图
cover: https://blog-1300014307.cos.ap-guangzhou.myqcloud.com/gddt.png
categories:
  - 经验
feature: true
---

### 前言

<p>uniapp自带的API&nbsp;&nbsp; <span style="background:#a59aca; color:#fff;padding:5px 5px">uni.getLocation</span>&nbsp;&nbsp;仅APP端支持获取adress，小程序是不支持的。</p>
<p>
<p>因此可以借助腾讯/高德的API&nbsp;&nbsp; <span style="background:#a59aca; color:#fff;padding:5px 5px">地理/逆地理编码</span></p>

### 授权地理位置

<span style="background:#a59aca; color:#fff;padding:5px 5px">manifest.json</span> 该文件中添加以下代码，会弹出位置授权弹框。

```
"mp-weixin" : {
    "permission" : {
        "scope.userLocation" : {
            "desc" : "将获取你的地址"
        }
    },
}
```

<p>但有些用户可能会选择关闭，因为当进入到需要获取地理位置的的页面时就无法进行定位，因为最好在具体页面再进行一次<em><strong>位置的授权</strong></em></p>

<!-- ![](./location/getLocationAuthority.png) -->

```
 getIsAuthority(){
    var that = this;
    uni.getSetting({
        success: (res) => {
            if (res.authSetting['scope.userLocation'] != undefined && res.authSetting['scope.userLocation'] != true) {//非初始化进入该页面,且未授权
            uni.showModal({
                title: '是否授权当前位置',
                content: '需要获取您的地理位置，请确认授权，否则地图功能将无法使用',
                success: function (res) {
                if (res.cancel) {
                    console.info("授权失败返回数据");

                } else if (res.confirm) {
                    uni.openSetting({
                        success: function (data) {
                            if (data.authSetting["scope.userLocation"] == true) {
                            uni.showToast({
                                title: '授权成功',
                                icon: 'success',
                                duration: 5000
                            })
                            //再次授权，调用getLocationt的API
                            that.getCurrentLocation(that);
                            }else{
                            uni.showToast({
                                title: '授权失败',
                                icon: 'success',
                                duration: 5000
                            })
                            }
                        }
                        })
                    }
                    }
                })
            } else if (res.authSetting['scope.userLocation'] == undefined) {//初始化进入
                that.getCurrentLocation(that);
            }else{
                that.getCurrentLocation(that);
            }
        }
    })
},
```

### 腾讯位置服务进行设置

[腾讯位置服务](https://lbs.qq.com/service/webService/webServiceGuide/address/Gcoder)

详细步骤点击链接查看:[步骤设置](https://blog.csdn.net/Ronion123/article/details/133887415)

```
 //获取定位
    getCurrentLocation(thatParam) {
        let that = thatParam
        uni.getLocation({
            type: 'gcj02',// (！！！必需)默认为 wgs84 返回 gps 坐标，gcj02（更准确） 返回可用于 uni.openLocation 的坐标
            isHighAccuracy: true,//开启高精度定位(！！！必需)
            success: function(res) {
                that.form.longitude = "经度： " +  res.longitude.toFixed(5) + "  " + "纬度： " + res.latitude.toFixed(5);
                //根据获取的经纬度去解析当前坐标的地址信息
                that.loAcquire(res.longitude, res.latitude)
            },
            fail: function(error) {
                uni.showToast({
                    title: '无法获取位置信息！无法使用位置功能',
                    icon: 'none',
                })
            }
        });
    },
    //根据获取的经纬度去解析当前坐标的地址信息
    loAcquire(longitude, latitude) {
        let that = this;
        uni.showLoading({
            title: '加载中',
            mask: true
        });
        uni.request({
            url: 'https://apis.map.qq.com/ws/geocoder/v1/?location=',
            method: 'GET',
            data: {
                key: key,
                location: `${res.longitude},${res.latitude}`
            },
            success: (e) => {
            },
            fail: () => {

            }
        })
    }
```
