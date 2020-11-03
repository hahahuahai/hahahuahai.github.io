---
layout:     post
title:      玩转Shadertoy系列——将矩形空间映射到正方形空间
subtitle:   
date:       2020-11-03
author:     huahai
catalog: true
tags:
    - 计算机图形学
---

# 前言

在shadertoy里编写各种效果的时候，最开始经常会需要把800*450的矩形屏幕空间映射到正方形屏幕空间中。这篇文章记录一下这个过程。

# 效果和代码

矩形屏幕空间的效果

![](\images\posts\Shadertoy\shadertoy_screenspace1.png)

```c
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    // Normalized pixel coordinates (from 0 to 1)
    vec2 uv = fragCoord/iResolution.xy;
    
    // Output to screen
    fragColor = vec4(uv,0.,1.0);
}
```

正方形屏幕空间的效果

![](\images\posts\Shadertoy\shadertoy_screenspace2.png)

```c
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    // Normalized pixel coordinates (from 0 to 1)
    vec2 uv = fragCoord/iResolution.xy;
    uv.x *= iResolution.x / iResolution.y;
    
    // Output to screen
    fragColor = vec4(uv,0.,1.0);
}
```

把uv坐标原点移到屏幕中间

![](\images\posts\Shadertoy\shadertoy_screenspace3.png)

```c
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    // Normalized pixel coordinates (from 0 to 1)
    vec2 uv = fragCoord/iResolution.xy;
    uv -= .5;
    uv.x *= iResolution.x / iResolution.y;
    
    // Output to screen
    fragColor = vec4(uv,0.,1.0);
}
```

