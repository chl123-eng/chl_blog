---
title: CSS多行溢出处理
date: 2019-10-03
tags:
  - CSS
  - SCSS
categories:
  - CSS样式
abstracts: CSS多行溢出处理
---

## 前言

之前做过的布局文字溢出效果 一段时间不用之后就老是忘记 今天记录下来

## 单行溢出

```css
overflow: hidden;
text-overflow: ellipsis;
white-space: nowrap;
```

## 多行溢出(webkit)

```css
overflow: hidden;
text-overflow: ellipsis;
display: -webkit-box;
-webkit-line-clamp: 2; // 在第几行处省略-webkit-box-orient: vertical;
```
