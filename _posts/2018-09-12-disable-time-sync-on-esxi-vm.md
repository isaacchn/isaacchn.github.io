---
layout: article
title:  "在ESXi 6.5虚拟机上禁用与宿主机时间同步"
tags: ops
modify_date: 2018-09-12 10:00:00
---
> 虚拟机配置了NTP但是时间仍然不同步？可能是宿主机时间同步机制的问题。

# 0x00 前言  
在ESXi上新建了一台虚拟机，但是开发人员报告说时间总是比当前时间快8个小时。一开始以为是ESXi