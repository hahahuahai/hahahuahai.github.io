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

[smoothstep函数](http://docs.gl/sl4/smoothstep)：这个函数作用是用来确定待渲染特征的范围，然后在作为mix函数的第三个参数，与原颜色进行混合。这两个函数经常搭配使用。

# 效果和代码

矩形屏幕空间的效果（代码含备注）

![](\images\posts\Shadertoy\shadertoy_smile1.gif)

无动画的笑脸代码

```c
// 作者为了简便，写的一些define
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

// 用于将uv坐标进行缩放。注意：rect是一个自定义的矩形范围，xy表示矩形的横纵坐标最小的坐标值（一般是左下角），zw表示矩形的横纵坐标最大的坐标值（一般是右上角），而不是矩形的width和height。之所以自定义一个这样的矩形范围，是为了方便各个子模块（眼睛、嘴巴和眉毛）里面的绘制，因为在这个自定义矩形里坐标范围是0——1的。
vec2 within(vec2 uv, vec4 rect)
{
    return (uv-rect.xy)/(rect.zw - rect.xy);    
}



vec4 Brow(vec2 uv) {
    //float offs = mix(.2, 0., smile);
   // uv.y += offs;
    
    float y = uv.y;
    uv.y += uv.x*.8 -.3 ;
    uv.x -= .1;
    uv -= .5;
    
    vec4 col = vec4(0.);
    
    float blur = .1;
    
   	float d1 = length(uv);
    float s1 = S(.45, .45-blur, d1);
    float d2 = length(uv-vec2(.1, -.2)*.7);
    float s2 = S(.5, .5-blur, d2);
    
    float browMask = sat(s1-s2);
    
    float colMask = remap01(.7, .8, y)*.75;
    colMask *= S(.7, .9, browMask);
    //colMask *= smile;
    vec4 browCol = mix(vec4(.4, .2, .2, 1.), vec4(1., .75, .5, 1.), colMask); 
   
    /*
    uv.y += .15-offs*.5;
    blur += mix(.0, .1, smile);
    d1 = length(uv);
    s1 = S(.45, .45-blur, d1);
    d2 = length(uv-vec2(.1, -.2)*.7);
    s2 = S(.5, .5-blur, d2);
    float shadowMask = sat(s1-s2);

	*/
    
    //col = mix(col, vec4(0.,0.,0.,1.), S(.0, 1., shadowMask)*.5);
    
    col = mix(col, browCol, S(.2, .4, browMask));
    
    return col;
}


vec4 Mouth(vec2 uv)
{
    uv -= 0.5;
    
 	vec4 col = vec4(.5, .18, .05, 1.);
    
    uv.y *= 1.5;
    uv.y -= uv.x*uv.x*2.;
    
    float d = length(uv);
    
    float td = length(uv-vec2(0.,.6));
    
    vec3 toothCol = vec3(1.)*smoothstep(.6,.35, d);
    
    col.rgb = mix(col.rgb, toothCol, smoothstep(.4, .37, td));
    
    td = length(uv+vec2(0.,.5));
    
    col.rgb = mix(col.rgb, vec3(1., .5, .5), smoothstep(.5, .2, td));
    
    col.a = smoothstep(.5, .48, d);
    
    return col;
    
}




vec4 Eye(vec2 uv)
{
    uv -= 0.5; 
    
    float d = length(uv);
    
    vec4 irisColor = vec4(.3, .5, 1., 1.); //虹膜的颜色，蓝色
    vec4 col = vec4(1.); // 巩膜的颜色，白色
    col = mix(col, irisColor, smoothstep(.1, .7, d)*.5);
    
    col.rgb *= 1. - smoothstep(.45, .5, d)* .5 * sat(-uv.y-uv.x);
    
    col.rgb = mix(col.rgb, vec3(0.), smoothstep(.3, .28, d));
    
    irisColor.rgb *= 1. + smoothstep(.3, .05, d); 
    
    col.rgb = mix(col.rgb, irisColor.rgb, smoothstep(.28, .25, d));
    col.rgb = mix(col.rgb, vec3(0.), smoothstep(.16, .14, d));
    
    
    float highlight = smoothstep(.1, .09, length(uv-vec2(-.15, .15)));
    
    highlight += smoothstep(.07, .05, length(uv+vec2(-.08, .08)));
    
    col.rgb = mix(col.rgb, vec3(1.), highlight);
    
    col.a = smoothstep(.5, .48, d);
    
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
    
    uv.x = abs(uv.x); // 脸（眼睛、眉毛、腮对称）的左右两边对称绘制
    
    vec4 head = Head(uv); // 头部
    vec4 eye = Eye(within(uv, vec4(.03, -.1, .37, .25))); // 眼睛
    vec4 mouth = Mouth(within(uv, vec4(-.3, -.4, .3, -.1)));
    vec4 brow = Brow(within(uv, vec4(.03, .2, .4, .45)));
            
    
    col = mix(col, head, head.a);
    col = mix(col, eye, eye.a);
    col = mix(col, mouth, mouth.a);
    col = mix(col, brow, brow.a);
    
    return col;
    
}



void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
	vec2 uv = fragCoord.xy / iResolution.xy;
    uv -= 0.5;
    
    uv.x *= iResolution.x/iResolution.y;
   
    
	fragColor = smiley(uv);
}
```

有动画的笑脸代码

```c
// "Smiley Tutorial" by Martijn Steinrucken aka BigWings - 2017
// License Creative Commons Attribution-NonCommercial-ShareAlike 3.0 Unported License.
// Email:countfrolic@gmail.com Twitter:@The_ArtOfCode
//
// This Smiley is part of my ShaderToy Tutorial series on YouTube:
// Part 1 - Creating the Smiley - https://www.youtube.com/watch?v=ZlNnrpM0TRg
// Part 2 - Animating the Smiley - https://www.youtube.com/watch?v=vlD_KOrzGDc&t=83s

// 作者为了简便，写的一些define
#define S(a, b, t) smoothstep(a, b, t)
#define B(a, b, blur, t) S(a-blur, a+blur, t)*S(b+blur, b-blur, t)
#define sat(x) clamp(x, 0., 1.)

// 将0-1空间的t重新映射到a-b空间
float remap01(float a, float b, float t) {
	return sat((t-a)/(b-a));
}
// 将a-b空间的t重新映射到c-d空间
float remap(float a, float b, float c, float d, float t) {
	return sat((t-a)/(b-a)) * (d-c) + c;
}

vec2 within(vec2 uv, vec4 rect) {
	return (uv-rect.xy)/(rect.zw-rect.xy);
}

vec4 Brow(vec2 uv, float smile) {
    float offs = mix(.2, 0., smile);
    uv.y += offs;
    
    float y = uv.y;
    uv.y += uv.x*mix(.5, .8, smile)-mix(.1, .3, smile);
    uv.x -= mix(.0, .1, smile);
    uv -= .5;
    
    vec4 col = vec4(0.);
    
    float blur = .1;
    
   	float d1 = length(uv);
    float s1 = S(.45, .45-blur, d1);
    float d2 = length(uv-vec2(.1, -.2)*.7);
    float s2 = S(.5, .5-blur, d2);
    
    float browMask = sat(s1-s2);
    
    float colMask = remap01(.7, .8, y)*.75;
    colMask *= S(.6, .9, browMask);
    colMask *= smile;
    vec4 browCol = mix(vec4(.4, .2, .2, 1.), vec4(1., .75, .5, 1.), colMask); 
   
    uv.y += .15-offs*.5;
    blur += mix(.0, .1, smile);
    d1 = length(uv);
    s1 = S(.45, .45-blur, d1);
    d2 = length(uv-vec2(.1, -.2)*.7);
    s2 = S(.5, .5-blur, d2);
    float shadowMask = sat(s1-s2);
    
    col = mix(col, vec4(0.,0.,0.,1.), S(.0, 1., shadowMask)*.5);
    
    col = mix(col, browCol, S(.2, .4, browMask));
    
    return col;
}

vec4 Eye(vec2 uv, float side, vec2 m, float smile) {
    uv -= .5;
    uv.x *= side;
    
	float d = length(uv);
    vec4 irisCol = vec4(.3, .5, 1., 1.);
    vec4 col = mix(vec4(1.), irisCol, S(.1, .7, d)*.5);		// gradient in eye-white
    col.a = S(.5, .48, d);									// eye mask
    
    col.rgb *= 1. - S(.45, .5, d)*.5*sat(-uv.y-uv.x*side); 	// eye shadow
    
    d = length(uv-m*.4);									// offset iris pos to look at mouse cursor
    col.rgb = mix(col.rgb, vec3(0.), S(.3, .28, d)); 		// iris outline
    
    irisCol.rgb *= 1. + S(.3, .05, d);						// iris lighter in center
    float irisMask = S(.28, .25, d);
    col.rgb = mix(col.rgb, irisCol.rgb, irisMask);			// blend in iris
    
    d = length(uv-m*.45);									// offset pupile to look at mouse cursor
    
    float pupilSize = mix(.4, .16, smile);
    float pupilMask = S(pupilSize, pupilSize*.85, d);
    pupilMask *= irisMask;
    col.rgb = mix(col.rgb, vec3(0.), pupilMask);		// blend in pupil
    
    float t = iTime*3.;
    vec2 offs = vec2(sin(t+uv.y*25.), sin(t+uv.x*25.));
    offs *= .01*(1.-smile);
    
    uv += offs;
    float highlight = S(.1, .09, length(uv-vec2(-.15, .15)));
    highlight += S(.07, .05, length(uv+vec2(-.08, .08)));
    col.rgb = mix(col.rgb, vec3(1.), highlight);			// blend in highlight
    
    return col;
}

vec4 Mouth(vec2 uv, float smile) {
    uv -= .5;
	vec4 col = vec4(.5, .18, .05, 1.);
    
    uv.y *= 1.5;
    uv.y -= uv.x*uv.x*2.*smile;
    
    uv.x *= mix(2.5, 1., smile);
    
    float d = length(uv);
    col.a = S(.5, .48, d);
    
    vec2 tUv = uv;
    tUv.y += (abs(uv.x)*.5+.1)*(1.-smile);
    float td = length(tUv-vec2(0., .6));
    
    vec3 toothCol = vec3(1.)*S(.6, .35, d);
    col.rgb = mix(col.rgb, toothCol, S(.4, .37, td));
    
    td = length(uv+vec2(0., .5));
    col.rgb = mix(col.rgb, vec3(1., .5, .5), S(.5, .2, td));
    return col;
}
//头部渲染
vec4 Head(vec2 uv) {
	vec4 col = vec4(.9, .65, .1, 1.);
    
    float d = length(uv);
    
    col.a = S(.5, .49, d);
    
    // 脸的边缘（边缘暗色+边缘描边）
    //// 边缘暗色
    float edgeShade = remap01(.35, .5, d);
    edgeShade *= edgeShade;
    col.rgb *= 1.-edgeShade*.5;    
    //// 边缘描边
    col.rgb = mix(col.rgb, vec3(.6, .3, .1), S(.47, .48, d));
    
    float highlight = S(.41, .405, d);
    highlight *= remap(.41, -.1, .75, 0., uv.y);
    highlight *= S(.18, .19, length(uv-vec2(.21, .08)));
    col.rgb = mix(col.rgb, vec3(1.), highlight);
    
    //腮cheek
    d = length(uv-vec2(.25, -.2));
    float cheek = S(.2,.01, d)*.4;
    cheek *= S(.17, .16, d);
    col.rgb = mix(col.rgb, vec3(1., .1, .1), cheek);
    
    return col;
}
//整个场景渲染（笑脸+背景）
vec4 Smiley(vec2 uv, vec2 m, float smile) {
	vec4 col = vec4(0.);	//背景
    
    if(length(uv)<.5) {					// only bother about pixels that are actually inside the head
        float side = sign(uv.x);
        uv.x = abs(uv.x); // 脸的左右两边对称绘制（眼睛、眉毛、腮对称）
        vec4 head = Head(uv);	//头部
        col = mix(col, head, head.a);

        if(length(uv-vec2(.2, .075))<.175) {
            vec4 eye = Eye(within(uv, vec4(.03, -.1, .37, .25)), side, m, smile);
            col = mix(col, eye, eye.a);
        }

        if(length(uv-vec2(.0, -.15))<.3) {
            vec4 mouth = Mouth(within(uv, vec4(-.3, -.43, .3, -.13)), smile);
            col = mix(col, mouth, mouth.a);
        }

        if(length(uv-vec2(.185, .325))<.18) {
            vec4 brow = Brow(within(uv, vec4(.03, .2, .4, .45)), smile);
            col = mix(col, brow, brow.a);
        }
    }
    
    return col;
}

void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
	float t = iTime;
    
    vec2 uv = fragCoord.xy / iResolution.xy;	//归一化，坐标范围（0,1）
    uv -= .5;	//把坐标原点移动到屏幕中间，坐标范围（-0.5,0.5）
    uv.x *= iResolution.x/iResolution.y;	//将x轴映射到y轴，即将矩形空间变为正方形空间，方便画正圆。
    
    vec2 m = iMouse.xy / iResolution.xy;
    m -= .5;
    
    if(m.x<-.49 && m.y<-.49) {			// make it that he looks around when the mouse hasn't been used
    	float s = sin(t*.5);
        float c = cos(t*.38);
        
        m = vec2(s, c)*.4;
    }
    
    if(length(m) > .707) m *= 0.;		// fix bug when coming back from fullscreen
    
    float d = dot(uv, uv);
    uv -= m*sat(.23-d);
    
    float smile = sin(t*.5)*.5+.5;
	fragColor = Smiley(uv, m, smile);
}
```

# 参考资料

- [无动画的笑脸](https://www.shadertoy.com/view/llscWn)
- [有动画的笑脸](https://www.shadertoy.com/view/lsXcWn)
- [教程地址](https://www.youtube.com/watch?v=ZlNnrpM0TRg)
- [动画教程地址](https://www.youtube.com/watch?v=vlD_KOrzGDc)