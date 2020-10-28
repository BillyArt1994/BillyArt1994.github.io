---
title: Shader—模拟光源角色展示渲染
date: 2020-01-04 12:05:44
categories: Shader
tags: [shader,模拟灯光,角色渲染]
---
&emsp;**最近项目开发需要一个3D英雄角色的展示界面，但是原本是2DSLG的游戏，所以没有适用的灯光及shader，而且模型只有一张手绘贴图，于是打算做个低消耗的光照模型，在内部写入灯光，然后考虑到美观问题所以用Matcap的办法制作一个通道可以模拟边缘光等，材质部分就用高光通道，做下金属及皮肤的区别。**
  <!--more--> 
***
![CharacterShow1.gif](https://i.loli.net/2020/01/04/T8Yv2jHR5SOzxVI.gif)
## 设定属性值
先设定几个属性值，主要的关键值
**Matcap** —— 用来模拟边缘光
**Diffuse** —— 基础色贴图
**LightDir** —— 灯光方向
**RampTex** —— 渐变贴图模拟灯光的衰减
```
_MainColor ("Main Color", Color) = (1,1,1,1)
_MatCapDiffuse ("MatCapDiffuse (RGB)", 2D) = "white" {}
_DiffuseTex ("DiffuseTex",2D) = "white" {}
_Gloss ("Gloss",Range(8,512))=128
_Specular ("Specular",2D) = "white" {}
_RampTex ("RampTex",2D) = "gray" {}
_lightDir ("lightDir",Float)= (0,0.25,0.5,0)
_LightStrength ("LStrength",Range(0,2)) =1
```
## 计算Matcap模拟边缘光
Matcap是一个很优秀的算法，计算成本非常低，效果也非常好，之前看过一篇[专门介绍MatCap的文章](https://zhuanlan.zhihu.com/p/37702186 "介绍MatCap文章")
![matcap_LineGlow.jpg](https://i.loli.net/2020/01/04/58XmejdVvQc2xEY.jpg)
```
fixed4 diffuse = tex2D(_DiffuseTex, i.uv);
fixed3 matcapLookup = tex2D(_MatCapDiffuse, i.viewNormal.xy*0.5+0.5).rgb;
```
## 用Vector4替换灯光向量

这里做法很普通，把_LightDir代替平常获取_WorldSpaceLightPos0.xyz，进行计算，然后Ramp用的是一个一维的普通黑白渐变图模拟BRDF ![微信截图_20200104180557.png](https://i.loli.net/2020/01/04/tvMFoKEH3BIbCdg.png)
可以换成二维的[乐乐女神的这篇文章有讲到](https://blog.csdn.net/candycat1992/article/details/17485631 "模拟BRDF")
![e5beaee4bfa1e688aae59bbe_20191226182942-1.png](https://i.loli.net/2020/01/04/cIAut2QPwX1sbH9.png)
```
fixed3 worldNormal = normalize(i.worldNormal);
fixed3 lightDir = normalize(_lightDir);
fixed dotNL =saturate(dot(worldNormal,lightDir))*0.5+0.5;
fixed3 rampTex = tex2D(_RampTex,float2(dotNL,0))*_LightStrength;
```
## 计算高光
基本的高光算法，只是拿_LightDir替换计算的
![e5beaee4bfa1e688aae59bbe_20191226183229.png](https://i.loli.net/2020/01/04/uLqYHTjWJD6dmyO.png)
```
fixed3 specularTex =tex2D(_Specular,i.uv);
fixed3 viewDir = normalize(UnityWorldSpaceViewDir(i.worldPos));
fixed3 halfDir = normalize(lightDir+ viewDir);
fixed3 specular = pow(max(0,saturate(dot(worldNormal,halfDir))),_Gloss)
*_LightStrength*specularTex;
```
## 最终计算
我加上了一个i.worldPos.y，加强一下顶底关系
![e5beaee4bfa1e688aae59bbe_20191226183335.png](https://i.loli.net/2020/01/04/1aHI798GRD4cSbX.png)
最终算法就很简单了
漫反射贴图	&times;i.worldPos.y	&times;灯光模拟渐变+轮廓光+高光&times;颜色控制
```
fixed3 finalColor =(diffuse.rgb*i.worldPos.y*rampTex+matcapLookup+specular)*_MainColor;
```
![CharacterShow1.gif](https://i.loli.net/2020/01/04/T8Yv2jHR5SOzxVI.gif)
## 完整代码
有部分修改
```
Shader "Billy/Matcap/Character/Diffuse+Rim+Light"
{
Properties {
		_MainColor ("Main Color", Color) = (1,1,1,1)
		[NoScaleOffset]_MatCapDiffuse ("MatCapDiffuse (RGB)", 2D) = "white" {}
		_Cutoff("Cutoff",Range(0,1)) = 0.5
		[NoScaleOffset]_DiffuseTex ("DiffuseTex",2D) = "white" {}
		_Gloss("Gloss",Range(8,512))=128
		[NoScaleOffset]_Specular ("Specular",2D) = "white" {}
		[NoScaleOffset]_RampTex ("RampTex",2D) = "gray" {}
		_lightDir("lightDir",Float)= (0,0.25,0.5,0)
		_LightStrength("LStrength",Range(0,2)) =1
	}
	
Subshader {
Tags { "RenderType"="Queue" "Queue"="Geometry" }
Pass {
Tags { "LightMode" = "Always" }
			 
CGPROGRAM
#pragma vertex vert
#pragma fragment frag
#include "UnityCG.cginc"
#include "Lighting.cginc"

struct v2f { 
	half4 pos : SV_POSITION;
	half2 uv : TEXCOORD0;
	half4 viewNormal: TEXCOORD1;
	half3 worldNormal :TEXCOORD2;
	half3 worldPos : TEXCOORD3;
};								
sampler2D _MatCapDiffuse;
sampler2D _DiffuseTex;
sampler2D _Specular;
sampler2D _RampTex;
half3 _lightDir;
fixed _LightStrength;
half _Gloss;
fixed4 _MainColor;
fixed _Cutoff;
				
v2f vert (appdata_tan v)
{
v2f o;
o.pos = UnityObjectToClipPos (v.vertex);
o.worldPos = mul(unity_ObjectToWorld, v.vertex);
o.viewNormal = mul(UNITY_MATRIX_IT_MV, v.normal);
o.worldNormal = mul(v.normal,(float3x3)unity_WorldToObject);
o.uv = v.texcoord;
return o;
}

float4 frag (v2f i) : COLOR
{	
//matcap
fixed4 diffuse = tex2D(_DiffuseTex, i.uv);
fixed3 matcapLookup = tex2D(_MatCapDiffuse, i.viewNormal.xy*0.5+0.5).rgb;					
					//RampTex灯光模拟
fixed3 worldNormal = normalize(i.worldNormal);
fixed3 lightDir = normalize(_lightDir);
fixed dotNL =saturate(dot(worldNormal,lightDir))*0.5+0.5;
					fixed3 rampTex = tex2D(_RampTex,float2(dotNL,0))*_LightStrength;
//高光
fixed3 specularTex =tex2D(_Specular,i.uv);
fixed3 viewDir = normalize(UnityWorldSpaceViewDir(i.worldPos));
fixed3 halfDir = normalize(lightDir+ viewDir);
fixed3 specular = pow(max(0,saturate(dot(worldNormal,halfDir))),_Gloss)*_LightStrength*specularTex;
//最终颜色
fixed3 finalColor =(diffuse.rgb*(saturate(i.worldPos.y+0.6))*rampTex+matcapLookup+specular)*_MainColor;
return fixed4 (finalColor,1);
				}
			ENDCG
		}
	}
}

```
## 参考文献
<font size=2>1. 王者荣耀角色展示界面模拟---<https://zhuanlan.zhihu.com/p/34270569>
2. MatCap详细介绍---<https://zhuanlan.zhihu.com/p/37702186>
3. 崩坏3模拟实时灯光---<https://blog.csdn.net/baicaishisan/article/details/81287240>
4. 次时代模拟实时灯光---<http://www.mamicode.com/info-detail-2111220.html>
5. RampTex模拟PBRF---<https://blog.csdn.net/candycat1992/article/details/17485631>></font>