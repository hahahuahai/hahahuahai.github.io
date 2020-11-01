---
layout:     post
title:      玩转Shadertoy系列——简单但有趣的效果记录
subtitle:   
date:       2020-10-30
author:     huahai
catalog: true
tags:
    - 计算机图形学
---

# 前言

记录一下平时玩的一些简单效果。

# 效果和代码

4个色块

![](/images/posts/Shadertoy/shadertoy_simple1.png)

```c
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    // Normalized pixel coordinates (from 0 to 1)
    vec2 uv = fragCoord.xy;
    
    vec2 center = iResolution.xy * 0.5;
    
    center = uv - center;

    // Output to screen
    fragColor = vec4(center, 0.0,1.0);
}
```

