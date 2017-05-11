---
title: Windows版OpenSSL简单应用
date: 2017-5-12 00:43:02
tags: 
- OpenSSL
categories: OpenSSL
---



## 一、下载

Windows版本下载，要下载Win64OpenSSL-1_0_2k版本。因为1.1.0版本生成证书时有时出现莫名其妙的问题。
下载地址：

[Windows版OpenSSL](http://slproweb.com/products/Win32OpenSSL.html)

<!-- more -->


## 二、生成私钥、证书

查看openssl命令帮助：openssl command /?

如：openssl genrsa /?

### 1、生成私钥：

#### 1）带密码和密码加密算法。建议用2048位密钥，少于此可能会不安全或很快将不安全。

```bash
openssl genrsa -des3 -out D:/openssl/privPwd_key.pem 2048

password:yourpwd
```

这个命令会生成一个2048位的密钥，同时有一个des3方法加密的密码。

注意要求输入密码时，密码不能太简单，否则会报错。比如输入test，会报错：

<font color=red>7140:error:0906906F:PEM routines:PEM_ASN1_write_bio:read key:.\crypto\pem\pem_lib.c:334</font>


#### 2）不带密码：

```bash
openssl genrsa -out D:/openssl/privkey.pem 2048
```

### 2、使用私钥，生成公钥：

```bash
openssl rsa -in D:/openssl/privkey.pem -pubout -out D:/openssl/pubkey.pem
```

### 3、生成一个证书请求 

```bash
openssl req -new -key D:/openssl/privkey.pem -out D:/openssl/cert.csr
```

### 4、如果是自己做测试，那么证书的申请机构和颁发机构都是自己。就可以用下面这个命令来生成证书： 

```bash
openssl req -new -x509 -key D:/openssl/privkey.pem -out D:/openssl/cacert.pem -days 365
```

这个命令将用上面生成的密钥privkey.pem生成一个数字证书cacert.pem，有效期365天。

### 5、遇到的错误解决：

1) 执行命令过程中，如果出现找不到openssl.cnf的错误：

Windows: 进入openssl安装目录/bin/cnf/下面，把openssl.cnf 拷贝到C:\Program Files\Common Files\SSL目录下

Linux: 进入/bin/下面，执行命令,把ssl目录下的openssl.cnf 拷贝到bin目录下


## 三、在线RSA加密解密工具

在线RSA公钥加密、解密：

[http://tool.chacuo.net/cryptrsapubkey](http://tool.chacuo.net/cryptrsapubkey)

在线生成RSA公钥、私钥密钥对：

[http://web.chacuo.net/netrsakeypair](http://web.chacuo.net/netrsakeypair)



