---
title: CSS拾遗
date: 2018-09-18 14:00:50
categories: 前端
description: css查漏补缺
---

# css的三种写处

## 内联式
`style="color:red;"`

## 嵌入式
```
<style>

</style>
```

## 外部式
css外部式
`<link href="css/style.css" rel="stylesheet">`

js外部式
`<script type="text/javascript" src="js/jquery-1.8.3.min.js"></script>`

## 优先级：内联式 > 嵌入式 > 外部式
其实总结来说，就是--就近原则（离被设置元素越近优先级别越高）。

# 选择器
```
 p{ color:red;}/*元素选择器*/
 
 #p1{color:blue;}/*id选择器*/
 
 .after{color:green; }/*类选择器*/
 
 *{color:red;}/*通用选择器*/
 
 a:hover{color:red;}/*伪类选择器*/
 
  h1,span{color:red;}/*两个都*/
```
选择官方参考手册http://www.w3school.com.cn/cssref/css_selectors.asp

!important 注意要写在分号的前面，比如
`a:hover{color:red!important;}/`
# 布局
这个慕课老师发了许多关于布局的课程，其中一些给人启发
http://www.imooc.com/t/197450
## 一、基本知识
- 块级元素
特点：
1.一个块级元素独占一行
2.元素的高度、宽度、行高以及顶和底边距都可设置。
常用的块状元素有：
`<div>、<p>、<h1>、<h6>、<ol>、<ul>、<dl>、<table>、<address>、<blockquote> 、<form>`


- 内联元素
特点：
1.和其他元素都在一行上。
2.元素的高度、宽度及顶部和底部边距**不可**设置；
常用的内联元素有：
`<a>、<span>、<br>、<i>、<em>、<strong>、<label>、<q>、<var>、<cite>、<code>`


inline-block 元素特点：
1、和其他元素都在一行上；
2、元素的高度、宽度、行高以及顶和底边距都**可**设置。

display:block  -- 显示为块级元素
display:inline  -- 显示为内联元素
display:inline-block -- 显示为内联块元素，表现为同行显示并可修改宽高内外边距等属性

tips：常将<ul>元素加上display:inline-block样式，原本垂直的列表就可以水平显示了。


## 二、定位
### float浮动定位
float的设计初衷仅仅是-文字环绕效果
属性有left、right、none

### positioin
relative相对定位：相对于它的初始位置而言移动。不可层叠。
absolute绝对定位：相对于最近的已定位的父元素。可层叠。
fixed悬浮定位：本质和absolute一样，不过不随浏览器滚动条向上或向下移动。


## 三、盒模型
元素实际宽度（盒子的宽度）=左边界+左边框+左填充+内容宽度+右填充+右边框+右边界。

## 四、布局模型
  1、流动模型（Flow）
 
  2、浮动模型 (Float)
  float的设计初衷仅仅是-文字环绕效果
  3、层模型（Layer）


# 静态网页常用代码

## 1.如何将网页整体居中？
- 方法一将整个网页放在一个大div容器下，容器设置margin和具体宽度。
`<div style="margin: 0 auto;width:1000px;">`
解释：上下边距为0，左右边距为自适应。
- 方法二1.定义大div的宽度 2.把容器position睡醒设置为relative 3.把left属性设置为50%
`<div style="width:1000px;position:relative;left:50%;">`


## 2.如何消除默认样式？
`2.<body style="margin:0;padding:0;">`


## 3.如何隐藏滚动条？
 - 方法一html { overflow-y: scroll; }
    原理：强制显示ie的垂直滚动条，而忽略水平滚动条
    优点：完全解决了这个问题, 允许你保持完整的XHTML doctype.
    缺点：即使页面不需要垂直滚动条的时候也会出现垂直滚动条。
    
 - 方法2:html { overflow-x: hidden; overflow-y: auto; }
    原理：隐藏横向滚动，垂直滚动根据内容自适应
    优点：在视觉上解决了这个问题.在不必要的时候, 未强制垂直滚动条出现.
    缺点：只是隐藏了水平滚动条，如果页面真正需要水平滚动条的时候，
    屏幕以外的内容会因为用户无法水平滚动，而看不到。


## 4.常用代码，搁这了，方便复制。
- 清除一些默认样式
 ```
*{margin:0;padding:0;}
body{font-size:12px;}
img{border:none;}
li{list-style:none;}
input,select,textarea{outline:none;}
textarea{resize:none;}
a{text-decoration:none;}
```
- 刚开始肯定就是刷刷几个div包起来
```
<div class=""></div>
<div class=""></div>
<div class=""></div>
```
- Bootstrap框架引用代码
```$xslt
<script src="http://how2j.cn/study/js/jquery/2.0.0/jquery.min.js"></script>
<link href="http://how2j.cn/study/css/bootstrap/3.3.6/bootstrap.min.css" rel="stylesheet">
<script src="http://how2j.cn/study/js/bootstrap/3.3.6/bootstrap.min.js"></script>
```
