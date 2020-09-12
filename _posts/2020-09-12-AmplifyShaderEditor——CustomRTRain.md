---
layout:     post
title:      AmplifyShaderEditor——CustomRTRain
subtitle:   
date:       2020-09-12
author:     huahai
catalog: true
tags:
    - Amplify Shader Editor
---



# 效果图

![](/images/posts/ASE/CustomRTRain1.gif)



# 原理

这个效果主要是利用了[自定义渲染纹理](https://docs.unity3d.com/cn/2019.4/Manual/class-CustomRenderTexture.html)来实现的，核心的shader是：CustomRTRainUpdate。这个shader主要用于自定义渲染纹理的更新材质上。

具体雨滴涟漪实现方法，暂时先不研究了，等以后有需求的时候再研究。

这个涟漪的实现参考相关链接地址：

https://www.shadertoy.com/view/ldfyzl

http://www.ctrl-alt-test.fr/

http://developer.download.nvidia.com/books/HTML/gpugems/gpugems_ch01.html