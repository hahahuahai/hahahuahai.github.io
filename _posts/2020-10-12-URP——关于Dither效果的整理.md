---
layout:     post
title:      URP——关于Dither效果的整理
subtitle:   
date:       2020-10-12
author:     huahai
catalog: true
tags:
    - Unity3d
---

# 前言

知乎上的一个[问题](https://www.zhihu.com/question/383313632/answer/1114694813)，说明了Dither在游戏里的应用。

# 效果

![](/images/posts/Unity3d/dither1.gif)

# Dither Pattern

Dither效果的核心就是Dither Pattern。这个Dither Pattern就是一张平铺在屏幕空间上的透明度阈值纹理。每一个屏幕空间上的片元在这个纹理上获取对应的透明度阈值，如果片元的透明度小于这个阈值，则遗弃；如果片元的透明度大于等于这个阈值，则保留。如此就得到了dither的效果。

shadergraph里面有专门的[Dither节点](https://docs.unity3d.com/Packages/com.unity.shadergraph@8.2/manual/Dither-Node.html)，里面有具体的Dither Pattern，采用的是[Ordered dithering](https://en.wikipedia.org/wiki/Ordered_dithering)。

# 扩展阅读资料

- [Transparency Dithering in Shader Graph and URP](https://danielilett.com/2020-04-19-tut5-5-urp-dither-transparency/)
- [从仿色图案（Dither Pattern）到半调（Halftone）](https://zhuanlan.zhihu.com/p/60325181)
- [Unity3D后期Shader特效-马赛克6-仿色图案Dither Pattern（灰度四舍五入|颜色相加计算）](https://zhuanlan.zhihu.com/p/70292743)
- ASE示例build-in示例里面有一个关于Dither的



