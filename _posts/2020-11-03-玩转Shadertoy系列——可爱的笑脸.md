---
layout:     post
title:      玩转Shadertoy系列——可爱的笑脸
subtitle:   
date:       2020-11-03
author:     huahai
catalog: true
tags:
    - 计算机图形学
---

# 前言

看到一个笑脸，记录一下。

# 用到数学函数中的数学知识

remap函数：将a-b范围的t重新映射到c-d范围。数学公式如下。
$$
\frac{t-a}{b-a}=\frac{x-c}{d-c}
$$

$$
x=\frac{t-a}{b-a}\times\left(d-c\right)+c
$$

remap01函数：将a-b范围的t重新映射到0-1范围。原理同remap函数。

[smoothstep函数](http://docs.gl/sl4/smoothstep)：这个函数作用是用来确定待渲染特征的范围，然后再作为mix函数的第三个参数，与原颜色进行混合。这两个函数经常搭配使用。

# 效果和代码

矩形屏幕空间的效果（代码含备注）

![](\images\posts\Shadertoy\shadertoy_smile1.gif)

无动画的笑脸代码

```c
#define sat(x) clamp(x, 0., 1.)
#define S(a,b,t) smoothstep(a,b,t) 

// 将a-b范围的t重新映射到0-1范围
float remap01(float a, float b, float t)
{
    return sat((t-a)/(b-a));
    
}

// 将a-b范围的t重新映射到c-d范围
float remap(float a, float b, float c, float d, float t)
{
    return sat(((t-a)/(b-a))*(d-c) + c);
    
}

// 用于将uv坐标进行转换至一个自定义矩形中，0——1范围。注意：rect是一个自定义的矩形范围，xy表示矩形的横纵坐标最小的坐标值（一般是左下角），zw表示矩形的横纵坐标最大的坐标值（一般是右上角），而不是矩形的width和height。之所以自定义一个这样的矩形范围，是为了方便各个子模块（眼睛、嘴巴）里面的绘制，因为在这个自定义矩形里坐标范围是0——1的。
vec2 within(vec2 uv, vec4 rect)
{
    return (uv-rect.xy)/(rect.zw - rect.xy);    
}

// 嘴巴渲染
vec4 Mouth(vec2 uv)
{
    uv -= 0.5; // 将uv范围0到1变为-0.5到0.5，从而(0,0)点在自定义矩形的中心
    
 	vec4 col = vec4(.5, .18, .05, 1.); // 口腔底色，暗红色
    
    
    uv.y *= 1.5; // 确定嘴型，控制张开程度。
    uv.y -= uv.x*uv.x*2.; // 确定嘴型，使嘴角上扬。正常情况下，x值不同，y如果相同，则是一条平行于水平坐标轴的线。所以，y = y-x*x，x*x越大，y如果相同，则是一条微笑曲线。*2.是为了控制上扬程度。
    
    float d = length(uv); // 嘴巴的圆心(0,0)
    
    // 牙齿
    float td = length(uv-vec2(0.,.6)); // 牙齿的圆心(0.,.6)
    vec3 toothCol = vec3(1.)*smoothstep(.6,.35, d);// 确定牙齿的上沿，不能超出嘴巴    
    col.rgb = mix(col.rgb, toothCol, smoothstep(.4, .37, td)); // 加入牙齿
    // 舌头
    td = length(uv+vec2(0.,.5));  // 舌头的圆心(0.,-.5)
    col.rgb = mix(col.rgb, vec3(1., .5, .5), smoothstep(.5, .2, td)); // 加入舌头
    
    col.a = smoothstep(.5, .48, d); // 确定整个嘴巴的范围
    
    return col;
    
}



// 眼睛渲染
vec4 Eye(vec2 uv)
{
    uv -= 0.5;  // 将uv范围0到1变为-0.5到0.5，从而(0,0)点在自定义矩形的中心
    
    float d = length(uv);
    
    vec4 irisColor = vec4(.3, .5, 1., 1.); //虹膜的颜色，蓝色
    vec4 col = vec4(1.); // 巩膜的颜色，白色
    col = mix(col, irisColor, smoothstep(.1, .7, d)*.5);
    
    // 眼角的阴影
    col.rgb *= 1. - smoothstep(.45, .5, d)* .5 * sat(-uv.y-uv.x);
    // 眼珠的描边，黑色
    col.rgb = mix(col.rgb, vec3(0.), smoothstep(.3, .28, d));
    // 虹膜
    irisColor.rgb *= 1. + smoothstep(.3, .05, d); // 使虹膜的蓝色由外向内有个渐变的效果    
    col.rgb = mix(col.rgb, irisColor.rgb, smoothstep(.28, .25, d)); //加入虹膜，半径为0.28
    // 瞳孔
    col.rgb = mix(col.rgb, vec3(0.), smoothstep(.16, .14, d)); // 加入瞳孔，半径为0.16，黑色
    
    // 眼球上的高光
    float highlight = smoothstep(.1, .09, length(uv-vec2(-.15, .15)));//大高光，右眼大高光圆心(-0.15,0.15),半径为0.1；左眼对称。    
    highlight += smoothstep(.07, .05, length(uv+vec2(-.08, .08)));//小高光，右眼小高光圆心(0.08,-0.08),半径为0.07；左眼对称。    
    col.rgb = mix(col.rgb, vec3(1.), highlight);// 加入高光，白色
    
    col.a = smoothstep(.5, .48, d); // 设置眼球的alpha通道，将眼球的半径限制在0.5
    
    return col;
    
}


// 头部渲染
vec4 Head(vec2 uv)
{
 	vec4 col = vec4(.9, .65, .1, 1.); //整张脸的皮肤底色，黄色
    
    float d = length(uv); // 整张脸的圆心为(0,0)
    
    col.a = smoothstep(.5, .49, d); //脸的半径为0.5
    // 脸的边缘（边缘暗色+边缘描边）
    //// 边缘暗色
    float edgeshade = remap01(.35, .5, d);      
    edgeshade *= edgeshade; // 让边缘暗色稍微亮一点
    col.rgb *= 1. - edgeshade* .5;
    //// 边缘描边
    col.rgb = mix(col.rgb, vec3(.6, .3, .1), smoothstep(.47, .48, d)); // 边的颜色为暗黄色
    
    // 额头的高光
    float highlight = smoothstep(.41, .405, d);    
    highlight *= remap(.41, .0, .75, .0, uv.y); // 确定额头高光的位置，使得uv.y在小于0的时候，remap返回值是负数，从而使高光下半部消失了，只保留上半部，且由下至上渐变亮。
    highlight *= smoothstep(.18, .19, length(uv-vec2(.21, .08))); // 眼睛最底下的黄色，右边的眼睛以(0.21,0.08)为圆心，左边的眼睛是以(0.21,-0.08)为圆心。    
    col.rgb = mix(col.rgb, vec3(1.), highlight); //额头的高光为白色
    
    //腮cheek    
    d = length(uv - vec2(.25, -.2)); // 右边的腮以(0.25,-0.2)为圆心，左边的腮以(-0.25,-0.2)为圆心    
    float cheek = smoothstep(.2, .01, d) * .4; // 腮的半径为0.2，从边缘向内红色渐变深。*0.4是让腮红整体变淡。    
    cheek *= smoothstep(.18,.17, d); // 这个效果太细微了
    col.rgb = mix(col.rgb, vec3(1., .1, .1), cheek); //腮为红色
    
    return col;
    
}

//整个场景渲染（笑脸+背景）
vec4 smiley(vec2 uv)
{
 	vec4 col = vec4(0.); // 背景
    
    uv.x = abs(uv.x); // 脸（眼睛、腮对称）的左右两边对称绘制
    
    vec4 head = Head(uv); // 头部
    vec4 eye = Eye(within(uv, vec4(.03, -.1, .37, .25))); // 眼睛，右眼自定义矩形左下角坐标(.03, -.1)右上角坐标(.37, .25)，左眼自定义矩形对称分布。
    vec4 mouth = Mouth(within(uv, vec4(-.3, -.4, .3, -.1))); // 嘴巴，自定义矩形左下角坐标(-.3, -.4)右上角坐标(.3, -.1)。
            
    
    col = mix(col, head, head.a); // 加入头部
    col = mix(col, eye, eye.a); // 加入眼睛
    col = mix(col, mouth, mouth.a); // 加入嘴巴
    
    return col;
    
}



void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
	vec2 uv = fragCoord.xy / iResolution.xy; // 屏幕空间归一化至0——1范围。
    uv -= 0.5; // 从0——1范围变为-0.5——0.5范围，坐标(0,0)由左下角移到矩形中心，便于后面绘制。
    
    uv.x *= iResolution.x/iResolution.y; // 将矩形屏幕变为正方形屏幕。由于屏幕是个800*450的矩形，如果不处理，画出的圆会是椭圆。用x*(screenwidth/screenheight)能得到按屏幕宽高比例缩小的x坐标。
   
    
	fragColor = smiley(uv); //笑脸绘制
}
```

# 参考资料

- [无动画的笑脸](https://www.shadertoy.com/view/llscWn)
- [有动画的笑脸](https://www.shadertoy.com/view/lsXcWn)
- [教程地址](https://www.youtube.com/watch?v=ZlNnrpM0TRg)
- [动画教程地址](https://www.youtube.com/watch?v=vlD_KOrzGDc)