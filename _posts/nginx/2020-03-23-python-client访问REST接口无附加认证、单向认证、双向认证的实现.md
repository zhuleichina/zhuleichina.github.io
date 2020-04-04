---
layout: post
title: python client访问REST接口无附加认证、单向认证、双向认证的实现
category: nginx
tags: [python, 单向认证, 双向认证]
description: python client访问REST接口无附加认证、单向认证、双向认证的实现
---


# python client访问REST接口无附加认证、单向认证、双向认证的实现

## 1. 背景描述

REST登录接口最基本的认证方式为POST消息中附加的用户名、密码认证:
```
datajson  = {
    "username": "xxx",
    "password": "xxx"
}
```

在用户名密码认证的基础上，加上CA证书认证，即为单向认证：
```
ca.cert
```

在用户名密码认证的基础上，加上CA证书和客户端证书，即为双向认证：
```
ca.cert
client.cert
client.key
```

假定需要登录认证的接口为：
```
https://localhost:8080/token
```
要求数据格式为
```
'Content-Type': 'application/json'
```
登录接口返回值为json格式，内容示例如下：
{'username':'xxx', 
'token':'113077031_fad82445c864c85fed08f6ebfbd0253f'}


## 2. 无附加认证
```
import urllib.request
import ssl
import json

def get_token(url, method, datajson):
    data = json.dumps(datajson).encode('utf-8')

    headers = {
        'Content-Type': 'application/json',
    }

    request = urllib.request.Request(url, headers=headers, data=data, method=method)
    response = urllib.request.urlopen(request)
    token_json = json.loads(response.read().decode('utf8'))
    return token_json.get("token")

url = 'https://localhost:8080/token'
method   = 'POST'
datajson  = {
    "username": "xxx",
    "password": "xxx"
}
token = get_token(url, method, datajson)
print (token)
```

## 3. 单向认证
```
import urllib.request
import ssl
import json

def get_token(url, method, datajson, cafile):
    context = ssl.SSLContext(ssl.PROTOCOL_TLSv1_2)
    context.load_verify_locations(cafile)

    data = json.dumps(datajson).encode('utf-8')

    headers = {
        'Content-Type': 'application/json',
    }

    request = urllib.request.Request(url, headers=headers, data=data, method=method)
    response = urllib.request.urlopen(request, context=context)
    token_json = json.loads(response.read().decode('utf8'))
    return token_json.get("token")

cafile   = 'ca.cert'
url = 'https://localhost:8080/token'
method   = 'POST'
datajson  = {
    "username": "xxx",
    "password": "xxx"
}
token = get_token(url, method, datajson, cafile)
print (token)
```

## 4. 双向认证
```
import urllib.request
import ssl
import json

def get_token(url, method, datajson, cafile, certfile, keyfile):
    context = ssl.SSLContext(ssl.PROTOCOL_TLSv1_2)
    context.load_cert_chain(certfile, keyfile)
    context.load_verify_locations(cafile)

    data = json.dumps(datajson).encode('utf-8')

    headers = {
        'Content-Type': 'application/json',
    }

    request = urllib.request.Request(url, headers=headers, data=data, method=method)
    response = urllib.request.urlopen(request, context=context)
    token_json = json.loads(response.read().decode('utf8'))
    return token_json.get("token")

cafile   = 'ca.cert'
certfile = 'client.cert'
keyfile  = 'client.key'
url = 'https://localhost:8080/token'
method   = 'POST'
datajson  = {
    "username": "xxx",
    "password": "xxx"
}
token = get_token(url, method, datajson, cafile, certfile, keyfile)
print (token)
```

## 5. 使用说明
1. 验证时，需要注意让server端开启同样的验证方法，不然会得不到理论结果。
2. 该代码仅为登录接口的示例，对于其他接口可能需要改为"GET"方法，且没有data，请根据需要自行修改
3. 关于单向认证，双向认证的原理，可以阅读参考资料

# 参考资料
1. python关于SSL/TLS认证的实现, https://blog.csdn.net/vip97yigang/article/details/84721027
2. python client访问REST Service实现双向TLS认证, https://www.jianshu.com/p/83360b973331
