---
title: Shader—模板测试
date: 2020-04-04 21:13:57
categories: Shader
tags: [shader,模板测试,Stencil，透明]
---
&emsp;**这周遇到一个模板测试的问题，因为之前一直没把模板测试弄明白，官方的文档又比较简单所以这次做了一次深层次的测试与理解。**
<!--more-->
 ## **模板测试在渲染流水线上的位置**
首先要有一个Unity的渲染流水线的了解，模板测试处在Unity渲染的哪一个步骤。
![GPU流水线.png](https://i.loli.net/2020/04/05/lI1vhcUeM6DdqmV.png)
前面的几何阶段直接跳过，我们的模板测试以及深度测试和混合是处在<u>**逐片元操作**</u>这一步里面的
![微信截图_20200405180043.png](https://i.loli.net/2020/04/05/unkIWHp4vrA8L3D.png)
然后我们可以看见，模板测试所处的位置，以及模板测试前后我们还会经行深度测试以及AlphaTest和混合。
 ## **参数**
现在来看一下StecilTest的具体参数，我下载了官方的Builtin_Shader包，查看了UI-Default这个shader，里面使用到的参数以及语法格式类型可以参考看一下
```C#
 Properties
    {
        _StencilComp ("Stencil Comparison", Float) = 8
        _Stencil ("Stencil ID", Float) = 0
        _StencilOp ("Stencil Operation", Float) = 0
        _StencilWriteMask ("Stencil Write Mask", Float) = 255
        _StencilReadMask ("Stencil Read Mask", Float) = 255
    }
        Stencil
        {
            Ref [_Stencil]
            Comp [_StencilComp]
            Pass [_StencilOp]
            ReadMask [_StencilReadMask]
            WriteMask [_StencilWriteMask]
        }
```

总共有这几个参数以及属性**Ref，Comp，Pass，Fail，ZFail，WriteMask，ReadMask.**

<font color=DarkRed>**Ref**</font>
>Ref referenceValue

**Ref**是用来设定自身的模板值**referenceValue**——中文名称为参考值，这个值将用来与模板缓冲中<u>**已经存有的模板值**</u>来做比较，取值范围在0-255之间的整数。


<font color=DarkRed>**Comp**</font>
>Comp comparisonFunction

Comp就是上述所说Ref（referenceValue）与已经存有的模板值（stencilBufferValue）进行比较的函数，默认值是：Always

函数 | 属性|取值
-|-|-
Greater | 等于“>”操作，即仅当左边>右边，模板测试通过，更新像素|
GEqual | 等于“>=”操作，即仅当左边>=右边，模板测试通过，更新像素|
Less | 等于“<”操作，即仅当左边<右边，模板测试通过，更新像素|
LEqual | 等于“<=”操作，即仅当左边<=右边，模板测试通过，更新像素|
Equal | 等于“=”操作，即仅当左边=右边，模板测试通过，更新像素|
NotEqual |等于“!=”操作，即仅当左边!=右边，模板测试通过，更新像素|
Always |不管公式两边为何值，模板测试总是通过，更新像素|
Never |不管公式两边为何值，模板测试总是失败 ，像素被抛弃|

<font color=DarkRed>**Pass**</font>

>Pass stencilOperation

Pass是指经过函数比较<u>**通过后**</u>，对当前缓存中的模板值（stencilBufferValue）做什么样的处理，默认值：Keep

函数 | 属性|取值
-|-|-
Keep | 保留当前缓冲中的内容，即stencilBufferValue不变|
Zero | 将0写入缓冲，即stencilBufferValue更新为0|
Replace | 将参考值写入缓冲，即将referenceValue赋值给stencilBufferValue|
IncrSat | stencilBufferValue加1，如果stencilBufferValue超过255了，那么保留为255，即不大于255|
DecrSat | stencilBufferValue减1，如果stencilBufferValue超过为0，那么保留为0，即不小于0|
Invert |将当前模板缓冲值（stencilBufferValue）按位取反|
IncrWrap |当前缓冲的值加1，如果缓冲值超过255了，那么变成0，（然后继续自增）|
DecrWrap |当前缓冲的值减1，如果缓冲值已经为0，那么变成255，（然后继续自减）|

在更新模板缓冲值的时候，也有writeMask进行掩码操作，用来对特定的位进行写入和屏蔽，默认值为255（11111111），即所有位数全部写入，不进行屏蔽操作。

<font color=DarkRed>**Fail**</font>

>Fail stencilOperation

Fail是指是指经过函数比较<u>**未通过**</u>，对当前缓存中的模板值做什么样的处理，默认值：Keep

<font color=DarkRed>**ZFail**</font>

ZFail是指当模板测试通过而深度测试失败时，对当前缓存中的模板值做什么样的处理，默认值：Keep

<font color=DarkRed>**ReadMask**</font>

>ReadMask  readMask

ReadMask 从字面意思的理解就是读遮罩，readMask将和referenceValue以及stencilBufferValue进行按位与（&）操作，readMask取值范围也是0-255的整数，默认值为255，二进制位11111111，即读取的时候不对referenceValue和stencilBufferValue产生效果，读取的还是原始值。

<font color=DarkRed>**WriteMask**</font>

>WriteMask writeMask

WriteMask是当写入模板缓冲时进行掩码操作（按位与【&】），writeMask取值范围是0-255的整数，默认值也是255，即当修改stencilBufferValue值时，写入的仍然是原始值。

Comp，Pass,Fail 和ZFail将会应用给背面消隐的几何体（只渲染前面的几何体），除非Cull Front被指定，在这种情况下就是正面消隐的几何体（只渲染背面的几何体）。你也可以精确的指定双面的模板状态通过定义CompFront，PassFront，FailFront，ZFailFront（当模型为front-facing geometry使用）和ComBack，PassBack，FailBack，ZFailBack（当模型为back-facing geometry使用）。

 ## 模板测试的公式
 
 if（referenceValue&readMask <font color=DarkRed>**comparisonFunction** </font>stencilBufferValue&readMask）
通过像素
else
抛弃像素

## 结论
1.模板测试设置中最重要的几个值，模板参考值（referenceValue）与比较函数（Comp）和通过后的处理（pass）
2.在使用模板测试中一定要仔细想想目前场景中的渲染顺序问题，到底先经行模板测试的是哪个物体，模板测试是否通过，通过后像素有没有更新，缓存中的模板有没有被更新替换。
3.在没有进行任何赋值情况下最初模板缓存值为0。
## 举个栗子
![1.gif](https://i.loli.net/2020/04/05/5uNCiZct1xwKR4O.gif)
一个普通的遮罩效果
这里的思路是，首先关闭面片的Zwrite调整渲染序列Queue为1999，这样的话，渲染顺序就是面片>猴子，此时我们设置面片的Ref为1，然后Comp总是通过Always，Pass设置为替换刷新Relace，把自身颜色设置为0。
![微信截图_20200405210816.png](https://i.loli.net/2020/04/05/uyqra1bJGEBnTKD.png)
猴子的话我们就很简单了，设置Ref为1，条件是当缓存中的模板值为1才通过刷新像素颜色，其他保持不变就行
![2.png](https://i.loli.net/2020/04/05/4LE8M2jQygYXJTW.png)

再举个栗子
![2.gif](https://i.loli.net/2020/04/05/sJQLe3XNibO29EK.gif)
这是一个镂空效果
思路差不多，首先关闭面片的Zwrite调整渲染序列Queue为1999，然后设置面片的Ref为0，这里的comp设置为Always或者Equal都行，然后pass设置为通过后＋1刷新缓存中的模板值。
![3.png](https://i.loli.net/2020/04/05/3KANn2yszQhqEVI.png)
茶壶的话保持不变，把comp设置为Equal，意思为只有当缓存中模板值为0相等时才会通过![4.png](https://i.loli.net/2020/04/05/72QYNH5Jn9AEsLh.png)

## 参考文献
 以上内容均来自本人观点以及网上资料,如果侵权，请联系删除，多谢观看。
 <font size=3>1. 参考文章---https://blog.csdn.net/u011047171/article/details/46928463#t4
 <font size=3>2.官方文档---https://docs.unity3d.com/Manual/SL-Stencil.html