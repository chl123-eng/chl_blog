---
title: 正反向+单选多选混合tab
date: 2024-08-23 17:49:03
tags: 常用组件封装
categories:
  - 常用组件封装
cover: https://chlblog.oss-cn-guangzhou.aliyuncs.com/switchTab1.jpg
---
---

### 前言

因为业务上多处地方使用到这样的功能组件，为了降低代码冗余率，增强代码复用性，因为封装了公用组件

使用场景：存在单选、多选；某个选项有正向排列和反向排列的选择

```
<template>
    <view class="switchTabs">
        <view class="s_bar" v-for="(item,index) in tabsList" :key="index" :class="item.isActive ? 's_bar_active' : ''" @click="selectSearchBar(item)">
            <view class="s_bar_text">{{ item.name }}</view>
            <view class="s_bar_icon background_img_base" v-if="item.hasIcon && !item.isActive"></view>
            <view class="s_bar_icon_active background_img_base" v-if="item.hasIcon && item.isActive" :class="item.value == item.downValue ? '' : 'icon_desc'"></view> 
        </view>
    </view>
</template>
<script>
export default{
    props:{
        list:{
            type: Array,
            default: (() => [])
        }
    },
    data(){
        return {
            //传值示例
            // search_bar_list: [
            //     {
            //         name: "综合",
            //         isActive: true,
            //         hasIcon: false,
            //         value: "default",
            //         isSingle: true
            //     },
            //     {
            //         name: "最新",
            //         isActive: false,
            //         hasIcon: true,
            //         value: "time_desc",
            //         upValue: "time_asc",
            //         downValue: "time_desc",
            //         isSingle: false
            //     },
            //     {
            //         name: "热门",
            //         isActive: false,
            //         hasIcon: true,
            //         value: "hot_desc",
            //         upValue: "hot_asc",
            //         downValue: "hot_desc",
            //         isSingle: false
            //     }
            // ],
            tabsList: this.list
        }
    },
    methods:{
        selectSearchBar(itemBar){
            this.tabsList = this.tabsList.filter((item) => {
                if(itemBar.hasIcon && itemBar.name == item.name){
                    if(itemBar.value == item.downValue){
                        item.value = item.upValue;
                    }else{
                        item.value = item.downValue;
                    }
                }
                if(itemBar.isSingle){
                    item.isActive = itemBar.name == item.name;
                }else{
                    if(item.name == itemBar.name){
                        item.isActive = true;
                    }
                    if(item.isSingle){
                        item.isActive = false;
                    }
                }
                return item
            })
            
        },
    }
}
</script>

<style lang="less" scoped>
.switchTabs{
    display: flex;
    .s_bar{
        margin-right: 40rpx;
        font-size: 28rpx;
        color: #999999;
        display: flex;
        align-items: center;
        .s_bar_icon{
            width: 16rpx;
            height: 10rpx;
            margin-left: 5rpx;
            margin-top: 5rpx;
            background-image: url(https://yizhu-new.oss-cn-shenzhen.aliyuncs.com/app_img/aIcon/icon508.png);
        }
        .s_bar_icon_active{
            width: 16rpx;
            height: 10rpx;
            margin-left: 5rpx;
            margin-top: 5rpx;
            background-image: url(https://yizhu-new.oss-cn-shenzhen.aliyuncs.com/app_img/aIcon/icon509.png);
        }
        .icon_desc{
            transform: rotate(180deg);
        }
    }
    .s_bar_active{
        color: #0988F3;
    }
}
</style>
```