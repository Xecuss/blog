---
title: git rebase入门教程
date: 2022-04-22 14:47:20
abbrlink: git-rebase-tutorial 
hide: true
published: false
tags: 
  - git
  - rebase
---
看到有的同事不会用git rebase，于是决定写一篇教程。笔者本身也是从完全不懂到后来慢慢理解，希望这个教程能够帮助到读者理解git rebase。

## 何为git rebase

git rebase，官方翻译为「变基」，即「改变基底」。改变，很好理解，那么关键就是在这个「基」上，什么是「基」呢，看看官方的解释

> 当执行rebase操作时，git会从两个分支的*共同祖先*开始提取待变基分支上的修改，然后将待变基分支指向基分支的最新提交，最后将刚才提取的修改应用到基分支的最新提交的后面。

看起来这个「基」，指的是*两个分支的共同祖先*，我们画一个图来理解一下：