---
layout: post
title: CSS Best Practice
comments: true
categories:
- web programming
---

## Use a reset styelsheet

Reset规则可以消除不同浏览器的差异，比较流行的reset规则有[Meyer
Reset](http://meyerweb.com/eric/tools/css/reset/index.html)和[Yahoo
Reset](http://developer.yahoo.com/yui/reset/).
<!--more-->

## Categorize CSS Rules

将样式规则分类，不同类型的样式采用不同的CSS规则描述，以方便阅读、理解和复用，
见[Scalable and Modular Architecture for CSS](http://smacss.com/book/).

## Use Object-Oriented CSS

面向对象的CSS的两条基本原则是：

*  将结构和皮肤分开
*  将容器和内容分开

遵循面向对象的原则可以使代码更加模块化和可重用。见[OOCSS](https://github.com/stubbornella/oocss/wiki).

## Use "margin: 0 auto" to Center Layouts

Inline元素用`text-align: center`居中对齐，但是block元素没有对应的CSS属性。

## Use Text-transform

`text-transform`有以下几个属性：

*  capitalize: 每个单词的首字母大写
*  uppercase: 全部大写
*  lowercase: 全部小写

## Use em as the unit of font size

See [http://webdesign.about.com/cs/typemeasurements/a/aa042803a.htm](http://webdesign.about.com/cs/typemeasurements/a/aa042803a.htm).

## Understanding CSS selector performance

复杂的CSS选择符会使浏览器耗用更多的时间计算一条CSS规则是否匹配一个元素，因
此为了让浏览器更快地渲染页面我们应该使用仅可能简单的选择符。

1.    使用单个选择符；
2.    使用子选择符；
3.    避免使用常见的标签名作选择符。

See ["Selector Performance"](http://smacss.com/book/selectors), and
["Writing Efficient CSS"](https://developer.mozilla.org/en-US/docs/CSS/Writing_Efficient_CSS).
