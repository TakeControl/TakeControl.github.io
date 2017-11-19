---
layout:     post
title:      "浏览器缓存机制"

date:       2017-11-18 22:00:00
author:     "ShiYu"
catalog: true
tags:
    - Web
---
# 浏览器缓存机制

浏览器在发起一次请求时，首先可能会使用浏览器端的缓存数据，其次即使请求发到服务端，也有可能访问到的是缓存服务器的数据，浏览器缓存可以通过HTTP来控制。

## Cache-Control/Pragma

Cache-Control用来指定所有缓存机制在请求/响应链中必须服从的指令，不仅可以控制浏览器，还可以控制和HTTP相关的缓存或代理服务器，其可选值如下：

可选值|说明
---|---
Public|所选内容都讲被缓存，在响应头中设置
Private|内容只缓存到私有缓存中，在响应头中设置
no-cache|所有内容都不会被缓存，在请求头和响应头中设置
no-store|所有内容都不会被缓存到缓存或Internet临时文件中，在响应头中设置
must-revalidation/proxy-revalidation|如果缓存失效，请求必须发送到服务器/代理以进行重新验证，在请求头中设置
max-age|缓存的内容必须在xxx秒后失效，这个选项只在HTTP1.1中可用，和Last-Modified一起使用时优先级较高，在响应头中设置


Cache-Control优先级较高，它和其它请求字段如（Expires）同时出现时，Cache-Control会覆盖其它字段。

Pragma字段作用和Cache-Control类似，最常用的就是Pragma:no-cache

## Expires


Expires使用格式是Expires:日期时间，后面指定一个时间值，当超过该时间时，缓存失效，即浏览器在发出请求之前检查这个页面是否过期，过期了就会重新向服务器发起请求

## Last-Modified/Etag

Last-Modified字段用于表示一个服务器上资源的最后修改时间，资源可以是静态或动态的，通过这个最后修改时间可以判断当前请求的资源是否是最新的。

一般服务器在响应头中返回一个Last-Modified字段，告诉浏览器这个页面最后修改时间，如Last-Modified:Sat,25 Feb 2012 12:55:03 GMT，浏览器再次请求时在请求头中增加一个If-Modieied-Since:Sat,25 Feb 2012 12:55:03 GMT字段，询问当前缓存页面是否是最新的，如果是最新的就返回304状态码，告诉浏览器是最新的 ，服务器也不会传输数据。

与Last-Modified有类似功能的还有一个Etag字段，这个字段的作用是让服务器给每个页面分配一个唯一的编号，然后通过这个编号来区分当前这个页面是否是最新的。这种方式比使用Last-Modified更加灵活，但是当web服务器有多台时就比较难以处理，因为每个Web服务器都要记住网站上所有资源，否则浏览器返回这个编号就没有意义了。