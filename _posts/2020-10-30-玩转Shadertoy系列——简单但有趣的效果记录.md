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

------

一个圆

![](/images/posts/Shadertoy/shadertoy_simple2.png)

```c
/**
 * @author jonobr1 / http://jonobr1.com/
 */

/**
 * Convert r, g, b to normalized vec3
 */
vec3 rgb(float r, float g, float b) {
	return vec3(r / 255.0, g / 255.0, b / 255.0);
}

/**
 * Draw a circle at vec2 `pos` with radius `rad` and
 * color `color`.
 */
vec4 circle(vec2 uv, vec2 pos, float rad, vec3 color) {
	float d = length(pos - uv) - rad;
	float t = clamp(d, 0.0, 1.0); // 圆内返回0，圆外返回1
	return vec4(color, 1.0 - t);
}

void mainImage( out vec4 fragColor, in vec2 fragCoord ) {

	vec2 uv = fragCoord.xy;
	vec2 center = iResolution.xy * 0.5;
	float radius = 0.25 * iResolution.y;

    // Background layer
	vec4 layer1 = vec4(rgb(210.0, 222.0, 228.0), 1.0);
	
	// Circle
	vec3 red = rgb(225.0, 95.0, 60.0);
	vec4 layer2 = circle(uv, center, radius, red);
	
	// Blend the two
	fragColor = mix(layer1, layer2, layer2.a);

}
```

