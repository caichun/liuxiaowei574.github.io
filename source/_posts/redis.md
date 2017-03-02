---
title: Redis主从集群及故障自动切换
date: 2017-3-2 21:38:02
tags: 
- Redis
categories: Redis
---

Redis官网：https://redis.io/

## 一、架构

操作系统：Debian 7

Master：127.0.0.1，端口：6379

Slave1：:127.0.0.1，端口：6378

Slave2：:127.0.0.1，端口：6377

Sentinel1：127.0.0.1，端口：26379

Sentinel2：127.0.0.1，端口：26378

Sentinel3：127.0.0.1，端口：26377


## 二、主从配置
### 1、下载redis压缩包，并解压，我这里下载的是2.8.24版本：
```bash
tar -zxvf redis-2.8.24.tar.gz
```

<!-- more -->

### 2、进入解压目录，修改配置文件`redis.conf`：
```bash
cd redis-2.8.24
vi redis.conf
```
### 3、其他保持默认，为客户端连接增加密码为`fish`：
```bash
masterauth fish
requirepass fish
```
至此，master节点配置完毕。下面开始配置slave节点。

### 4、拷贝一份redis-2.8.24目录的副本，命名为`redis-2.8.24-slave1`：
```bash
cp  redis-2.8.24  redis-2.8.24-slave1
```
### 5、进入上面目录，修改配置文件`redis.conf`：
```bash
cd redis-2.8.24-slave1
vi redis.conf
```
### 6、分别进行以下修改：
```bash
#端口
port  6378

#设置连接主节点的密码：
masterauth fish

#修改路径
dir "/home/fish/redis/redis-2.8.24-slave1/src"

#配置为slave
slaveof 127.0.0.1 6379
```
### 7、同样的步骤再复制一份`redis-2.8.24-slave2`目录，并修改配置文件：
```bash
port  6377
masterauth fish
dir "/home/fish/redis/redis-2.8.24-slave2/src"
slaveof 127.0.0.1 6379
```
至此，两个slave也配置完毕。

### 8、测试集群配置

#### 1）启动master节点：
```bash
cd  redis-2.8.24
src/redis-server  redis.conf
```
应该可以看到如下界面：

![](/images/redis/01.png) 

#### 2）启动slave1节点
新开一个终端并执行：
```bash
cd  redis-2.8.24-slave1
src/redis-server  redis.conf
```
可以看到如下界面：

![](/images/redis/02.png) 


表示已经开始和master节点通信。
#### 3）新开一个终端，同样方式启动slave2。
#### 4）客户端连接master节点，查看信息
```bash
cd  redis-2.8.24
src/redis-cli 
127.0.0.1:6379> auth fish
127.0.0.1:6379> info replication
```
可以看到当前连接的是master节点，并且有两个slave节点：

![](/images/redis/03.png) 

存储一个string到redis：
```bash
127.0.0.1:6379> set name zhangsan
```
可以观察到两个slave节点也将信息同步到自己这边：

![](/images/redis/04.png) 

#### 5）客户端连接slave节点，查看信息
退出客户端连接：
```bash
127.0.0.1:6379> quit
```
再重新连接一个slave节点：
```bash
src/redis-cli -p 6378
127.0.0.1:6378> auth fish
127.0.0.1:6378> info replication
```
可以看到当前连接信息和master的信息：

![](/images/redis/05.png) 

从slave节点上获取存入master节点的值：
```bash
127.0.0.1:6378> get name
"zhangsan"
```

至此，主从集群配置完毕。但是如果master节点发生故障，尚不能自动切换到slave节点。下面使用redis sentinel来配置监控集群，便于自动切换。


## 三、Sentinel集群配置

### 1、Redis Sentinel
Sentinel(哨兵)是用于监控redis集群中Master状态的工具，其已经被集成在redis2.4+的版本中
Sentinel作用：
1)：Master状态检测 

2)：如果Master异常，则会进行Master-Slave切换，将其中一个Slave作为Master，将之前的Master作为Slave

3)：Master-Slave切换后，master_redis.conf、slave_redis.conf和sentinel.conf的内容都会发生改变，即master_redis.conf中会多一行slaveof的配置，sentinel.conf的监控目标会随之调换。
### 2、进入redis目录，修改sentinel配置文件
```bash
cd  redis-2.8.24
cp  sentinel.conf  sentinel_bak.conf
```
清空原文件内容
```bash
>sentinel_bak.conf
```
存入内容：
```bash
port 26379
#master
sentinel monitor master 127.0.0.1 6379 3
sentinel down-after-milliseconds master 5000
sentinel auth-pass master fish
sentinel config-epoch master 8
sentinel leader-epoch master 9
```

其中标红的`3`：代表有3个sentinel实例
### 3、复制两份`sentinel.conf`，分别命名为`sentinel23678.conf`，`sentinel23677.conf`：
```bash
cp  sentinel.conf  sentinel23678.conf
cp  sentinel.conf  sentinel23677.conf
```
并分别修改端口为`23678`、`23677`
### 4、运行redis-sentinel
开三个终端，分别执行
```bash
src/redis-sentinel  sentinel.conf  --sentinel
src/redis-sentinel sentinel26378.conf --sentinel
src/redis-sentinel sentinel26377.conf --sentinel
```
可以看到启动信息：

![](/images/redis/06.png) 

### 5、通过sentinel查看主从信息：
```bash
cd  redis-2.8.24
src/redis-cli  -p  26379
127.0.0.1:26379> info
```

![](/images/redis/07.png) 


## 四、测试故障切换配置

### 1、停掉master节点
在master节点的运行终端中，按下Ctrl+C。
### 2、观察sentinel的输出信息，会看到已经检测到变化，slave1节点（6378端口）自动切换为master：

![](/images/redis/08.png) 

### 3、观察slave1节点的输出信息，说明它已经切换到master状态：

![](/images/redis/09.png) 


至此，Redis的主从集群、故障自动切换机制已经完成。

