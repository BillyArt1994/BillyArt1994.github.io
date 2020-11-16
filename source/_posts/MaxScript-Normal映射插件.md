---
title: MaxScript-法线传输
date: 2020-11-16 15:32:53
tags: [3DMax,Maxscript,插件]
categories: Develop
---
&emsp;**Maya中有一个内置功能，是把参考物的法线映射到目标物体身上，挺实用的，但是在3DMax中并没有找到这个功能，但是功能不难，用MaxScript也挺简单的**
<!--more-->
____
先画一个框，把一些需要用到的UI放进来，添加两个pickbutton，分别用来拾取参考物体(RefObject_btn)和目标物体(TagObject_btn)，我设置了button的初始enabled：false ，主要就是让用户能够按顺序执行，算是防呆操作把，再添加一些文字提示稍微整理下位置。
```
DataTransfer=newRolloutFloater "数据传输" 300 570 
rollout NormalTransfer "法线映射"(
    label FristStep "1." pos:[5,15] across:2
    pickbutton RefObject_btn"<<拾取参考物体>>" pos:[20,10] width:260
    label SecondStep "2.   先拾取参考物体，然后拾取需被修改的目标物体" pos:[5,40] 
    label ThirdStep "3." pos:[5,65] across:2
    pickbutton TagObject_btn"<<拾取目标物体>>" enabled:false  pos:[20,60] width:260
    button  ConvertNormal_btn"法线映射" enabled:false width:150 height:50
    )
    addRollout NormalTransfer DataTransfer
```
![MaxScript.png](https://i.loli.net/2020/11/16/kj3Pisp1czTClVt.png)
我这里是因为想制作多个功能，所以单独创建了一个卷展栏然后用addRollout方法添加到主浮框里面
其实如果没有这个需求的话可以直接创建一个浮窗即可。
```
rollout testDialog "testDialog"(
)
createDialog testDialog 150  500
```
接下来写一下拾取的功能，很简单就是在按下拾取按钮后改变下caption，然后这里的refobj是一个我声明的一个本低变量，用来接收拾取到的obj，然后把下一步的button.enabled打开
```
    on RefObject picked obj do(
        RefObject.caption = "参考 -> " + obj.name
        refObj = obj
        TagObject.enabled =true
     )
```
![MaxScript1.gif](https://i.loli.net/2020/11/16/2BczApQ3LdTfWw4.gif)
同理TagObject_btn也同样去处理
```
    on TagObject_btn picked obj do(
        TagObject.caption =  "目标 -> " + obj.name
        tagObj = obj
        ConvertNormal.enabled = true
    )
```
接下来就是核心逻辑代码了,我是这样想的,拿目标物体的顶点去与每一个参考物体的顶点做距离运算,算出相距最小的点,就是目标物体需要修改成的那个顶点法线。
按照思路,先去拿到目标物体的每一个顶点,然后存在数组里面。
声明一个数组tagObjetVert，然后convertToMesh把tagObj模型转换成EditableMesh，之后getNumVerts得到顶点总数，之后就遍历一遍用getVert得到每一个顶点的位置并且存进数组里面。
```
   on ConvertNormal_btn pressed do(
        tagObjetVert=#()
        convertToMesh tagObj
        num_tagObjVert = getNumVerts tagObj
        for i=1 to num_tagObjVert do (
            vertPos = getVert tagObj i
            append tagObjetVert vertPos
        )
```
同理也获得一下参考物体的顶点位置并存在数组里面。
```
        refObjectVert=#()
        convertToMesh refObj
        num_refObjVert = getNumVerts refObj
        for i=1 to num_refObjVert do (
        vertPos = getVert refObj i
        append refObjectVert vertPos 
        )
```
现在拿到了两个物体的顶点位置，接下来就是算距离了，然后找出最小距离的点并得到顶点的ID索引  
写一个for循环遍历目标物体的顶点，然后在里面再写一个for循环遍历参考物体的顶点  
用distance计算两点之间的距离，然后把参考物体的VertexID以及距离值一起存在一个二维的数组里面  
接下来用amin来找到最小值（不知道为什么MaxScript min()非要写成amin(),可能是因为max()被使用了= - =）
之后拿着最小值用findItem找到这个值的索引，然后用这个索引去找到对应的顶点ID
```
        for i=1 to tagObjetVert.count do (
            dis_refObj2tagObj = #(#(),#())
            for j =1 to refObjectVert.count do(
                append dis_refObj2tagObj[1] j
                dis =  distance tagObjetVert[i] refObjectVert[j]
                append dis_refObj2tagObj[2] dis
            )
            min_dis = amin dis_refObj2tagObj[2]
            index = findItem dis_refObj2tagObj[2] min_dis
            vertID = dis_refObj2tagObj[1] [index]
            append refVertID vertID
```
接下来最后一步了，因为VertexID已经拿到了，所以直接拿顶点法线然后赋值给目标物体就结束了
```
        for i=1 to num_tagObjVert do(
            refNormal = getNormal refObj refVertID[i]
            setNormal tagObj i refNormal
        )
```
测试一下,貌似是按照理想方向走的
![Normaltt.png](https://i.loli.net/2020/11/16/Lxb8rtCdMzHkX47.png)
git:https://github.com/BillyArt1994/3DMax-Pugins
---
这个插件在处理顶点量很大的模型的时候会卡死,优化空间还是很大的,以后有时间再想想看
## 参考文献
1.[Maxscript - Normal Thief脚本源码学习](https://zhuanlan.zhihu.com/p/48470962)  
2.[Maxscript文档](https://help.autodesk.com/view/3DSMAX/2017/ENU/?guid=__files_GUID_F039181A_C072_4469_A329_AE60FF7535E7_htm)  