---
title: 使用Redirector插件解决PASELI充值页无法使用的问题
abbrlink: 14319
date: 2022-09-07 18:13:05
tags: 
  - 音游
  - SDVX
---
## 问题原因
其实是因为充值页的JQuery运行库引用自Google CDN，然而因为众所周知的原因，Google在国内并不能访问。
![](error.jpg)

## 解决方案
1. 理论上可以使用炼金术上网解决这个问题，但在实际操作过程中发现并不好使。
2. 可以使用插件将引用的运行库资源指向国内镜像，本文介绍这个方法。

## 步骤

如果你比较精通计算机，可以前往[插件Github页面](https://github.com/einaregilsson/Redirector)直接参考文档使用，无需阅读本文。

### 安装Redirector插件
前提：需使用Edge/Chrome/Firefox浏览器PC版。笔者推荐使用Edge浏览器，因为微软商店在国内可以直接访问到，并且比较新的PC自带Edge浏览器，可以省去很多麻烦。

1. 请根据你的浏览器点击以下链接：
[Edge浏览器点此链接](https://microsoftedge.microsoft.com/addons/detail/redirector/jdhdjbcalnfbmfdpfggcogaegfcjdcfp)
[Chrome浏览器点此链接](https://chrome.google.com/webstore/detail/redirector/ocgpenflpmgnfapjedencafcfakcekcd)
[FireFox浏览器点此链接](https://addons.mozilla.org/firefox/addon/5064)

2. 点开后的操作都大同小异，以Edge为例，点击「获取」：
![](getAddon1.png)
![](getAddon2.png)
然后插件就会安装到浏览器。

### 配置Redirector插件
1. 首先<a href="redirector.txt" download target="_blank">点此下载</a>配置文件[^1]，把它放在你能找到的位置。如果你的浏览器打开了这个文件而不是下载，请在链接上单击右键，然后选择链接另存为
2. 以Edge为例，Edge安装插件后可能会折叠起来，所以先点开插件列表然后点击插件
![](setAddon1.png)
3. 然后会弹出菜单，点击「Edit Redirects」进行配置
![](setAddon2.png)
4. 点击「import」，然后选择你在第1步下载的配置文件
![](setAddon3.png)
5. 成功导入
![](setAddon4.png)
6. 回到PASELI充值页并刷新，现在应该就可以正常使用了
![](finish.png)

## 参考
[^1]: [使用Redirector插件解决googleapis公共库加载的问题 - 上官飞鸿 - 博客园](https://www.cnblogs.com/jackadam/p/11258463.html)