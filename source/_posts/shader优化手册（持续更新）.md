---
title: shader优化笔记（持续更新）
date: 2020-01-07 15:38:45
tags: [shader,优化,移动端]
categories: Shader
---
&emsp;**记录一些平常遇到和需要注意的优化点，转载出处及地址会写在最后。**
    <!--more--> 
 ## **[Gamma]与[Linear]指定**
>当颜色空间配制为Gamma时，对属性变量指定为[Gamma]和[Linear]都是无效的，只有当颜色空间配制为Linear时，指定才有效。所以当我们我们需要将一个Vector做为颜色使用，在Linear颜色空间下，指定其为[Linear]是比[Gamma]要高效的，因为如果指定为[Gamma]，Unity会在CPU端做一次转换计算。
```python
Properties: [Gamma][Linear]
[Linear]_myVector("display vector",Vector)=0.5
[Gamma]_myVector("display float",Float)=0.5

myColor("display color",Color)=(0.5,0.5,0.5,0.5)
颜色空间配制为Gamma时，[Gamma]与[Linear]指定无效
颜色空间配制为Linear时，[Gamma]与[Linear]指定有效
```

## **[NoScaleOffset]**
>[NoScaleOffset]可以为我们省掉一个变量的分配，如果不需要用到纹理Tiling和Offset，请使用[NoScaleOffset]。
```python
Properties: [NoScaleOffset]
[NoScaleOffset]my2D("display 2d",2D) = "white"{}

sampler2D my2D;
float4 my2D_ST;
v2f vert (appdata v)
{
      v2f o;
      o.vertex = UnityObjectToClipPos(v.vertex);
      o.uv = TRANSFORM_TEX(v.uv, my2D);
      return o;
}
```
## **[PerRendererDate]**
>如果要为每个材质设不同的颜色，Unity会帮助你创建一百个这样的材质，相当于你渲染一百个物体，每个物体颜色不一样，所以直接设置的时候，会帮助你创建一百个材质，这对于GPU来讲会影响，但是更重要是内存方面，会有一百个材质的内存开销，这是一个最大的瓶颈。所以我们引入了PerRenderData的这么一个属性的控制，它可以帮你把这个数据。例如：Color颜色，我们不需要将其画到材质里面，而是通过Render可以设置。
```Python
Properties: [PerRendererDate]

mayColor("display color",Color)=(0.5,0.5,0.5,0.5)
[PerRendererDate]mayColor("display color",Color)=(0.5,0.5,0.5,0.5)
```

## **DisableBatching**
>DisableBatching建议大家不要开启，默认的情况是不开启的。
>如果顶点计算必须在object space下进行，则需要开启，否则尽量不要开启
```Python
SubShader:Tags
Tags{"DisableBatching"="True"|"False"|"LodFading"}
```

## **ForceNoShadowCasting**
>当我们画不透明的物体，需要替换半透明的物体，但是半透明的物体没有阴影，你可以更改Shader代码让它不投射阴影，直接加上ForceNoShadowCasting就可以了。
```java
SubShader:Tags
Tags{"ForceNoShadowCasting"="True"|"False"}
替换shader时非常有用，如将不透明的物体变成半透明
```

## **GrabPass**
>我们需要抓缓冲图有二种方式，如下图所示，这二种方式的区别是很明显的，第一种方式是比较低效的，因为GrabPass调用的时候必定会进行抓取的操作，所以没次都是不同的。但是下面比较高效，一帧里面最多只执行一次，就是第一次使用会执行，后面不会执行。这个根据大家的实际的应用进行选择。
```java
SubShader:GrabPass
GrabPass
{
}
sampler2D _GrabTexture;

GrabPass
{
"_BackgroundTexture"
}
sampler2D _BackgroundTexture;
```

## **Render State**
>图形程序可能会比较关注怎么设置渲染状态，当它有变化的时候我们需要设置，没有变化时候该怎么处理?
Unity引擎是这样处理的，它把渲染状态缓存，缓存的作用就是为了不要频繁的切换，因为会判断当前这个状态和上一次的状态是不是相同？如果是相同，不需要调用图形API的接口，因为接口调用也有一定的消耗。那么这里可以引出一个优化点，如果渲染状态频繁的切换，那么起不到优化的作用。
所以我们写Shader的时候，尽量不要让这些连续的，不要有太多的渲染状态，尽量保证是少的，这样优化作用是大的。例如：AAA连续和BBB连续画，这样的效率高于AB间插的渲染。 
![123.jpg](https://i.loli.net/2020/01/07/bmFMns4JywlVEe7.jpg)

## **Alpha Testing**
>限于硬件的机能限制，移动端尽量不要开启，在移动端开启，它的性能会比较低。

## **Color Mask**
>移动端也尽量不要开启，这是固定的，大家一定要记住，因现在受于移动端的限制，PC端没有这个限制。

## **Surface Shader优化**
>* 首先是Approxview，它从View Direction Normalized移到Vertex Shader进行计算。
* 其次是Halfasview，是使用光照方向和视角方向的中间向量替代一个视角方向，如果大家对比效果，觉得二种比较下来差不多，就可以选择优化过的效果。
* Noforwarddadd，如果只有一盏像素光，你可以开启这个选项。
* 最后是环境光，我们可以关掉一个环境光，因为关了之后有一些优化的计算，这里优化强度比较大，但是光照损失也是比较大，如果觉得效果可以接受，就可以关掉它。

## **代码优化**
>可优化的空间不大，空间大的地方在哪儿？我们思考优化这些最后的效果，思考这些数据是不是真的应该去进入了渲染管线？这是重点。
>进入渲染管线的数据优化之后，我们再做Shader优化就是锦上添花的事情。首先我们CPU的软件裁减是不是高效且正确？当然我们会使用Unity的裁减功能，或者可以自己写。
>当有Over Draw时，渲染顺序是不是可以再优化 。例如：远处的阴影是不是真的需要开启？分辨率可以再小一点吗，最后一个全屏特效做的次数，大家讲的Bloom次数非常多，可不可以减少一些？或者其它的全屏特效可不可以集中一次把它做完,这些事情我们思考之后已经优化了很多效率。
>* 我们要保证效果的前提下，尽量把计算放在Vertex Shader 。
>* 我们尽量不要写多Pass，SubShader。
>* 善用LOD，我们Mesh有LOD，纹理有Mipmap，Shader也有LOD。GPU执行二个Float的乘法和执行二个Vector4的乘法效率是一样的。
>* 少用分支， 还有一些内置函数，不建议自定义实现，我们可以使用提供的函数，那些是经过优化的。
>* 最后是精度问题，Fixed，Half、Float，不同的设备上它的性能不一样，我们尽量移动端使用一些低精度的数据。PC端这三种是没有什么区别的。

## 参考文献
 以上内容均来自本人观点以及网上资料,如果侵权，请联系删除，多谢观看。
<font size=3>1. Unite 2018 大会技术专场 | Unity Shader着色器优化---<https://connect.unity.com/p/unity-shaderzhao-se-qi-you-hua?_ga=2.232505247.1413049713.1578380858-1230826648.1578380858></font>