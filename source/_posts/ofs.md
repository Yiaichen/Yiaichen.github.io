---
title: ofs1.5部署
date: 2019-12-25 10:33:58
tags: [Linux]
categories: Linux
---

### 服务器配置（推荐）


服务器类型 | 数量/台 | CPU | 内存 | 系统磁盘 | 数据磁盘 | 网络带宽 | 操作系统
:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:
应用服务器 | 1 | 8核 | 32G | 40G | 300G SSD | 4M | Ubuntu Server 16.04
接口服务器 | 1 | 4核 | 16G | 40G | 300G SSD | 5M | Ubuntu Server 16.04
RDS数据库(高可用版) | 1 | 8核 | 32G | / | 1T SSD | / | Mysql 5.7
基础服务器 | 1 | 4核 | 16G | 40G | 500G SSD | 2M | Ubuntu Server 16.04
报表服务器 | 1 | 4核 | 16G | 40G | 200G | 2M | Ubuntu Server 16.04
测试服务器 | 1 | 4核 | 16G | 40G | 500G | 2M | Ubuntu Server 16.04
测试RDS数据库(高可用版) | 1 | 4核 | 16G | / | 500G SSD | / | Mysql 5.7

>   备注：如果有京东平台业务，还需加购一台同等配置京东云鼎ECS服务器、京东云鼎RDS服务器

### 服务器类型

服务器类型 | 部署应用
:----:|:----:
应用服务器 | ofs
接口服务器 | 各种edi
RDS数据库(高可用版) | mysql 
基础服务器 | redis、mongoDB、rabbitMQ
报表服务器 | ofs(只给报表类功能使用)
测试服务器 | 所有应用
测试RDS数据库(高可用版) | mysql

>   <font color=#DC143C>**以下流程仅以测试服务器做演示, 生产服务器以上述表格为标准, 具体根据项目情况进行改动**</font>

### 添加用户
>   系统必须在ttx用户下运行，所以需要为新系统添加ttx用户

>   以下操作以root用户权限执行


- 新建用户并指定用户目录
    ```
    useradd -d /home/ttx -m ttx
    passwd ttx
    ```
    
    >   用户密码重启之后才会生效
    
- 赋予sudo权限
    ```
    vi /etc/sudoers
    ```
    
    >   在root下添加一行，如下所示
    
    >   [注意] 此文件为只读文件，请保存时使用:`wq!`命令
    
    ```
    # User privilege specification
    root    ALL=(ALL:ALL) ALL
    ttx     ALL=(ALL:ALL) ALL
    ```
    
    >   [可选] 修改新建用户的Shell类型
    
    >   找到ttx的一行，检查是否与下面一致，如不一致，修改

    ```
    root@iZ8vbeurni16yy34ptzcdzZ:/# vi /etc/passwd
    ttx:x:1000:1000::/home/ttx:/bin/bash
    ```

- 创建应用主目录
    ```
    mkdir /home/ttx/app
    ```


### 挂载数据盘
    仅针对数据盘默认未挂载的情况，比如阿里云。(这里以50G数据盘做演示)
    以下操作以root用户权限执行

- 查看设备目录，通常数据盘为`/dev/vdb`
    ```
    fdisk -l
    ```
    
    ![image](http://img.localhostes.com/images/2019/12/22/fdiskbefore.png)

- 创建分区

    ```
    fdisk /dev/vdb
    ```
    
    >   根据提示，依次输入**n->p->1->回车->回车->wq**成功后会有以下输出

    ![image](http://img.localhostes.com/images/2019/12/22/fdisk.png)
    
    >   检查分区是否创建成功

    ```
    fdisk -l
    ```
    ![image](http://img.localhostes.com/images/2019/12/22/fdiskafter.png)

    >

- 格式化 
    ```
    mkfs.ext4 /dev/vdb1
    ```
    
    ![image](http://img.localhostes.com/images/2019/12/22/mkfs.png)

- 挂载数据盘

    ```
    chown ttx:ttx home/ttx/app/
    mount /dev/vdb1 /home/ttx/app
    ```
    
    >   检查挂载结果 `df -h`
    
    ![image](http://img.localhostes.com/images/2019/12/22/df-h.png)
    

- 设置自动挂载（请根据实际挂载硬盘参数对下面命令做适当修改）
    ```
    echo '/dev/vdb1 /home/ttx/app ext4 defaults 0 0 ' >>/etc/fstab  
    ```



### 安装基础运行环境

#### 准备工作
- 以创建好的`ttx`身份进行登录
- 创建存放安装包的目录: `mkdir /home/ttx/installer`
- 准备好[安装包](http://www.baidu.com)


#### 安装docker

- 使用阿里解决方案 安装`docker-engine`

    ```
    # [docker-engine已停止支持，请使用以下脚本安装]

    # step 1: 安装必要的一些系统工具
    sudo apt-get update
    sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
    # step 2: 安装GPG证书
    curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
    # Step 3: 写入软件源信息
    sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
    # Step 4: 更新并安装 Docker-CE
    sudo apt-get -y update
    sudo apt-get -y install docker-ce
    
    # https://help.aliyun.com/document_detail/60742.html
    ```
    
    >   注意：如果curl不存在，请使用以下命令安装
    
    ```
    sudo apt-get install curl
    ```
    
    >   **非必要流程** 修改daemon配置文件`/etc/docker/daemon.json`来使用加速器
    
    ```
    sudo mkdir -p /etc/docker
    sudo tee /etc/docker/daemon.json <<-EOF
    {
      "registry-mirrors": ["https://gkhkf8gb.mirror.aliyuncs.com"]
    }
    EOF
    sudo systemctl daemon-reload
    sudo systemctl restart docker
    ```
    
    ![image](http://img.localhostes.com/images/2019/12/22/EOF.png)
    
    
- 将用户ttx加入允许执行docker组
    ```
    sudo usermod -aG docker ttx
    ```


#### 安装docker-compose
- 上传 `docker-compose-Linux-x86_64.1.7.1` 文件到`/home/ttx/installer` 也可以下载最新版本

    ```
    sudo curl -L "https://github.com/docker/compose/releases/download/1.23.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose`
    ```
    
- 执行以下脚本安装
    > 安装

    ```
    cd /home/ttx/installer
    #复制并重命名
    sudo cp docker-compose-Linux-x86_64.1.7.1 /usr/local/bin/docker-compose
    #更改文件权限使其可以执行　+x　代表执行权限
    sudo chmod +x /usr/local/bin/docker-compose  
    ```
    
    >   检查安装结果
    
    ```
    ttx@iZ8vbeurni16yy34ptzcdzZ:/# docker-compose --version
    docker-compose version 1.7.1, build 0a9ab35
    ```


#### mongo、rabbit、redis、mysql
>   创建镜像存放目录

```
sudo mkdir /home/ttx/installer/images
sudo mkdir /home/ttx/installer/compose
```

- 上传镜像
    - 上传对应的镜像文件到`/home/ttx/installer/images`
    - 上传compose.zip到`/home/ttx/installer/compose`

- 安装镜像
    - 执行以下脚本装载镜像
    ```
    cd /home/ttx/installer/images
    sudo docker load < mongo_3.2.4.tar
    sudo docker load < rabbitmq_3.6.1-management.tar
    sudo docker load < redis_3.2.0.tar
    #仅当mysql在容器中运行时才运行以下命令
    sudo docker load < mysql_5.7.11.tar
    ```


- 启动服务 分别进入每一个容器目录，执行以下脚本
    ```
    sudo docker-compose up -d
    ```
    
    >   检查镜像是否创建成功 `docker ps`
    
    ![image](http://img.localhostes.com/images/2019/12/22/dockerps.png)
    
    - 注意：MySql参数设置在 `/etc/my.cnf` 中的`[mysqld]`后添加以下段：
        - 使用utf-8编码：`character-set-server=utf8`
        - 表不区分大小写：`lower_case_table_names=1`
        - 分组连接最大长度：`group_concat_max_len=102400`
        - 缓冲池字节大小：`innodb_buffer_pool_size = 数据库内存一半`
        
    - 注意：创建rabbitMQ的`虚拟主机(VirtualHost)`新安装的rabbitMQ必须新建主机，不允许使用默认的。新主机名称为`企业ID`，企业ID的格式为`4位项目编码 + ofs +xxxx`，其中xxxx为`prod`或者`test`登录`http://服务器地址:35673`进行创建，创建后将主机权限赋予ttx用户
    


#### tomcat
>   以下操作以ttx用户权限执行

- 上传**tomcat-deploy-1.6.zip到/home/ttx/installer**目录下
    - 解压缩
        ```
        sudo unzip ./tomcat-deploy-1.6.zip -d /home/ttx/app/tomcat-deploy
        ```
        
    - 创建系统环境
    
        >   运行前请检查`set_env.py`中`APP_HOME`的目录是否正确，应为`/home/ttx/app/`
        
        ```
        cd /home/ttx/app/tomcat-deploy
        python install_env.py
        ```

>   如果服务器上没有python命令，而只有python3，则使用python3代替


#### nginx
>   以下操作以ttx用户权限执行

- 上传**nginx-1.11.6.tar.gz到/home/ttx/installer**目录下
- 执行以下脚本安装
     ```
     sudo apt-get install build-essential  libpcre3-dev libssl-dev
     sudo tar -zxvf /home/ttx/installer/nginx-1.11.6.tar.gz -C /home/ttx/app/
     cd /home/ttx/app
     cd nginx-1.11.6
     #对nginx进行配置
     sudo ./configure --with-http_ssl_module --prefix=/home/ttx/app/nginx/
     #编译安装
     sudo make && make install
     cd /home/ttx/app/nginx
     # 备份配置文件
     sudo mv conf/nginx.conf conf/nginx.conf.backup
     sudo mkdir webapps
     ```
>   tar命令如果不存在的话先下载

>   如果服务器上没有python命令，而只有python3，则使用python3代替

-   上传配置文件`nginx.conf`和`proxy.conf`到`/home/ttx/app/nginx/conf/`目录下

- 在[nexus](http://nexus.cybertrans.ittx.com.cn/)上下载对应的`cbt-web`(这里以`cbt-web-2.5.12`做演示)
    - 搜索`cbt-web-2.5.12`
        ![image](http://img.localhostes.com/images/2019/12/23/nexus.png)

    - 点击对应版本
        ![image](http://img.localhostes.com/images/2019/12/23/seach.png)
        
    - 下载cbt_web压缩包
        ![image](http://img.localhostes.com/images/2019/12/23/download.png)
        
    - 把压缩包上传到服务器`/home/ttx/app/nginx/webapps/ofs1_5/`目录下
    
        ```
        cd /home/ttx/app/nginx/webapps/
        # 创建ofs1_5文件夹
        sudo mkdir ofs1_5
        
        # 执行上传操作 ... ...
        
        # 解压cbt-web
        sudo tar -zxvf cbt-web-2.5.12-20190718.063712-5.tar.gz
        ```

- 测试并启动`nginx`
    
    ```
     cd /home/ttx/app/nginx/sbin
     #测试nginx配置文件
     sudo ./nginx -t
     #启动　nginx
     sudo ./nginx
    ```
    
    >   Nginx配置没问题会有以下输出
    
    ![image](http://img.localhostes.com/images/2019/12/23/93ZTJ882G88VANDT5G.png)
    
    >   **注意：** 使用以下命令重启Nginx服务
    
    ```
    sudo ./nginx -s reload
    ```
    
- 检查是否启动成功
    - 在浏览器输入 `http://服务器地址:30001/`
    - 有以下展示就说明Nginx已经配置成功了
        ![image](http://img.localhostes.com/images/2019/12/23/XKCNLIN1M49SFEC3.png)

### 准备程序环境
>   准备程序的部署目录和运行目录(下述以`ofs1_5`、`license`演示)

```
cd /home/ttx/app/tomcat-deploy
python install_app.py license 30008
# 比如内存限制为最低1g，最高4g
python install_app.py ofs1_5 9005 1g 4g
```
    
>   注意：如果服务器上没有python命令，而只有python3，则执行以下命令

```
cd /home/ttx/app/tomcat-deploy
python3 install_app.py license 30008
# 比如内存限制为最低1g，最高4g
python3 install_app.py ofs1_5 9005 1g 4g
```


### 安装license服务
- 上传license.zip文件到/home/ttx/app/license/upload 目录
- 执行命令
    ```
    sudo chmod +x /home/ttx/app/jre/lib/amd64/*
    ```
- 创建执行脚本
    ```
    cd /home/ttx/app/license
    sudo echo "export LD_LIBRARY_PATH=/home/ttx/app/jre/lib/amd64 && nohup /home/ttx/app/jre/bin/java -jar /home/ttx/app/license/upload/license.zip &" > run.sh
    sudo chmod +x run.sh
    ```
- 启动license服务
    ```
    sh ./run.sh
    ```
- 访问license页面：`http://服务器地址:30008`,点击注册文件进行下载
    ![license.png](http://img.localhostes.com/images/2019/12/24/license.png)
- 把下载的cluster-blank.lic给项目经理，进行授权
- 得到授权者提供的正式license文件后，将文件（通常为lic文件(`ttx.lic`)）放在licenses目录下
    ```
    # 重启license服务
    ps -ef | grep license
    ```
    >   通常输出如下：
    
    ```
    ttx       9354     1  1 20:27 pts/1    00:00:15 /home/ttx/app/jre/bin/java -jar /home/ttx/app/license/upload/license.zip
    ttx       9632  1277  0 20:42 pts/0    00:00:00 grep --color=auto java
    ```
    >   找到含有license.zip的进程，杀掉进程；比如上例中，进程ID为9354
    
    ```
    # 将9354替换为实际的进程ID
    sudo kill 9354
    cd /home/ttx/app/license
    ./run.sh
    ```
- 访问`http://服务器地址:30008`，正常情况应显示服务器的license信息


### Jenkins构建
- 进入`Jenkins`配置页面: [http://ci.ittx.com.cn](http://ci.ittx.com.cn) 进入对应文件夹 新建项目目录(例如我的项目是`xxx`)
![mainhtml.png](http://img.localhostes.com/images/2019/12/24/mainhtml.png)
![dirs.png](http://img.localhostes.com/images/2019/12/24/dirs.png)
![jenkinsdir.png](http://img.localhostes.com/images/2019/12/24/jenkinsdir.png)

>   最后save保存就行

- 新建一个项目，这里可以直接复制已有的项目，先找到对应项目的路径。 比如`ofs1_5`的路径是这个`xoms/OFS-1.5/ofs-1.5`

>   新建项目，粘贴复制好的 `xoms/OFS-1.5/ofs-1.5` 
    
![jenkinsitem.png](http://img.localhostes.com/images/2019/12/24/jenkinsitem.png)
![jenkinsitem.png](http://img.localhostes.com/images/2019/12/23/jenkinscopy.png)

>   根据项目进行修改`参与人`以及对应的`customerID`,然后`save`即可

![jenkinsitem.png](http://img.localhostes.com/images/2019/12/23/jenkinsedit1.png)
![jenkinsitem.png](http://img.localhostes.com/images/2019/12/23/jenkinsedit2.png)
![jenkinsitem.png](http://img.localhostes.com/images/2019/12/23/jenkinsedit3.png)

- 打包项目，获取对应的`下载脚本`

![jenkinsitem.png](http://img.localhostes.com/images/2019/12/23/jenkinsscript1.png)
![jenkinsitem.png](http://img.localhostes.com/images/2019/12/23/jenkinsscript2.png)
![jenkinsitem.png](http://img.localhostes.com/images/2019/12/23/jenkinsscript3.png)

- 上传刚下载的`update-xxx-xofs-test.sh`到`/home/ttx/app/tomcat-deploy`
- 配置阿里`oss`（可选：具体看项目是否支持）
    - oss地址：https://ttx-download.oss-cn-hangzhou.aliyuncs.com/projects
    - 修改 `update-xxx-xofs-test.sh` url `(https://release.cybertrans.ittx.com.cn)`为`oss地址`
    ![jenkinsitem.png](http://img.localhostes.com/images/2019/12/23/oss.png)


### ofs配置
- 执行数据库初始化脚本
- 上传`application.yml、application-xxx.yml` 到 `/home/ttx/app/ofs/conf/default`目录下
    - 注意: `xxx`的取值为：
        - 正式服务器： `prod`
        - 测试服务器： `test`
- 根据项目情况配置对应的`mysql、redis、mongodb、rabbitmq`链接
- 拉取`Jenkins`打好的包，编译并启动
    ```
    cd /home/ttx/app/tomcat-deploy
    sh update-xxx-xofs-test.sh
    ```
- 启动完成之后检查日志输出是否正常, `ctrl + c`退出查看模式
    ```
    tail -fn 100 /home/ttx/app/ofs1_5/catalina/9005/logs/catalina.out
    ```
- 启动成功，访问`http://服务器地址:30001`进行访问页面

![jenkinsitem.png](http://img.localhostes.com/images/2019/12/23/main.png)
![jenkinsitem.png](http://img.localhostes.com/images/2019/12/23/ofs.png)


- 启动、停止、编译、重启命令
    ```
    ./app_start.sh ofs1_5
    ./app.stop.sh ofs1_5
    ./app_deploy.sh ofs1_5
    ./app_restart.sh ofs1_5
    ```


### 参考信息

#### 系统目录
```
|-/
  |- home
    |- ttx
      |- installer
        |- images
        |- compose
      |- app
        |- redis
        |- mongo
        |- mysql
        |- rabbitMQ
        |- tomcat-deploy
        |- nginx
        |- license
        |- ofs1_5
        |- scheduler
        |- edi-qimen
        |- edi-taobao
```

#### 端口分配
服务 | 端口 | 备注
:----:|:----:|:----:
nginx | 80 | ofs 端口可转80端口
mysql | 33306 | 当mysql为通天晓安装时，必须修改端口号
mongo | 37017 | 生产服务器必须设定密码
rabbitMQ | 35672 | mq连接端口
rabbitMQ-management | 35673 | 消息队列管理可视化
redis | 36379 | 生产服务器必须设定密码
ofs | 9005 | ofs内部端口
ofs | 30001 | ofs外部端口
scheduler | 30001 | 使用到计划任务组件时使用
license | 30008 | 仅当授权服务部署在客户服务器时使用
edi | 30012-30020 | 多个端口为客户部署多套edi保留