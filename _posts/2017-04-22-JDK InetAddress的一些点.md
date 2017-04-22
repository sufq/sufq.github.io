---
layout:     post
title:      "JDK InetAddress的一些点"
date:       2017-04-22
author:     "示水"
catalog:    true
tags:
    - java
    - dns
---


# JDK InetAddress的一些点

## 背景
项目使用c3p0等数据库连接池， 偶尔会报出连接数据库慢，连接超时等。

由于一个域名可以对应多个ip， 所以不能马上定位那台机器有问题，但这边问题可能有网络、网卡等因素(可以tcpdump抓包)。

所以在建连的时候，先解析成ip， 在对应创建连接池`ip —> 连接池`。这样就可以看到连接的对应机器的了。

## 问题跟踪
之前做法：

```
JdbcUrl ---->  DmainResolver  ----->  JdbcTemplate(ip-->对应连接池)
                       |
                       |
                       v
                   InetAddress.getByName
                  固定返回列表的第一个ip
                 默认本地缓存30s, 解析失败的缓存10s
```

由于是直接使用`InetAddress.getByName(host)`， 这边有个坑， 看源码可以发现。

InetAddress的一些点：

1. 用域名拿ip， 对应的接口是`getAllByName(host), getByName(host)`
2. `getByname(host)`的实现是`getAllByName(host)[0]`, 取第一个元素
3. 从DNS服务器拿到的是A记录的列表，并按顺序缓存。 解析成功默认缓存30s，解析失败的默认缓存10s。可通过配置设置缓存时间，但要每台机器都设置。
4. 如果有使用`SecurityManager`， 则缓存的ip列表是永久， 这边要特别注意
5. 会产生并发阻塞问题（未完全确认） 

这边`getByName`在30s内会产生负载问题， 因为返回永远是同一个ip， 因为直接从缓存拿。


## 解决
去掉实时解析域名， 改有池子方式。规避阻塞等问题。

换成如下方式：

```
                          +--------------+
                          |              |
             Domain       |              |
         ---------------> |     IP池     | --------> IP
                          |              |
                          |              |
                          +-------+------+
                                  ^
                                  |
                                  |
                                  |
                        动时刷新IP  |
                       -----------+

          1. 第一次请求的时候，解析出所有ip，放入池子，做hash均衡输出一个ip。
          2. 启动一个后台进程定时刷新ip， 更新ip池
```

