---
layout:     post
title:      使用LoxodonFramework踩过的一些坑
subtitle:   
date:       2020-03-15
author:     huahai
catalog: true
tags:
    - LoxodonFramework
---





### 一些需要注意的地方，记录一下

##### 1.UI的prefab中需要添加Canvas Group组件，不然UIprefab会创建在Hierarchy的根节点，而不是自己想要的地方。具体原因还没看源码，应该跟内部实现有关。

