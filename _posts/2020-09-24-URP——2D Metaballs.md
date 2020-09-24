---
layout:     post
title:      URP——2D Metaballs
subtitle:   
date:       2020-09-24
author:     huahai
catalog: true
tags:
    - Unity3d
---

主要是对[这个博客](https://danielilett.com/2020-03-28-tut5-2-urp-metaballs/)里面的效果进行分析。效果图如下。

![](/images/posts/Unity3d/Metaballs1.gif)

关于Metaballs的基本介绍可以看[这个](https://zh.wikipedia.org/wiki/%E5%85%83%E7%90%83)。

这个效果主要是通过对ScriptableRenderFeature和ScriptableRenderPass扩展实现的。核心的shader是Metaballs2D.shader，核心原理是：在屏幕空间后处理之后执行这个shader，这个shader主要做的事情是将这么多个球（没有mesh，有rigidbody和collider）在屏幕空间绘制出来，绘制的过程主要有三种情况，第一种是球内，用蓝色填充；第二种是球边缘，用白色填充；第三种是球外，用粉色填充。不过这个过程中需要处理两个球紧挨着的情况，这种情况会去掉球的边缘，将两个球合并为一个整体。

球之所以会动是通过物理引擎控制的，可以通过scene场景中看到，如图。

![](/images/posts/Unity3d/Metaballs2.gif)