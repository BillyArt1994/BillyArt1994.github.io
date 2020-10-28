---
title: 透明+阴影+LightMap
date: 2020-05-04 12:11:02
tags: [lightmap,Unity,shader]
categories: shader
---
&emsp;**最近做了一个透明shader，但是要支持lightmap，然后就发现了一个有趣的现象，烘焙出来的阴影都是方块状态的并没有透明处理**
<!--more--> 
最初烘焙状态，左边黑色闪电为官方shader，LegacyShader/Transparent/Coutout
![微信截图_20200504121943.png](https://i.loli.net/2020/05/04/9oYTyR5uQi3Pbzh.png)
然后查到论坛中有说，是shader的名字问题= 0 =
真神奇，然后我试着把名字改了下
```java
Shader "Billy/TransparentCutout"  // 原本是Billy/test
{
	Properties
	{
		_Color ("Color",Color) = (0.5,0.5,0.5,1)
		_MainTex("MainTex", 2D) = "white" {}
		_Cutoff ("Cutoff",Range(0,1)) =0.5
	}
  ```
 没想到![1.png](https://i.loli.net/2020/05/04/nDpwQkHTbBAYhcq.png)
 说是lightmapper会去搜索"Transparent"去烘焙阴影
## 参考文献
 以上内容均来自本人观点以及网上资料,如果侵权，请联系删除，多谢观看。
<font size=3>1. 官方论坛---https://forum.unity.com/threads/alpha-cutout-lightmap-shadows.250269/