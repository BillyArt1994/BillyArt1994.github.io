---
title: RenderTexture中粒子不显示
date: 2020-11-28 15:53:38
tags: [Alpha,Unity,Shader]
categories: Shader
---
&emsp;**在用到RenderTexture的时候,经常出现在渲染带粒子特效的物体时,粒子总是不会被渲染出来。**
<!--more-->
![AlphaBlend.png](https://i.loli.net/2020/12/08/aZJdHKBqvcuWCoX.png)
![微信截图_20201208114147.png](https://i.loli.net/2020/12/08/xZlu3FTiGBRzMKh.png)
为什么会出现,在使用RawImage显示RenderTexture的时候粒子会被剔除,必须在有物体遮挡的情况下才能显示。
然后可以选中RenderTexture查看一下，可以发现RenderTexture其实渲染上了粒子
![微信截图_20201208120104.png](https://i.loli.net/2020/12/08/EkNLP7wZSXdTqg3.png)
既然如此那原因肯定就在RawImage的渲染上，回到RawImage发现其使用的是默认的UIDefault，可以去Unity官网下载到版本下使用的Shader包，查看UI-Default的时候
```
        Cull Off
        Lighting Off
        ZWrite Off
        ZTest [unity_GUIZTestMode]
        Blend SrcAlpha OneMinusSrcAlpha
        ColorMask [_ColorMask]
```
排出片元着色器，输出图片颜色之外，能够影响颜色的就只有Blend和ColorMask了，但是这里的ColorMask只是输出颜色并没有屏蔽掉颜色，所以问题出在Blend上
那么这个Blend到底是如何计算得到最终颜色的呢
Blend顾名思义混合，在Unity默认渲染管线中的位置是位于AlphaTest之后
![PipelineBlend.png](https://i.loli.net/2020/12/08/41BOP3XtAYMkfhI.png)
在Blend被开启的情况下，Blend操作会拿取已经存储在颜色缓存中的颜色值，与目前片元着色器得到的颜色进行混合，然后得到一个最终颜色，在这个过程中，因为在不同情况下需要到不同的结果，所以融合的算法是可控制的
公式为Blend SrcFactor DstFactor或者Blend SrcFactor DstFactor，SrcFactorA DstFactorA(SrcFactor,DstFactor都为操作因子)
先看下第一个式子Blend SrcFactor DstFactor,这里引入几个概念,分别是**源颜色-S**，**目标颜色-D**，**输出颜色-O**
- **源颜色**：目前片元着色器所得到的颜色
- **目标颜色**：目前储存在颜色缓存中的颜色
- **输出颜色**：很好理解那就最后经过Blend混合后的结果颜色  

在此公式下
O<sub>rgb</sub>=SrcFactor X S<sub>rgb</sub>+DstFactor X D<sub>rgb</sub>
O<sub>a</sub>=SrcFactor X S<sub>a</sub>+DstFactor X D<sub>a</sub>
翻译成语言就是，因子X源颜色的RGB+因子X目标颜色的RGB=输出颜色的RGB，Alpha值同理。
那么操作因子又有哪些呢
![微信截图_20201208154540.png](https://i.loli.net/2020/12/08/MTQYzlsgipI3RC6.png)
可以看到，在之前查看到的UI-Default中使用过的SrcAlpha以及OneMinusSrcAlpha现在再来理解下Blend SrcAlpha OneMinusSrcAlpha这个式子得到的颜色到底是怎么来的，把因子代入公式中
O<sub>rgb</sub>=SrcAlpha X S<sub>rgb</sub>+OneMinusSrcAlpha X D<sub>rgb</sub>
O<sub>a</sub>=SrcAlpha X S<sub>a</sub>+OneMinusSrcAlpha X D<sub>a</sub>
现在来看O<sub>rgb</sub>有个很关键的点SrcAlpha，源颜色会乘以源Alpha，目标颜色会乘以1-SrcAlpha，所以RenderTexture的Alpha值是什么样的呢
![微信截图_20201208160455.png](https://i.loli.net/2020/12/08/CfBYkibrIgyA3VW.png)
可以看到，粒子的Alpha并没有出现，是不是粒子并没有写入Alpha呢
去看一下粒子的Shader
```
    Blend SrcAlpha One
    ColorMask RGB
    Cull Off Lighting Off ZWrite Off
```
ColorMask RGB，利用ColorMask屏蔽了Alpha通道，使得粒子是不写入Alpha的，所以Blend SrcAlpha OneMinusSrcAlpha 是肯定得不到正确的渲染的，所以修改一下因子Blend One OneMinusSrcAlpha让源颜色全部通过
```
Shader "Billy/RenderTexture"
{
    Properties
    {
		_Color("Color",Color) = (0.5,0.5,0.5,1)
		[PerRendererData] _MainTex ("Texture", 2D) = "white" {}
		[Header(Stencil)]
		[IntRange]_Stencil("Stencil ID", Range(0,255)) = 0
		[Enum(UnityEngine.Rendering.CompareFunction)]_StencilComp("Stencil Comparison", Float) = 8
		[Enum(UnityEngine.Rendering.StencilOp)]_StencilOp("Stencil Operation", Float) = 0
		[Enum(UnityEngine.Rendering.StencilOp)]_StencilFail("Stencil Fail", Float) = 0
		[Enum(UnityEngine.Rendering.StencilOp)]_StencilZFail("Stencil ZFail", Float) = 0
		[Enum(UnityEngine.Rendering.ColorWriteMask)]_ColorMask("Color Mask", Float) = 15
		_StencilWriteMask("Stencil Write Mask", Float) = 255
		_StencilReadMask("Stencil Read Mask", Float) = 255
		[Header(Option)]
		[Enum(UnityEngine.Rendering.CullMode)]_CullMode("CullMode", float) = 2
		[Enum(Off, 0, On, 1)]_ZWriteMode("ZWriteMode", float) = 1
		[Enum(UnityEngine.Rendering.BlendOp)]  _BlendOp("BlendOp", Float) = 0
		[Enum(UnityEngine.Rendering.BlendMode)] _SrcBlend("SrcBlend", Float) = 1
		[Enum(UnityEngine.Rendering.BlendMode)] _DstBlend("DstBlend", Float) = 0
    }
    SubShader
    {
        Tags { "RenderType"="Opaque""PreviewType" = "Plane"
			"CanUseSpriteAtlas" = "True" }

		Stencil
		{
			Ref[_Stencil]
			Comp[_StencilComp]
			Pass[_StencilOp]
			ReadMask[_StencilReadMask]
			WriteMask[_StencilWriteMask]
		}

		BlendOp[_BlendOp]
		Blend[_SrcBlend][_DstBlend]
		Cull[_CullMode]
		ZWrite[_ZWriteMode]
		ZTest On
		ColorMask[_ColorMask]

        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"

            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct v2f
            {
                float2 uv : TEXCOORD0;
                float4 vertex : SV_POSITION;
            };

            sampler2D _MainTex;
            float4 _MainTex_ST;

            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                o.uv = TRANSFORM_TEX(v.uv, _MainTex);
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                fixed4 col = tex2D(_MainTex, i.uv);
                return col;
            }
            ENDCG
        }
    }
}

```
## 参考文献
 以上内容均来自本人观点以及网上资料,如果侵权，请联系删除，多谢观看。
 1.Unity手册-https://docs.unity3d.com/cn/2018.4/Manual/SL-Blend.html
 2.Unity Shader之Blend-https://zhuanlan.zhihu.com/p/133762725