---
layout: post
title:  使用vue开发微信公众号下SPA站点的填坑之旅
crawlertitle: "使用vue开发微信公众号下前后端分离的SPA站点的填坑之旅，包括微信支付和微信JSSDK授权"
summary: "前端时间做一个微信公众号的新项目，打算做一个前后端分离的SPA单页站点，调研后决定使用vue全家桶(vue+vue-router+vuex)，本文为实际过程中的填坑之旅"
categories: posts
tags: ['vue', 'JSSDK', 'SPA']
author: greedying
math: y
---

本文为我创业过程中，开发项目的填坑之旅。作为一个技术宅男，我的项目是做一个微信公众号，前后端全部自己搞定，不浪费国家一分钱^_^。

### 直接放最佳实践，解决SPA下使用jSSDk和微信支付问题

>注意，下面是我总结的最佳实践，但并非唯一方法  

- vue-router路由使用hash模式(#分隔)  
- 项目url(即js中location.href)分隔符#前面需要增加一个问号(?)。如果没有问号，则js跳转到有问号的url上  
- 签名or加密的时候，wx.config签名通过window.location.href.split(‘#’)[0]获取签名使用的url  
- 接上，而微信支付签名使用的url，通过用window.location.href获取  
- 每次url更改的时候，重新调用JSSDK的config接口  
- 为了解决微信支付要求至少二级目录的问题，所有前端url，统一加一个**/frontend**前缀，变成 http(s)://domain.com/frontend/?#hashstring 的形式，同时在微信后台设置支付目录为 http(s)://domain.com/frontend/  
- 每次url变化后，重新进行微信config，并且重新设置微信分享接口(onMenuShare系列接口)  


### 我决定实现如下功能

- 架构上，实现前后端分离。方便以后前后端的分工
- 考虑到体验，前端做成SPA站点，也就是单页面应用
- 需要使用微信的JSSDK
- 需要有微信支付功能

作为一个偏后端的半专业前端人士，经过一两周的调研和学习后，

### 我决定使用如下技术

- 后端使用php搭建接口，本文主要讲前端，不细说
- webpack实现前端代码打包
- [vue](https://github.com/vuejs/vue)实现数据绑定，[vue-router](https://github.com/vuejs/vue-router)实现前端路由
- [weui](https://github.com/weui/weui)提供UI框架
- [vux](https://vux.li/#/)，提供各种组件，包括对weui的组件化封装

然后

### 我遇到了如下的坑

1. 微信JSSDK签名出错
2. 微信支付签名出错
3. 微信支付路径要求二级或以上路径
4. 开启调试模式后，微信支付仍然没有错误提示
5. 授权回调处理
6. 微信的模板消息，会自动把url中的问号(?)去掉
7. 动态设置页面的title
8. ios下微信分享页面错误问题


一一详述

#### 微信JSSDK签名出错

JSSDK在普通网站中是没问题的，但是在SPA站点中，签名经常出错

[JSSDK官方文档](https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1421141115&token=&lang=zh_CN)是这么说的

>所有需要使用JS-SDK的页面必须先注入配置信息，否则将无法调用（同一个url仅需调用一次，对于变化url的SPA的web app可在每次url变化时进行调用,目前Android微信客户端不支持pushState的H5新特性，所以使用pushState来实现web app的页面会导致签名失败，此问题会在Android6.2中修复）。

也就是说，android下的微信客户端里，不支持vue-router的history模式。

解决办法见支付签名问题  

- vue-router使用hash模式 
- 每次url更改的时候，重新调用JSSDK的config接口 

#### 微信支付签名出错

支付授权的坑，大家可以参考[这篇文章](http://www.kejik.com/article/152868.html)
按照文中的描述，其实我们也可以在js中根据android还是ios，来分别进行处理；但是推荐采用文中的方式，逻辑上更统一，使用也更方便。

另外说明一点，文中的**#!**做分隔符的方式已经废弃了，大家使用**#**即可，叹号(!)去掉了

另外就是wx.config的签名url和支付签名url，微信处理也不一样，见下面的解决办法

**解决办法**  

- vue-router路由使用hash模式  
- 每次url更改的时候，重新调用JSSDK的config接口  
- hash分隔(#)前面加一个问号(?)，如果js判断没有问号，则跳转一次  
- wx.config签名使用的url，通过**window.location.href.split('#')[0]**获取  
- 微信支付签名使用的url，通过用window.location.href获取  

#### 微信支付路径要求二级或以上路径
在遇到这个问题之前，我的php接口都统一加了一个前缀api，也就是***http://example.com/api/other***这样的url，服务器会自动转发给php服务，其他url则转发给前端服务器。遇到微信这个问题后，我把前端url也统一加了一个前缀frontend，这样前端url就变成了***http://example.com/frontend/?#hash***

**解决办法**

- 所有前端url，统一加一个**/frontend**前缀  

#### 开启调试模式后，微信支付仍然没有错误提示
不止微信支付，JSSDK的其他接口，也经常没有错误提示，或者提示很模糊，微信这简直是慢性谋杀。
不过我对比发现，ios下的各种提示，要比android下全面很多，如有必要，推荐大家在ios下进行调试  

**解决办法**

- 使用iphone进行开发调试  


#### 授权回调处理
这个不算坑，只是说下我的处理。
每次加载页面后，我都会调用后台接口判断是否登陆，如果没登陆，则跳转回到后台url进行授权，授权后再跳转回当前页面


#### 微信的模板消息自动去掉url的问号(?)

前面解决微信签名问题的时候，我们给每个url特意加了一个问号(?)，但是我发现，在发送微信模板消息的时候，即使给微信的url是对的，当用户点击模板消息的时候，微信打开的链接中，仍然把问号去掉了，这个很让人无语。考虑到尽量自己解决问题的原则，最后我的解决方案是在js中进行判断处理，自动把缺失的问号加上

**解决办法**  

- 如果页面没有问号(?)，则跳转到正确的url，代码如下  

```javascript
function directRightUrl () {
  let paths = window.location.href.split('#')
  paths[1] = paths[1] || '/'
  // 老式的#!分隔跳转
  if (paths[0].charAt(paths[0].length - 1) !== '?') {
    paths[0] = `${paths[0]}?`
  }
  if (paths[1].charAt(0) === '!') {
     paths[1] = paths[1].substr(1)
  }
  let url = `${paths[0]}#${paths[1]}`
  if (window.location.href !== url) {
    window.location.href = url
  }
}
```

以上代码有三个作用
1. 自动添加问号(?)
2. 自动把分隔符由#!变成#
3. 分隔符后面，自动判断是否为斜杠(/)，没有则添加上


#### 动态设置页面的title

微信网页里面，因为会在最上面显示页面的title，所以title的设置很重要。想当然的，我们会这么写

```javascript
  document.title = 'new title'

```

这句代码在android中是没问题的，但是在ios中却无效。
这个问题，应该说是ios的问题，锅不能让微信背，但我遇到了，也就写在这里。
这方面文章很多，我也是抄袭的网上代码

**解决办法**

每次url变化的时候，都调用以下函数  

```javascript
function setDocumentTitle (title) {
  title = title || '默认titile'
  document.title = title
  if (/ip(hone|od|ad)/i.test(navigator.userAgent)) {
    let iframe = document.createElement('iframe')
    iframe.src = '/MP_verify_zxjwxCcP80t475ww.txt'
    iframe.style.display = 'none'
    iframe.onload = function () {
      setTimeout(function () {
        iframe.remove()
      }, 0)
      document.body.appendChild(iframe)
    }
  }
}
```


2.x版本的vux也内置了一个插件做同样的事情，我看了下，代码基本一致，应该是抄的同一个地方，O(∩_∩)O~，大家可以参考下

#### ios下微信分享页面错误问题
简单讲，我打开SPA的时候是页面A，然后经过一系列操作，跳转到了页面B，这时候点击微信右上角的分享按钮，发送给别人或者分享到朋友圈，别人打开后，仍然是页面A

这个问题，其实和前面支付签名的问题是一样的，都是对当前url的判定不同。

幸好微信的JSSDK提供了以下几个接口能够设定分享的url和图片等

1.onMenuShareTimeline  
2.onMenuShareAppMessage  
3.onMenuShareQQ 
4.onMenuShareWeibo  
5.onMenuShareQZone  

**解决办法**

每次url变化的时候，调用onbMenuShare系列接口，设定分享的各种信息

### 结束语
以上就是我在开发过程中遇到的一些还记得的坑，欢迎大家探讨

另外介绍一下我的公众号**事事约**
这是一个帮助大家实现目标的项目，大家可以定一个目标，交付一定押金，相当于立了一个flag，然后转发给朋友进行监督，根据结果来决定最后押金是给你的朋友还是返还给你

扫描以下二维码可以关注
<img alt="事事约" title="事事约" style="width=50%;"  src="{{ site.images }}/ssy.jpg" />
