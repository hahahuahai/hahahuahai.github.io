---
layout:     post
title:      URP-BoatAttack项目研究
subtitle:   
date:       2020-10-05
author:     huahai
catalog: true
tags:
    - Unity3d
---

# 前言

[BoatAttack 示例地址](https://github.com/Unity-Technologies/BoatAttack)，这个项目是unity官方用来展示URP功能的开源项目。



# 山崖的渲染分析

![](/images/posts/Unity3d/boatattack2.gif)

- 山崖上方的草（Grass Mask）：用一个Grass Height Blend属性，来控制长草的最低高度和最大高度，从而对草生长的高度进行了限制；然后前面的高度结果与岩石法线在世界空间中的Y坐标相乘，然后引入一个Grass Angle属性，来控制长草的最小角度和最大角度；这样就能得到需要长草的范围和不需要长草的范围。（草的覆盖跟雪的覆盖差不多？）

# 云的渲染分析

![](/images/posts/Unity3d/boatattack1.png)

模板使用的Unlit Graph。

- Vertex Normal：法线部分看的不太明白。





# 扩展阅读资料

- [Achieve beautiful, scalable, and performant graphics with the Universal Render Pipeline](https://blogs.unity3d.com/2020/02/10/achieve-beautiful-scalable-and-performant-graphics-with-the-universal-render-pipeline/) 和[中文版](https://www.gameres.com/862776.html)
- [Boat Attack 项目海水技术解析](https://zhuanlan.zhihu.com/p/127116312)
- [视频讲解](https://unity.cn/projects/5f55e2c9edbc2a001f5cdc92?app=true)
- [Unity《Boat Attack》Demo幕后揭秘（内附源码下载；上）](https://unity.cn/projects/unity-boat-attack-demomu-hou-jie-mi-nei-fu-yuan-ma-xia-zai)
- [Unity《Boat Attack》Demo幕后揭秘（下）](https://unity.cn/projects/unity-boat-attack-demomu-hou-jie-mi-xia)



