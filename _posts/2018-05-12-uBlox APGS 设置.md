---
layout: post
title: uBlox APGS 设置
date: 2018-5-12
categories: blog
tags: [AGPS]
description: 
---

**server**: offline-live1.services.u-blox.com
**port**: 80
**token**: {# Gaty1NDYLkGQmVNvva2FRQ #}
**HTTP param**: token={# Gaty1NDYLkGQmVNvva2FRQ #};gnss=gps;period=2;resolution=1-->
**HTTP Request**:
```
GET /GetOfflineData.ashx?token={# Gaty1NDYLkGQmVNvva2FRQ #};gnss=gps;period=2;resolution=1 HTTP/1.1
Host: offline-live1.services.u-blox.com
Accept: */*
Connection: Keep-Alive
```

