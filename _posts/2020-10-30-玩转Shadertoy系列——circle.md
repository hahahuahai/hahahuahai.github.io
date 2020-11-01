---
layout:     post
title:      玩转Shadertoy系列——太阳
subtitle:   
date:       2020-10-30
author:     huahai
catalog: true
tags:
    - 计算机图形学
---

# 前言

从这篇开始，不定期记录一下在Shadertoy上做的一些效果。

# 效果

太阳

![](/images/posts/Shadertoy/shadertoy_sun1.png)

# 代码

```c
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    // Normalized pixel coordinates (from 0 to 1)
    vec2 uv = fragCoord /iResolution.xy;
    
    uv -= .5; // 将坐标原点移到屏幕中央，屏幕范围（-0.5,0.5）
    uv.x *= iResolution.x / iResolution.y; // 将宽度映射到高度上，从而让椭圆变成正圆。
    
    
    float d = length(uv);
    float r = 0.3;
    
    float c = smoothstep(r-0.1, r, d);
    c = 1. - c;

    // Output to screen
    fragColor = vec4(c, c, 0.0, 1.0);
}
```

