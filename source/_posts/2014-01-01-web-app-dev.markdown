---
layout: post
title: "Web应用开发中的几个问题"
date: 2014-01-01 12:21:40 +0800
comments: true
categories: [Web Programming]
---

## Introduction

由于Ajax技术在Gmail中的成功应用和高性能的V8引擎的推出使得编写Web应用变得流行
起来，使用前端技术也可以编写具有复杂交互的应用。相对于native应用，Web应用具
有如下优点：

* 跨平台，开发和维护成本低；
* 升级和发布方便，没有版本的概念，随时随地发布，用户没有感知，不需要安装；
* 响应式设计(Responsive Design)使得Web应用可以跨平台，同一份代码自适应各种
  屏幕大小
* 即使最终不采用Web应用方案，也很适合开发原型

<!-- more -->

当然，Web应用也不是没有缺点。由于不同平台和厂商的浏览器并不完全一样，跨平台
也有一些兼容成本。另外，Web应用的性能不如native应用，交互有时候不是很流畅，
再加上HTML5的API上的限制，使得有些功能采用Web应用不太合适。由于这些原因，结
合两者优点的混合方案变得流行起来（比如微信、手机QQ和手机QQ浏览器中会嵌入一
些Web页面）。

根据笔者的开发经验，下面总结一些Web应用开发过程中的要面临的几个问题。

## 模块化编程

模块化编程是编写大规模应用必不可少的一个特性，与其它主流的编程语言相比
Javascript没有对模块提供直接的支持，更不用说维护模块之间的依赖关系，这使得维
护Javascript代码变得异常困难，在`<script>`标签中包含代码的顺序需要人工维护。

要支持模块化编程必须解决两个问题：

1. 支持编写模块并为模块命名，防止名字冲突和全局变量的使用；
2. 支持显示指定模块之间的依赖关系，并在程序执行时自动加载依赖的模块。

Douglas Crockford在"Javascript: The Good Parts"一书中提出的Module Pattern利
用Javascript的闭包技术来模拟模块的概念，防止名字冲突和全局变量的使用。这解
决了第一个问题。
``` js
var moduleName = function () {
    // Define private variables and functions
    var private = ...

    // Return public interface.
    return {
        foo: ...
    };
}();
```
为了解决第二个问题[CommonJS](http://www.commonjs.org/)组织定义了
[AMD规范](http://wiki.commonjs.org/wiki/Modules/AsynchronousDefinition)方便
开发者显示指定模块之间的依赖关系，并在需要时加载依赖的模块。
[RequireJS](http://requirejs.org/)是AMD规范的一个比较流行的实现。

首先我们在`a.js`中定义模块`A`.
``` js
define(function () {
    return {
        color: "black",
        size: 10
    };
});
```
然后定义模块`B`依赖模块`A`.
``` js
define(["a"], function (A) {
    // ...
});
```
当模块`B`执行时RequireJS保证模块`A`已被加载。具体细节可参考RequireJS官方文
档。

## 脚本加载

最简单的脚本加载方式是放在`<head>`加载。

``` html
<head>
  <script src="base.js" type="text/javascript"></script>
  <script src="app.js" type="text/javascript"></script>
</head>
```
其缺点是：

1. 加载和解析是顺序是同步执行的，先下载`base.js`然后解析和执行，然后再下载
`app.js`；
2. 加载脚本时还会阻塞对`<script>`之后的DOM元素的渲染。

为了缓解这些问题，现在的普遍做法是将`<script>`放在`<body>`的底部。

``` html
  <script src="base.js" type="text/javascript"></script>
  <script src="app.js" type="text/javascript"></script>
</body>
```
但并不是所有的脚本都可以放在`<body>`的底部，比如有些逻辑要在页面渲染时执行，
不过大多数脚本没有这样的要求。

将脚本放在`<body>`底部仍然没有解决顺序下载的问题，一些浏览器厂商也意识到了
这个问题并开始支持异步下载。HTML5也提供了标准的解决方案：

``` html
<script src="base.js" type="text/javascript" async></script>
<script src="app.js" type="text/javascript" async></script>
```
标上`async`属性的脚本表明你没有在里面使用`document.write`之类的代码。浏览器
将异步下载和执行这些脚本，并且不会阻止DOM树的渲染。但是这会导致另一个问题：
由于是异步执行，`app.js`可能在`base.js`之前执行，如果它们之间有依赖关系这将
导致错误。
    
讲到这里从开发者角度来看我们其实需要的是这些特性：

1. 异步下载，不要阻塞DOM的渲染；
2. 按照模块的依赖关系解析和执行脚本。

所以脚本的加载其实需要与模块化编程问题结合起来解决。RequireJS不仅记录了模
块之间的依赖关系，并且提供了根据依赖关系的按需加载和执行（详情请参考
RequireJS官方文档）。

关于脚本加载的更多方案请看
[这里](http://www.html5rocks.com/en/tutorials/speed/script-loading/?redirect_from_locale=zh).

## 静态资源文件的部署

这里的静态资源文件是指CSS、Javascript和CSS需要的一些图片文件。它们的部署需
要考虑两个问题：

1. 下载速度
2. 版本管理

静态资源文件的一个特点变化不频繁，且与用户身份无关（即与Cookie无关），因此
很适合缓存。另一方面，一旦静态资源文件变化时，浏览器必须从Web服务器下载最新
的版本。当发布新版本的Web应用时，并不是所有用户马上就用上新版本，老版本和新
版本将会共存，这就涉及到版本匹配问题。老版本的应用需要下载老版本的CSS和
Javascript，新版本的应用需要下载新版本的静态资源。

1. 为了防止版本不一致，每当发布新版本的应用时静态资源文件都需要改名，让旧的
   HTML引用旧的静态文件，新的HTML引用新的静态文件。一个常见办法就是在文件名
   中加时间戳；
2. 为了防止悬挂引用，资源文件应该比HTML先发布。

上述方案可以解决版本问题，这样每个静态文件的缓存时间可以设置得任意大，防止
重复下载，同时在新版本发布时浏览器将及时更新。

为解决下载速度问题，可以考虑以下几个方案：

1. 合并静态文件以免文件数量过多，过多的文件需要更多的连接来下载，浏览器通常
对同一个域名的连接数量有限制；
2. 压缩静态文件；为了可读性，CSS和Javascript通常有很多空行、缩进和注释，这
些在发布时都可以去掉；
3. 静态文件通常与Cookie没有关系，所以为了减小传输大小和增加缓存命中率（缓存
   的key需要考虑Cookie），静态文件最好托管在没有Cookie的域名上；

最后也是最重要的，要使上述过程自动化。

## MVC编程模型

Web应用采用的是事件驱动编程模型，与native应用是一样的，区别仅在于基础设施提
供的API不一样。UI编程通常采用MVC设计模式，以流行的
[Backbone.js](http://backbonejs.org/)为例包括如下部分：

1. Model
   * 数据的唯一来源
   * 负责获取和存储数据
   * 可提供缓存机制
   * 数据变化时通过事件通知其它对象
2. View
   * 负责渲染
   * 监听UI事件和Model事件并重绘UI
   * 渲染结果取决于两类数据：Model和UI交互状态
   * UI的交互状态通常存在View对象中，有时候为了方便也存在DOM树节点中
   * 为了降低渲染成本，尽量减少需要渲染的区域，每次当数据变化时只渲染受影响
     的区域
3. Router
   * 负责监听URL的变化，并通知相应的View对象渲染页面

为了有效地使用MVC，有几个问题需要注意。

### Model应与View完全隔离

Model仅提供数据的访问，不应该依赖View，因此Model不应该知道View的存在。所以
Model不能持有对任何View对象的引用。Model的数据发生变化时只能通过事件通知
View.

### View在初始化时采用委派方式监听UI事件

这里有两个关键点：

1. 在初始化时监听事件
``` js
var View = Backbone.View.extend({
    initialize: function () {
        this.$el.on('click', '#id', function () {
            // ...
        });
    }
});
```
除了一些特殊情况外（请看下文），所有UI事件都应该在View初始化时初始化，防止同
一个事件被绑定多次。即使有些事件是动态监听的（有时候需要监听，有时候有不需要
监听，比如有些按钮有时候是有效的，有时候又无效），也需要在初始化时监听，然后
在事件回调函数里判断是否需要处理。这样逻辑更简单，更容易维护。

2. 采用委派方式监听UI事件

关于委派方式监听请参考[jQuery文档](http://api.jquery.com/on/).

上面已强调要在初始化时监听事件，但是初始化时需要监听的DOM节点可能还不存在，
所以没法直接绑定事件，只能采用委派方式。不过采用委派方式要求事件可以冒泡。

对于那些没法冒泡的事件（比如`<img>`的`load`事件）只能在保证其存在的情况下直
接绑定，而不一定要在初始化时绑定。

### 复杂的View组织成树形层次结构

函数太大了需要拆分成几个子函数。同样，View的逻辑如果过于复杂也应根据页面结
构拆成几个子View：

1. 父View通过引用访问子View，但是子View不应该持有父View的引用；
2. 子View只负责自己区域的渲染，其它区域由父View负责渲染；
3. 父View通过函数调用访问子View的功能，子View通过事件与父View通信；
4. 子View之间不能直接通信。

其它技巧可查看
[Backbone技巧与模式](http://coding.smashingmagazine.com/2013/08/09/backbone-js-tips-patterns/).

## 离线应用缓存

为使Web应用体验更加流畅，可考虑使用HTML5离线应用缓存，不过有以下几点需要注
意：

1. 不要将离线应用缓存与HTTP缓存机制搞混淆，前者是HTML5引入的新特性，与HTTP缓
   存机制是相互独立并存的；
2. Cache manifest文件不应被HTTP缓存太久（通过HTTP头`Cache-Control`控制缓存
   时间），否则发布新版后浏览器不会及时监测到变化并下载新文件；
3. 在Cache manifest文件的`NETWORK`节放一个`*`，否则没有列在这个文件的资源不
   会被请求；
4. 不适合缓存的请求最好都放在`NETWORK`节；
5. 如果之前使用过离线应用缓存现在不想再使用了，从`<html>`删除`manifest`属性，
   并发送404响应给manifest文件请求。仅仅删除`manifest`属性是没有效的。

## 线上错误报告

Javascript是一个动态语言，许多检查都是在运行时执行的，所以大多数错误只有执
行到的时候才能检查到，只能在发布前通过大量测试来发现。即使这样仍可能有少数
没有执行到的路径有错误，这只能通过线上错误报告来发现了。
``` js
window.onerror = function (errorMsg, fileLoc, linenumber) {
    var s = 'url: ' + document.URL + '\nfile:  ' + fileLoc
        + '\nline number: ' + linenumber
        + '\nmessage: ' + errorMsg;
    Log.error(s);       // 发给服务器统计监控
    console.log(s);
};
```
通常线上的Javascript都是经过了合并和压缩的，上报的文件名和行号基本上没法对
应到源代码，对查错帮助不是很大。不过最新版的Chrome支持在`onerror`的回调函数
中获取出错时的栈轨迹：`window.event.error.stack`.
