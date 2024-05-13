---
title: windows安装mysql（version:5.6.17）
date: 2017-08-24 22:49:41
tags: [mysql]
categories: Mysql
---

说明：我安装的是免安装版  下载压缩文件解压就直接ok的

>新建文件 `my.ini`
```
[mysql]
# 设置mysql客户端默认字符集
default-character-set=utf8 
[mysqld]
#设置3306端口
port = 3306 
# 设置mysql的安装目录
basedir=D:\mysql\
# 设置mysql数据库的数据的存放目录
datadir=D:\mysql\data
# 允许最大连接数
max_connections=200
# 服务端使用的字符集默认为8比特编码的latin1字符集
character-set-server=utf8
# 创建新表时将使用的默认存储引擎
default-storage-engine=INNODB
```
>cmd输入 `mysqld install` 回车运行就行了

>输入 `net start mysql` 启动服务
```bash
net start mysql
```
>启动不了则先**删除** `data` 目录(或移动到其他地方)，再执行:
```bash
mysqld --initialize
```
>输入 `mysql -uroot -p` ,默认是没有密码的。回车进入
>有密码的话，可以 `mysql -uroot -p密码`
```bash
mysql -uroot -p
mysql -uroot -p密码
```
>也是可以是 `mysql -uroot -p` >回车输入密码，推荐第二种，原因，你动手之后就知道了。(其实是看看有没有输错 **QAQ**)
    
>退出 `exit` 就行了。记住直接关闭 `cmd` 窗口是没有退出的，要输入 `exit` 才会退出

>然后添加环境变量：

    
 <i class="icon-pencil"></i> 添加完成就可以直接：
```
进入cmd -> mysql -uroot -p -> 回车输入密码
显示所有数据库 -> show databases;
查找数据库 -> use 数据库名;
切换数据库目录 -> show tables;
查找表 -> sql查询工作select * from 表名;
退出 -> exit
```
