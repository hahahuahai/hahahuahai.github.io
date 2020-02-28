---
layout:     post
title:      fairygui源码阅读（2）——GList
subtitle:   fairygui
date:       2020-02-28
author:     huahai
catalog: true
tags:
    - fairygui
---





## GList中自己平时用的多的地方探索

#### 1.管理列表内容

###### 文档说明

官方介绍在此：[管理列表内容](https://www.fairygui.com/docs/guide/editor/list.html#管理列表内容)

###### 源码探索

GList内建了对象池，这个对象池对应于GObjectPool类。目前FairyGUI用到对象池的地方只有GList。

这个对象池的初始化，需要传一个UnityEngine.Transform对象。FairyGUI也提供了GList对象池的只读属性itemPool。