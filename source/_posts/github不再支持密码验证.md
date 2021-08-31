---
title: github自2021年8月13日起不再支持密码验证解决方案
date: 2021-08-20 15:42:43
categories: git
tags: github

---



# 前言



前几天正准备往github上提交代码时，直接push失败，而且无法拉取代码，然而之前还是好好的，后来查看git日志后发现报了这个错

![](GitHub-error.png)

这个错误的意思是说自从2021年8月13日起，github便不再支持密码验证了，要使用个人token来进行验证，那么如何来解决这个问题呢



# 解决办法



既然官方说了必须要使用token验证，那么我们就去github官网去申请token，申请的步骤依次如下截图：

1、点击头像里的Settings，选择右侧的Developer settings

<img src="settings.png" style="zoom: 50%;" />  <img src="developer.png" style="zoom:50%;" />

<br/>

2、选择Personal access tokens，可以看到目前还没有申请任何可用的token

<img src="token.png" style="zoom:80%;" />

<br/>

3、点击Generate new token按钮去申请新的token，将生成好的token保存下来

<img src="generate.png" style="zoom:75%;" />

<br/>

<img src="save.png" style="zoom:75%;" />

<br/>

4、下次提交代码时输入对应的token即可，这里我用的是vscode插件
