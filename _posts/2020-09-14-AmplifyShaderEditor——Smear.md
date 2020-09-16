---
layout:     post
title:      AmplifyShaderEditor——Smear
subtitle:   
date:       2020-09-14
author:     huahai
catalog: true
tags:
    - Amplify Shader Editor
---



# 效果图

![](/images/posts/ASE/Smear1.gif)

# 原理

大致的步骤：

1.用模型当前的世界坐标和模型前一个位置的世界坐标，计算出每个顶点运动的方向，并将模型运动方向的背面凸出来。先对两个向量（模型上的点到模型基准点的向量、前一帧模型上的该点到现在模型上的该点的向量）求夹角（dot）。

2.将凸出来的坐标从世界坐标转换到局部坐标，然后对局部坐标的顶点位置进行偏移即可。



具体代码文件：

1.smear.cs：用于设置模型的当前位置和上一帧的位置，然后传给材质。

2.SimpleMoveExample.cs：用于让模型运动起来，运动的范围是在以初始模型位置的正负（3,1,3）的外包矩形内。

（写到一半发现网上有人已经写的很详细了，具体原理可以看第三个参考文章，就不重复造轮子了）



# 参考文章

1. https://www.bruteforce-games.com/post/soft-body-shader-jelly-joy-devblog-2
2. https://github.com/cjacobwade/HelpfulScripts/tree/master/SmearEffect
3. https://zhuanlan.zhihu.com/p/76570702（这篇文章里讲的很详细）