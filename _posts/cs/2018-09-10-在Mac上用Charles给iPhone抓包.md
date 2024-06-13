---
title:      "在Mac上用Charles给iPhone抓包"
date:       2018-09-10
tags:       ["抓包", "计算机网络"]
category:   Computer Science
math:       true
---

在日常的编程中，很多的网络数据都能在控制台里面看到，但是APP里面web的网络请求就需要抓包了，这里用的工具就是Charles。

网上有一篇比较出名的文章，是唐巧的 [Charles 从入门到精通](https://blog.devtang.com/2015/11/13/charles-introduction/)，可以说比较详细了，但是在最近的使用中，发现这个教程似乎行不通，所以写下这篇文章记录一下我的方法。

## 安装Charles

去官网下载Charles，安装。

## 给Charles打开代理

打开Charles,点击Proxy -> 点击Proxy Setting，Port填8888，勾上Enable transparent HTTP proxying
找到当前使用的Mac的地址，可以点Charles->Help->Local IP Addres。也可以在terminal中输入ifconfig，en0中inet后面接的就是本机IP

## 给iPhone设置代理

打开iPhone，连接wifi，确保手机和Mac连的是同一个wifi。
点击wifi名后面的蓝色感叹号，在最下面找到“HTTP 代理”，点击进入
选择“手动”，服务器填上面找的IP地址，端口填8888。
退出设置，随便连上任意的网络，Charles会弹出请求连接的确认菜单。选择“Allow”
在手机的浏览器中，输入chls.pro/ssl，会弹出安装证书的请求，输入密码，一直点安装就好了。
打开设置，点击通用->关于本机->证书信任设置，找到当前要作为代理的电脑名，打开信任开关。

此时，已经能截取到HTTP/HTTPS的网络请求了，在Charles中选择Structure视图，点击想查看的网络请求，点击content，就能查看到json数据了。

## 其他

如果换了一部电脑作为代理，要重新安装证书，并且信任。