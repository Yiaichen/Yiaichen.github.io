---
title: Github+Hexo
tags: [hexo]
categories: Hexo
---
记录整个搭建Hexo的步骤吧~

## 准备工作

1、[git](https://git-scm.com/downloads)
2、[nodejs](https://nodejs.org/en/)
3、[github](https://github.com/)账号

### 第一步

首先把git跟nodejs下载ok(地址在上方)
安装的话就不细说了，一直next就可以了
验证有没有安装好的话直接在控制台中输出：

``` bash
$ node -v
```

``` bash
$ npm -v
```

``` bash
$ git --version
```

<img src="/img/version.png">


如果有对应的版本输出就说明安装已经ok啦

### 第二步

在本地建立一个blog文件夹（我是建在了D盘下面）
然后打开git-bash，千万别用cmd跑后面的命令了，这里要圈起来，重点要考！！！

现在开始安装Hexo了，cd进我们的blog目录下：

``` bash
$ npm install hexo-cli -g
```

<img src="/img/hexocli.png">

这里会提示一个warn不用管他，然后输入：

``` bash
$ npm install hexo --save
```

<img src="/img/--sava.png">

我这里直接用cmd截图了，这里无所谓，最好都用git-bash，出现了一堆白字跟warn之后，我们得hexo就安装好了，输入：

``` bash
$ hexo -v
```

<img src="/img/hexo -v.png">

如果看到对应的hexo版本信息就说明已经安装ok了


### 第三步

安装好了hexo我们就要开始来使用他了，首先执行初始化：

``` bash
$ hexo init
```

<img src="/img/hexo init.png">

这里会在我们自己建的目录下生成hexo的文件，如果执行hexo init的时候报not empty之类的错
解决方案是建议删除目录下的所有文件  然后重新执行一次hexo init命令

然后输入：

``` bash
$ npm install
```

<img src="/img/npm install.png">

之后npm将会自动安装你需要的组件，只需要等待npm操作即可。

然后在命令行输入：

``` bash
$ hexo g
```

``` bash
$ hexo s
```

<img src="/img/hexo g.png">

hexo g的话会在目录下生成一个public文件夹，hexo s就是启动服务啦~
如果显示上面的信息的话就在本地已经启动ok啦~
然后我们在浏览器中输入：http://localhost:4000/

<img src="/img/hexo.png">

大功告成~ （停止服务的话Ctrl+c就可以了）

### 第四步

本地发布部署

``` bash
$ hexo clean
$ hexo g
$ hexo s
```

后面有改动的话基本就是这三个命令，然后本地就可以显示更新后的内容啦~

