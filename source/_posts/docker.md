---
title: Windows版Docker为Centos镜像安装ssh、vsftpd服务
date: 2017-5-23 00:05:02
tags: 
- Docker
categories: Docker
---



## 一、安装docker
我安装的是Windows版的DockerToolbox-17.04.0-ce，这个集成了一些常用的工具包。
下载地址：[DockerToolbox](https://www.docker.com/products/docker-toolbox)

安装过程略。
安装完，双击桌面的“Docker Quickstart Terminal”启动docker。首次启动时会比较慢，因为要下载默认镜像，耐心等待就好了。我看了下自己的，是下载到：C:\Users\Administrator\.docker\machine\machines\default\boot2docker.iso，大概40M左右。

docker启动后会分配一个ip地址：`192.168.99.100`，这样的话，就可以通过SecureCRT等工具来连接docker了。


<!-- more -->


## 二、下载centos镜像
搜索

`docker search centos`

下载

`docker pull centos`

查看本地所有镜像：

`docker images`


## 三、为centos镜像安装常用服务

### 1、打开镜像，进入容器

`docker run -itd 40d8d9466567`

40d8d9466567为镜像Id，也可以输入镜像名，如centos

-d：表示运行于后台，-i表示打开标准输入用于控制台交互，-t表示分配一个tty设备，可以支持终端登录。
直接docker run imageId的话，会直接在前台打开并且立刻关闭。这里我们要打开镜像，并进行一些操作。

下面的2-7、9步操作，都是在打开的容器中进行的，即命令之前的提示符为：`[root@f0a02b473067 /]#`，而非docker。

### 2、设置root口令，以便后面ssh和ftp连接：

输入新口令 `passwd`

### 3、安装less

`yum install -y less`

-y代表交互时自动选择y，全自动。

### 4、安装ifconfig：

`yum search ifconfig`

`yum install -y net-tools.x86_64`

试试ifconfig命令可不可用

### 5、安装sshd：

`yum install -y openssh-server`

启动sshd：

`/usr/sbin/sshd -D`

设置sshd服务自动启动：

`chkconfig sshd on`

停止服务：`Ctrl+C`

启动sshd如果报错：
```bash
Could not load host key: /etc/ssh/ssh_host_rsa_key
Could not load host key: /etc/ssh/ssh_host_ecdsa_key
Could not load host key: /etc/ssh/ssh_host_ed25519_key
```
则运行：
```bash
ssh-keygen -t ecdsa -f /etc/ssh/ssh_host_rsa_key  
ssh-keygen -t ecdsa -f  /etc/ssh/ssh_host_ecdsa_key 
ssh-keygen -t ecdsa -f   /etc/ssh/ssh_host_ed25519_key
```

### 6、安装vsftpd服务：

1) 查看是否已经安装vsftpd

`rpm -qa | grep vsftpd`

如果没有，就安装，并设置开机启动

`yum -y install vsftpd`

`chkconfig vsftpd on`

2) 配置vsftpd：

`vi /etc/vsftpd/vsftpd.conf`

```bat
#chroot_list_enable=YES
# (default follows)
#chroot_list_file=/etc/vsftpd.chroot_list
改为
chroot_list_enable=YES
# (default follows)
chroot_list_file=/etc/vsftpd/chroot_list

增加：
local_root=/ftp
allow_writeable_chroot=YES
```
保存。

3) 配置ftp连接用户

`vi /etc/vsftpd/chroot_list`

每一行为一个允许连接的用户名，这里输入root，并保存


4) 创建/ftp作为ftp的根目录：

`mkdir /ftp`

`chmod -R a+w /ftp`


5) 设置root用户可以登录：

`vi /etc/vsftpd/user_list`

`vi /etc/vsftpd/ftpusers`

修改上述两个文件，把root那一行删除或者注释掉。

**注意：使用ftp客户端连接容器时，要修改传输模式为： 主动传输**

### 7、安装service服务：

docker提供的centos镜像并没有service指令，即指令文件实际不存在：/sbin/service

查看有无安装相关软件：

`yum list | grep initscripts`

查找软件：

`rpm -qa | grep initscripts`

安装：

`yum install initscripts`

查看指令文件：

`ll /sbin/service`

### 8、保存为新镜像

注意，上述任何对镜像的改动，如果想永久保留，一定要进行保存，否则镜像一旦关闭，下次启动时又回到原来的配置。前面的改动全都丢了。

在docker环境或新打开一个SecureCRT连接docker，查看正在运行的镜像：

`docker ps`

保存为新镜像，命令：docker commit  容器Id（非镜像Id）  镜像名

`docker commit 75955f546d00 mycentos-ssh-vsftpd`

### 9、退出容器

## 四、运行新镜像并测试

上面为新镜像安装了ssh、vsftpd服务，但要想在容器外部连接并登录，要在docker运行该镜像的时候，进行端口映射。

另外，还要启动dbus-daemon服务，以保证容器中的service指令可以正常使用。具体可以参考：
[http://welcomeweb.blog.51cto.com/10487763/1735251](http://welcomeweb.blog.51cto.com/10487763/1735251)

最终运行镜像的指令是：

`docker run --privileged -itd -e "container=docker"  -v /sys/fs/cgroup:/sys/fs/cgroup -p 50001:22 -p 50000:21 mycentos-ssh-vsftpd  /usr/sbin/init`

上面将容器内的ssh服务端口22，映射到docker的50001，ftp端口21映射到docker的50000。mycentos-ssh-vsftpd是我的镜像名。

下面就可以通过SecureCRT和FileZilla（修改传输模式为：主动传输）进行连接了。

SecureCRT：

![](/images/docker/01.png) 
![](/images/docker/02.png) 

FileZilla：

![](/images/docker/03.png) 
![](/images/docker/04.png) 

**还是那句话，对容器进行的任何修改，如想保存，都必须在docker里进行保存。否则下次启动都会丢失。包括ftp连接后上传进去的文件。**

