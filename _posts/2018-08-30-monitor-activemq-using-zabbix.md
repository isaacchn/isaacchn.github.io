---
layout: article
title:  "使用Zabbix监控ActiveMQ"
tags: ops
modify_date: 2018-08-30 10:00:00
---
> 使用Zabbix监控ActiveMQ的配置。

# 0x00 前言  
项目中使用了ActiveMQ作为消息中间件，如何对其进行行之有效的监控成为了一个问题。项目中对于MySQL、Tomcat、MongoDB都使用Zabbix作为监控平台，ActiveMQ当然也不能例外。