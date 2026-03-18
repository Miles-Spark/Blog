---
title: UnityShader学习笔记--复杂光照
date: 2025-08-17 15:01:23
tags: Shader
categority: 计算机图形学
---


# 复杂的光照

### 一、Unity 渲染路径

渲染路径决定了光照如何应用到 Unity Shader 中的，渲染路径主要有三种： 前向渲染路径、延迟渲染路径、顶点照明渲染路径（unity5.0 后被抛弃）。在 Unity Shader 中用 LIghtMode 标签来指定不同的渲染路径

#### 前向渲染路径

1.原理

进行一次完整的前向渲染需要计算两个缓冲区的信息，一个是深度缓冲区，一个是颜色缓冲区。利用深度缓冲区来决定一个片元是否可见，可见就更新颜色缓冲区中的颜色值。对于每一个光源都需要进行一次完整的渲染流程。如果一个物体位于多个光源的影响区域内，该物体就需要执行多个 pass,每个 pass 计算光源的光照效果，然后再帧缓存中将所有光照效果混合起来得到最终的颜色值。假设一个一个场景里面有 N 个物体，M 个光源，那么渲染场景一共需要 N*M 个 Pass。

2.Unity 中的前向渲染

事实上，一个 Pass 不仅仅可以用来计算逐像素光照，还可以用来逐顶点光照等。在 Unity 中前向渲染路径有三种处理光照的方式： **逐顶点处理、逐像素处理、球谐处理。**决定一个光源使用哪种处理模式取决于它的类型和渲染模式。再前向渲染中，unity 会根据场景中各个光源的设置以及光源对物体的影响程度对这些光源进行重要度排序。其中一定数量的光源按照逐像素处理，最多四个按照注逐顶点处理，剩余光源可以按照 SH 球谐方式处理。

3.内置光照变量和函数

内置光照变量

<table>
<tr>
<td>名称<br/></td><td>类型<br/></td><td>描述<br/></td></tr>
<tr>
<td>_LightColor0<br/></td><td>float4<br/></td><td>逐像素光源的颜色<br/></td></tr>
<tr>
<td>_LightMatrix0<br/></td><td>float4x4<br/></td><td>从世界空间到光源空间的变换矩阵<br/></td></tr>
<tr>
<td>unity_4LightPosX0,<br/>unity_4LightPosY0,<br/>uniyt_4LightPosZ0<br/></td><td>float4<br/></td><td>仅用于Base Pass。前四个非重要的点光源在世界空间中的位置<br/><br/></td></tr>
<tr>
<td>unity_4LightAtten0<br/></td><td>float<br/></td><td>仅用于Base Pass。存储了前4个非重要的点光源的衰减因子<br/></td></tr>
<tr>
<td>unity_LightColor<br/></td><td>half4[4]<br/></td><td>仅用于Base Pass。存储了前四个非重要点光源的颜色<br/></td></tr>
</table>


内置光照函数

<table>
<tr>
<td>float3 WorldSpaceLightDir(float4 v)<br/></td><td>仅用于前向渲染中。输入一个模型空间中的顶点位置，返回世界空间下该点到光源方向的光照方向。<br/></td></tr>
<tr>
<td>float3 UnityWorldSpaceLightDir(float4 v)<br/></td><td>仅用于前向渲染中。输入一个世界空间中的顶点位置，返回世界空间下该点到光源方向的光照方向。<br/></td></tr>
<tr>
<td>float3 ObjWorldSpaceLightDir(float4 v)<br/></td><td>仅用于前向渲染中。输入一个模型空间中的顶点位置，返回模型空间下该点到光源方向的光照方向。<br/></td></tr>
<tr>
<td>float3 Shade4PointLights(...)<br/></td><td>仅用于前向渲染中。计算四个顶点的光照。<br/></td></tr>
</table>


#### 顶点照明渲染路径

顶点照明渲染路径是对硬件配置要求最小，运算性能最高，但是得到效果最差，而且不支持逐像素才能得到的效果，如阴影、法线映射、高光精度的高光反射等。

顶点照明渲染路径通常是在一个 pass 中就可以完成对物体的渲染，并且计算是按照逐顶点处理的。由于顶点照明渲染仅仅是前向渲染路径中的一个子集，因此在 Unity5 后移除了该渲染路径。

#### 延迟渲染路径

延迟渲染是来解决前向渲染在场景中有较多光源造成的性能瓶颈问题。

1.原理

延迟渲染主要包含两个 Pass。第一个 Pass 不进行任何光照计算，主要是通过深度缓冲区计算哪些片元是可见的，如果方向片元是可见的，将把片元相关的信息放入 G 缓冲区中。然后再第二个 Pass 中，利用 G 缓冲区的各个片元信息，例如表面法线，视角方向，漫反射系数等进行真实的光照计算。

2.Unity 中的延迟渲染

对于延迟渲染来说，它适合再场景光源数目很多，使用前向渲染会造成性能瓶颈的情况。但是延迟渲染也有一些缺点：不支持真正的抗锯齿功能； 不能处理半透明物体； 对显卡有一定要求，显卡必须支持 MRT， ShaderMode 3.0 及以上， 深度渲染纹理和双面模板缓冲。

3.可访问的内置变量和函数

<table>
<tr>
<td>名称<br/></td><td>类型<br/></td><td>描述<br/></td></tr>
<tr>
<td>_LightColor<br/></td><td>float4<br/></td><td>光照颜色<br/></td></tr>
<tr>
<td>_LightMatrix0<br/></td><td>float4x4<br/></td><td>从世界空间到光源空间的变换矩阵<br/></td></tr>
</table>


### 二、Unity 的光源类型与衰减

unity 一共支持四种光源类型，平行光、点光源、聚光灯和面光源。

#### 光源类型的影响

不同光源类型在 Shader 上体现的属性会有所不同。

1.平行光

平行光几何定义最简单，平行光可以照亮的范围没有限制，没有唯一的位置，也就没有衰减的概念，光照强度不会随着距离而发生改变。平行光到场景中所有点的方向是一致的。

2.点光源

点光源表示由一个点发出的向所有方向延展的光线，照亮空间有限，光照强度会随着距离衰减。

3.聚光灯

聚光灯是比较复杂的，照亮的空间是有限的，不再是简单的球体，而是由一块锥形区域定义在锥形顶点处光照强度最强，锥形边缘处强度最弱。

#### Unity 的光照衰减

unity 中使用一张纹理作为查找表来计算逐像素计算光照的衰减，这种做法在一定程度上可以提升性能，但是也存在一些弊端，比如：1. 需要预处理得到纹理，纹理的大小会影响衰减的精度； 2. 不够直观。

1.从衰减纹理中采样衰减值

为了对_LightTexture0 纹理采样得到定点到光源的衰减值，首先要得到该点在光源空间中的位置，需要通过_LightMatrix0 矩阵变换得到。

```csharp
float3 lightCoord = mul(_LightMatrix0, float4(i.worldPosition,1).xyz);
//使用在光源空间中距离的平方来采样可以避免开方，然后在使用宏UNITY_ATTEN_CHANNEL来得到衰减值所在的分量
fixed atten = tex2D(_LightTexture0, dot(lightCoord, lightCoord).rr).UNITY_ATTEN_CHANNEL;
```

2.使用数学公式计算衰减

```csharp
float distence = length(_WorldSpaceLight-Pos0.xyz - i.worldPosition.xyz);
atten = 1.0 / distance;
```

由于我们没法通过 Shader 内置变量得到光源的范围、聚光灯的朝向、张开的角度，因此得到的效果在有些时候往往不尽如人意。

### 三、Unity 阴影

#### 阴影是如何实现的

Shader Map 技术首先将光源位置与摄像机位置重合，那么场景中该光源的阴影区域就是摄像机看不到的区域。

前向渲染路径中如果最重要的平行光开启了阴影，Unity 就会为该光源计算它的阴影映射纹理（shader Map)，这张纹理本质上也是一张深度图，记录了从光源出发到能看到场景中距离它最近的表面位置。在 shander 中使用一个 Light Mode 为 Shadow Caster 的 Pass 来计算该映射纹理。

#### 统一管理光照衰减和阴影

光照衰减和阴影对最终物体的渲染结果影响本质上是相同的，最后都要把衰减因子和阴影之及光照结果相乘得到最终的渲染结果。unity shader 中可以通过内置宏 UNITY_LIGHT_ATTENUATION 接受光照衰减和阴影来共同计算，这样就不需要在 Additional Pass 中判断光源类型来处理光照衰减了。