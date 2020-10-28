---
title: MaxScript-初试
date: 2020-05-27 21:40:47
tags: [3DMax,Maxscript,插件]
categories: Develop
---
&emsp;**最近因项目原因，需要渲染一套3转2八方向的序列帧，因此手动操作实在太繁琐，所以就有了一个开发max插件的想法。**
<!--more-->
____
在经过一个星期的学习和测试后，讲一下作为美术人员对maxscript的理解，首先我想说，maxscript是非常适合美术人员去学习的一种编程方式，如果有一直想涉入编程或者了解程序世界的美术人员，maxscript非常适合入手。
主要理由呢，其一**maxscript对于编码格式非常宽松！不分大小写，没有固定格式要求，就是简单的写完一句就结束了。**
其二**Max内部的MaxScriptLisetener实在太便利了，进行测试，查找变量，查看属性，非常方便**
其三**MaxScript的api非常简单而且[maxscript官方帮助文档](https://help.autodesk.com/view/3DSMAX/2020/ENU/?guid=GUID-F039181A-C072-4469-A329-AE60FF7535E7)
里面非常详细**
然后说一下在制作八方向渲染脚本的原理和过程中遇到的的几个问题。
原理很简单，按钮控制渲染方法render camer：<camer>执行然后用一个for语句套一个旋转辅助体，和渲染多帧的for循环，进行，每次旋转45°后渲染X帧图像后保存然后继续转45°进行渲染保存。
____

第一个，在默认扫描线渲染器下进行渲染和输出非常简rander（）和outputFile，但是如果是Vray呢？查阅了[vray的帮助手册](https://docs.chaosgroup.com/display/VRAY4MAX/MAXScript)
有几个办法输出Vrayvfb
1.vr.output_splitbitmap，不能直接输出需要先创建位图对象然后赋值过去，然后保存下来，再去删除之前创建的位图
```
      tmpbm = Bitmap 10 10 fileName:vrayOutputFullPath
      save tmpbm
      close tmpbm
      vr.output_splitbitmap = tmpbm
      deleteFile tmpbm
```
2.vrayVFBGetChannelBitmap i，可以输出vfb的通道，索引从1开始，可以直接输出图片
```
      thebitmap = vrayVFBGetChannelBitmap 1
      thebitmap.filename = vrayOutputFullPath
      save thebitmap
```
这两种方法都必须用max quick render，不能使用render（）因为render（）是在max面板上的快速渲染之外创建的一个新的渲染方式所以没办法存储vray的vfb，max quick render就是F10的面板渲染器。
____
第二个，在outputfile输出range范围内渲染后，文件名后面会自动增加帧号后缀，没有查到能够禁止的方法，但是换了一种方式可以解决，就是按单帧进行多次渲染然后进行重新命名这样就可以控制文件名的尾缀了，虽然很麻烦。
____
第三个，edittext中输入，字符串然后转换成数组，我的做法是用filterstring过滤掉符号“,”然后execute把字符串转换成数值
```
    on frameN entered txt do(
      frame = txt
      frameArray = filterstring frame ","
      for f in frameArray do(
        renderframe = execute x
      )
    )
```
![微信截图_20200603174314.png](https://i.loli.net/2020/06/03/cWu2rlTEKgxY7CH.png)
## 参考文献
<font size=2>1.maxscript官方手册---<https://help.autodesk.com/view/3DSMAX/2020/ENU/?guid=GUID-F039181A-C072-4469-A329-AE60FF7535E7>
2.vray帮助手册---<https://docs.chaosgroup.com/display/VRAY4MAX/MAXScript>