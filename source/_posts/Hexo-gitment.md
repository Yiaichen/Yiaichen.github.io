---
title: Hexo+gitment
date: 2017-08-16 22:16:59
tags: [hexo]
categories: Hexo
---

因为多说关闭的原因，所以选择gitment来搭建留言评论 /(ㄒoㄒ)/~~

## 搭建步骤

### 1. 注册 OAuth Application

首先需要注册一个OAuth Application [点击此处](https://github.com/settings/applications/new)

<img src="/img/oauth.png">

其他内容可以随意填写，但要确保填入正确的 callback URL（一般是评论页面对应的域名，比如 我的是https://yiaichen.github.io/）
注册完成之后你会得到一个 client ID 和一个 client secret，这个将被用于之后的用户登录。

### 2. 引入 Gitment

将下面的代码添加到你的页面：

``` html
<div id="container"></div>
<link rel="stylesheet" href="https://imsun.github.io/gitment/style/default.css">
<script src="https://imsun.github.io/gitment/dist/gitment.browser.js"></script>
<script>
var gitment = new Gitment({
  id: '页面 ID', // 可选。默认为 location.href
  owner: '你的 GitHub ID',
  repo: '存储评论的 repo',
  oauth: {
    client_id: '你的 client ID',
    client_secret: '你的 client secret',
  },
})
gitment.render('container')
</script>
```

这里的id是可选的  不填就删掉（这里画个圈，因为我是踩过坑的 QAQ）
owner：这里填github的用户名 比如我就填 Yiaichen
repo：这里是存储评论的仓库 一般是建一个仓库地址 我这边就是 Yiaichen.github.io

<img src="/img/owner.png">

然后下面的client_id 跟 client_secret就填你刚刚注册得到的就ok了

### 3. 初始化

其实到这里差不多已经配置好了 只需要初始化一下
发布你的页面 （不要在本地测试，因为本地是一定初始化失败的）
然后登录你的github账号，必须跟第二步owner用户名相同的账号
登录之后点击初始化按钮，bingo~

<img src="/img/gitment.png">


### 4.常见错误

ERROR:NOT FOUND ：一般是owner或者repo配置错误了，照着第二步来就好
初始化的时候报alidation failed ：都说已经踩过的坑了啦QAQ 自己往上找吧/(ㄒoㄒ)/~~




