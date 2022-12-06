---
title: DNS域名解析全过程
toc: true
date: 2022-11-27 16:42:21
tags:
categories: 网络
---

## 目的
将域名地址解析成可以访问的ip地址

<!-- more -->

## 流程
输入网址
  ↓
浏览器缓存
  ↓
本地DNS缓存
  ↓
LDNS(本地域名服务器)
  ↓
Root Server(根域名服务器)
  ↓
返回主域名服务器地址（gTLD Server，国际顶尖域名服务器，如.com .cn .org等）
  ↓
LDNS请求gTLD，返回Name Server域名服务器地址
  ↓
Name Server根据映射关系返回ip地址给LDNS
  ↓
LDNS缓存域名和ip，返回结果给用户
  ↓
用户侧缓存域名和ip
  ↓
解析结束