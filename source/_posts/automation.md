---
title: automation
date: 2023-02-19 10:30:34
tags:
categories: 
password: panyuro@163.com
---

# 1. automation是干什么的？

自动化项目是从金数据项目独立出来的项目，主要的业务包括给表单设置一个触发的规则和执行的动作，当条件达到的时候。比如有新数据提交，数据更新或者到达了指定的时间，就执行设置的动作，包括推送数据到企业微信，webhook，短信，邮件的发送。

这个项目主要使用到的技术是SpringBoot,mybatis, 数据库用的是mongoDb消息中间件采用的是Rabbitmq，前后端通过g raphql进行的分离。


自动化任务：

- 新填写数据
- 修改数据
- 基于数据中的日期
- 定时触发
- 推送邮件/短信/企业微信/发送webhook

