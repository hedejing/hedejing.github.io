---
layout: post
title:  "一个VR Funhouse的Mod"
date:   2019-01-29 18:13:23
categories: [Show Some Demos]
---

这是虚拟现实课的project，我用Nvidia提供的[VR Funhouse Mod Kit]在Unreal里做了一个[VR Funhouse]的Mod。[VR Funhouse Mod Kit]在Unreal里提供了一些[VR Funhouse]的blueprint、材质、模型之类的资源，我主要是用这些提供的东西重新实现了一个小游戏。

这个小游戏的基本玩法很简单（但是可能有点莫名其妙）：通过手柄按键可以射出球并“召回”，如果球击中场景中的标记了分数的木板，就能获得对应的分数。场景里有些木板是朝向后面的，需要利用球的召回曲线才能击中得分。

起初其实是想实现战神4/新战神里控制奎爷扔斧头并召回的解密小游戏，后来时间紧迫就只简单实现了类似的扔和召回的效果。为了强调召回的功能所以加上了一些只能通过召回击中的木板，所以不懂的话就会让人感觉有些莫名其妙，估计老师验收的时候也看不出玩起来有啥特别的。。。

[VR Funhouse]: https://store.steampowered.com/app/468700/NVIDIA_VR_Funhouse/
[VR Funhouse Mod Kit]: https://developer.nvidia.com/vr-funhouse-mod-kit

video:
<iframe src="http://pv35g2uxf.bkt.clouddn.com/vr.mp4" width="750px" height="500px" frameborder="0" scrolling="no" allowfullscreen="true"></iframe>

