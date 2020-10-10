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

# 植被的渲染分析

![](/images/posts/Unity3d/boatattack3.gif)

- 植被的动画和着色，这里参考的是《GPU Gems 3》里面的第16章Crysis 中植被的程序化动画和着色（Vegetation Procedural Animation and Shading in Crysis）。主弯曲还是很容易把书上的代码和这里的代码对应起来的，细节弯曲好像不太一样，需要再多看看，可以采用倒推的方式看，比如从SmoothTriangleWave这个函数倒推。
- 需要注意的是：1.书上的Y和Z坐标，与这里的Z和Y坐标对应。2.计算vWaves时这里没有像书中用frac。
- 网上的一篇可以参考的[文章](https://mtnphil.wordpress.com/2011/10/18/wind-animations-for-vegetation/)和对应的[视频](https://www.youtube.com/watch?v=Mb5QVoqFkKo)。这篇文章相当于在详细解释GPU GEMS里面的动画部分的内容，说的比较详细。
- 知乎上的一个[问题](https://www.zhihu.com/question/50202986?utm_source=cowlevel)也提到了这个。
- 植被也做了LOD，距离近的是非billboard版本，距离远的是billboard版本。用的都是同一个shader。
- （动画部分细节，以后再看）（着色部分似乎也是根据GPU Gems 3那文章上做的，具体以后再详细看）

# 房屋的渲染分析

![](/images/posts/Unity3d/boatattack4.png)

- 这里的房屋渲染有个特别之处：会根据时间是白天还是晚上进行调整。晚上会让房屋的窗户亮起来，模拟房屋开灯的效果。挺有意思的。
- 窗户亮的效果是通过把全局的时间（DayNightController.cs里面的time，time发生变化的时候就会实时同步到shaderd的_NightFade参数）作为参数引入，然后通过Emission自发光来做。

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



