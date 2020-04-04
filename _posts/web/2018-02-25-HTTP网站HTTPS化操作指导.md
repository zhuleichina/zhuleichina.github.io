---
layout: post
title: HTTP网站HTTPS化操作指导
category: 工具
tags: [HTTP , HTTPS]
description: 最开始接触HTTPS的概念是因为非HTTP网站，在谷歌等浏览下直接标注为不安全，经过HTTPS化之后，浏览器会标注为安全。因此想到需要将网站HTTPS化。本文将从HTTPS基本概念介绍开始，依次介绍证书申请，和如何部署证书到nginx服务器上。
---

# HTTP网站HTTPS化操作指导

最开始接触HTTPS的概念是因为非HTTP网站，在谷歌等浏览下直接标注为不安全，经过HTTPS化之后，浏览器会标注为安全。因此想到需要将网站HTTPS化。本文将从HTTPS基本概念介绍开始，依次介绍证书申请，和如何部署证书到nginx服务器上。

本文按如下顺序组织：
1. HTTPS和HTTP的区别介绍
2. SSL证书介绍
3. Let's Encrypt免费DV证书申请
4. Let's Encrypt DV证书nginx服务器部署

## 1 HTTPS和HTTP的区别

超文本传输协议HTTP协议被用于在Web浏览器和网站服务器之间传递信息。HTTP协议以明文方式发送内容，不提供任何方式的数据加密，如果攻击者截取了Web浏览器和网站服务器之间的传输报文，就可以直接读懂其中的信息，因此HTTP协议不适合传输一些敏感信息，比如信用卡号、密码等。

为了解决HTTP协议的这一缺陷，需要使用另一种协议：安全套接字层超文本传输协议HTTPS。为了数据传输的安全，HTTPS在HTTP的基础上加入了SSL协议，SSL依靠证书来验证服务器的身份，并为浏览器和服务器之间的通信加密。
HTTPS和HTTP的区别主要为以下四点：
1. https协议需要到ca申请证书，一般免费证书很少，需要交费。
2. http是超文本传输协议，信息是明文传输，https 则是具有安全性的ssl加密传输协议。
3. http和https使用的是完全不同的连接方式，用的端口也不一样，前者是80，后者是443。
4. http的连接很简单，是无状态的；HTTPS协议是由SSL+HTTP协议构建的可进行加密传输、身份认证的网络协议，比http协议安全。

## 2 SSL证书介绍

SSL 证书按大类一般可分为 DV SSL 、OV SSL 、EV SSL 证书。有的也会叫做域名型、企业型、增强型证书：

1. 域名型 https 证书（DV SSL）：信任等级一般，只需验证网站的真实性便可颁发证书保护网站；
2. 企业型 https 证书（OV SSL）：信任等级强，须要验证企业的身份，审核严格，安全性更高；
3. 增强型 https 证书（EV SSL）：信任等级最高，一般用于银行证券等金融机构，审核严格，安全性最高，同时可以激活绿色网址栏。

对于个人博客，小型企业展示站点来说，申请DV SSL就足够了，更重要的网上有很多供应商提供免费的DV证书，降低网站部署HTTPS的成本。目前市面上比较容易找到的免费证书如下：

1. 腾讯云SSL证书管理（赛门铁克TrustAsia DV SSL证书）
2. 百度云SSL证书服务
3. Let's Encrypt

BAT的SSL免费证书有效期均为一年，每个证书支持单个域名，不支持泛域名。阿里云之前也提供免费1年的DV证书，现在已经找不到申请入口了。

Let's Encrypt免费证书期限均为3个月，证书支持泛域名，但可以无限续期三个月。

此外，还有2个已经被Firefox, Chrome等浏览器弃用的证书厂商，在目前情况下我们应该避免使用：
1. StartSSL免费DV证书
2. 沃通（Wosign）免费DV证书

接下来我们介绍如何使用Let's Encrypt进行证书申请。

## 3 Let's Encrypt DV证书申请

Let's Encrypt官方推荐使用Certbot程序来获取证书，但会相对比较抽象，我们使用浏览器手动获取证书来学习证书校验的过程。（更多获取Let's Encrypt DV证书的方式见参考资料3）

浏览器打开如下地址：
```
https://gethttpsforfree.com/
#Get HTTPS for free!
```

### 3.1 验证账户信息 Account Info

1.输入自己邮箱

2.生成私钥private key
```
#在控制台工作目录下操作：
openssl genrsa 4096 > account.key
```
3.输出公钥public key
```
openssl rsa -in account.key -pubout
```
将屏幕输出信息复制到“Account Public Key”输入框内并验证

### 3.2 证书签名请求 Certificate Signing Request（CSR）

1.生成TLS私钥
```
openssl genrsa 4096 > domain.key
```
2.生成CSR
```
#DNS:www.example.com为需要HTTPS化的网站地址
#不同系统的openssl配置位置不同，需要根据需求替换
#change "/etc/ssl/openssl.cnf" as needed:
#  Debian: /etc/ssl/openssl.cnf
#  RHEL and CentOS: /etc/pki/tls/openssl.cnf
#  Mac OSX: /System/Library/OpenSSL/openssl.cnf

openssl req -new -sha256 -key domain.key -subj "/" \
  -reqexts SAN -config <(cat /etc/ssl/openssl.cnf \
  <(printf "\n[SAN]\nsubjectAltName=DNS:www.example.com"))
```
将屏幕输出信息复制到“Certificate Signing Request”框内并验证

### 3.3 验证私钥签名请求 Sign API Requests

将上面文本框中的信息复制到控制台执行，并将输出结果填写到下面的文本框，全部完成后进行验证。

### 3.4 验证网站所有权 Verify Ownership

1.像上步操作一样，验证私钥签名请求。

2.有两个选项："Option 1 - python server" "Option 2 - file-based"，对于一般的网站我们选择Option 2 - file-based

3.要求在网站的某个公共目录下，返回某些字符串。如对于我的要求
```
Under this url:
http://www.example.com/.well-known/acme-challenge/foPYRXbJcScL91deCJ1_iNQeCyR_tj8iM56R9rJDFnU

Serve this content:
foPYRXbJcScL91deCJ1_iNQeCyR_tj8iM56R9rJDFnU.kuzVPNxeWS-ioFoGisdzWzu9-UJRuh_Xzpbtjkf0A0k
```
我们首先应当确定网站的根目录，新建文件夹
```
mkdir -p .well-known/acme-challenge
```
再新建文件，并将content内容复制保存
```
vim .well-known/acme-challenge/foPYRXbJcScL91deCJ1_iNQeCyR_tj8iM56R9rJDFnU

foPYRXbJcScL91deCJ1_iNQeCyR_tj8iM56R9rJDFnU.kuzVPNxeWS-ioFoGisdzWzu9-UJRuh_Xzpbtjkf0A0k
```

TIPS：不同网站的公共根目录不一样，打开浏览器查看页面代码，查看资源文件所在的根目录一般就是网站的公共目录

4.点击验证按钮，即可验证成功，在下面的“Signed Certificate”和“Intermediate Certificate”两个文本框内生成证书。


## 4 Let's Encrypt DV证书nginx服务器部署

1.将上面“Signed Certificate”和“Intermediate Certificate”内生成的密钥信息先后复制到chained.pem文件中

2.将chained.pem和domain.key复制到文件夹/etc/nginx/(目录可以自己选择)下

3.修改nginx配置文件，增加HTTPS访问

修改前nginx配置文件
```
upstream www.example.com {
    server 192.168.1.11;
    server 192.168.1.12;
}

server
{
    listen 80;
    server_name www.example.com;
    location / { 
        proxy_pass http://www.example.com; 
    }
}
```
proxy_pass配置了负载均衡，所以略复杂，一般如果不使用负载均衡，后面直接配置服务器的地址即可

增加443端口HTTPS配置：
```
server {  
  
    listen 443;  
    server_name www.example.com;
  
    ssl on;  
    ssl_certificate /etc/nginx/chained.pem;
    ssl_certificate_key /etc/nginx/domain.key;
  
    ssl_session_timeout 5m;  
    ssl_protocols  TLSv1 TLSv1.1 TLSv1.2;  
    ssl_ciphers  HIGH:!RC4:!MD5:!aNULL:!eNULL:!NULL:!DH:!EDH:!EXP:+MEDIUM; 
    ssl_prefer_server_ciphers   on;         
  
    location / { 	
        proxy_pass http://www.example.com; 
    }
}
```
重载nginx配置
```
nginx -s reload
```

浏览器输入HTTPS地址，验证HTTPS访问是否正常
```
https://www.example.com
```

4.增加HTTP强制跳转功能

修改nginx原配置：
```
server
{
    listen 80;
    server_name www.example.com;
    location / { 
        proxy_pass http://www.example.com; 
    }
}
```
修改为：
```
server
{
    listen 80;
    server_name www.example.com;
    return  301 https://$server_name$request_uri;
}
```
重载nginx配置
```
nginx -s reload
```
此时访问
```
http://www.example.com
```
浏览器会自动301跳转到HTTPS的地址，配置成功。

对应Apache来说，该网站上也有对应的操作说明。

需要注意的是对于个人免费DV证书来说，只有3个月的有效期，如果使用官网脚本的话是可以加定时任务执行无限延期的，本文不做进一步的讨论。

# 如有疑问，欢迎交流

# 参考资料
1. https，https://baike.baidu.com/item/https/285356?fr=aladdin
2. SSL 证书服务，大家用哪家的？，https://www.zhihu.com/question/19578422
3. ACME Client Implementations，https://letsencrypt.org/docs/client-options/
4. Get HTTPS for free!，https://gethttpsforfree.com/
