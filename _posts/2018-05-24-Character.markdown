---
layout: post
title:  "Next Generation Character Rendering"
date:   2018-05-24 18:13:23
categories: [Share My Thoughts]
---

这是2013年动视暴雪在GDC上给的一个talk，讲诉了他们实现人像绘制的一些技术细节。内容主要包含Skin Rendering、Eyes Rendering以及Post Fx三部分。

SKIN RENDERING
==============

皮肤绘制效果主要由三部分组成，GI（全局光照）、specular
reflection（镜面反射）以及SSS（次表面散射）。下面先从specular
reflection开始介绍。

![](/images/Character/7557cb96fb93b0a932367f91a36e3fb4.png)

SKIN REFLECTION
---------------

一般的specular计算模型会使得人脸看起来像塑料，而他们针对人脸的specular计算做了一系列改进。

首先，对于procedural lights，使用基于物理的specular
brdf计算specular；对于环境光，使用pre-filtered cube
map。使用基于物理的brdf能够保证不同光照环境下效果的正确性，减少美术手动调节的工作量。

基于物理的着色计算已经被广泛使用，为了进一步改进人脸的specular效果，他们参考了一篇人脸扫描并绘制的paper，最终在不需扫描人脸的情况下做到了接近这篇paper的效果。

![](/images/Character/1a198fc4e79a2888247c4aee4229fdcb.jpg)

他们从这篇paper中了解到一个重要结论：人脸的specular
brdf是有两个lobe的。下面左图是一般方法绘制的效果，而右图是paper中使用扫描结果fit出来的brdf绘制的效果。对比可以发现右图比起左图多一个明亮层次，用只有一个lobe的specular
brdf是无法表现这种效果的。

![](/images/Character/0e9ff9ac0ce2bc54314e102610496013.png)

为了实现这种两个lobe的效果，他们给定一个公式从原来的lobe计算出第二个lobe，并且将两个lobe的specular结果进行插值作为结果。

![](/images/Character/ad66b83b1f5f05dfbdaad7b1fe758299.png)

虽然要计算两个lobe的到的specular，但是可以在shader代码中将相关标量合并到一个vec矢量中同时计算，这样实现两个lobe所增加的计算量就很少。

另外，那篇paper不仅用皮肤扫描结果来fit
brdf，还用扫描结果作为纹理表现人脸上的皱褶。为了把皮肤表面的微小褶皱也表现出来，他们用一个tiled
sinusoidal noise去扰动原来的base（normal）map。

<p align="center">
<img src="
/images/Character/65796e8cc9a3d0e2856cc1521d500969.jpg">
</p>

下面是是否使用noise扰动得到的绘制结果对比，可以看出右图将皮肤的皱褶很好地表现出来。

![](/images/Character/8b506268425ed473366bf91ff3673fd7.png)

这个noise组成的detail map用一系列正弦波的和生成，下面是生成公式：

![](/images/Character/84307370107c548d28b05d053977023d.jpg)

以一张1024\*1024的detail map为例，px和py表示[1,
1024]之间的点。减去的rand()表示每个正弦波的中心位置，而频率也是由这个rand()随机生成。用100个这样的正弦波求和得到一张height
map，然后用NVIDIA Texture Tools转成normal map。

<p align="center">
<img src="
/images/Character/aac8992732afa7c80b212ba465175689.jpg">
</p>

得到detail map后，使用Reoriented Normal Mapping 完成base Map 和detail
Map之间的blend，使得扰动的结果能够很好地保留二者的细节。Reoriented Normal
Mapping与其他normal map blend算法的比较具体可见[1]。

要表现皮肤的microgeometry，roughness的使用也很重要。从远处看，可以认为microgeometry
跟microfacet一样，而这里的detail
map对应的就是microfacet模型里面的roughness。于是我们可以用normal
variance的估计值当作glossiness，这样可以避免远处看时detail
normal因为mipmap而变得平滑的问题，同时将detail
normal信息以另外一种形式参与到specular的计算中。用variance修改glossiness的公式如下：

<p align="center">
<img src="
/images/Character/2541271f87f3a73dfa8c3b6e003b34d1.png">
</p>

这部分内容可参考[2]。

HUMAN PORES
-----------

之前的绘制方法都忽略了毛孔的作用，但是他们通过对比发现如果不考虑毛孔的话，毛孔的位置会出现不应该有的高光。

<p align="center">
<img src="
/images/Character/e50635defa495cd21b5084f82b3e613b.jpg">
</p>

所以，他们做了一张cavity
map来表示这些孔的位置，用作一种occlusion与specular的结果相乘，去掉这种不正常的高光。

<p align="center">
<img src="
/images/Character/a76fac2ae5faaa244a44484fa615a641.jpg">
</p>

这种方法在场景光比较均匀的情况下可以得到很好的效果，但是在光从侧面打到人脸上时，侧面应有的高光会变暗很多，出现一种不自然的噪声一样的效果。如下图所示。

![](/images/Character/055010182424239032f4ab78f37c8407.jpg)

出现这种错误是因为在直接相乘的作用下，毛孔的occlusion效果跟view方向无关。而实际上，如果从一个比较斜的角度看毛孔，它的occlussion作用是比较小的。如下图所示：

![C:\\Users\\Jorge\\Desktop\\GDC\\Figures\\Cavity.JPG](/images/Character/934901a08de590cc7354e93a1edd7af8.jpg)

从这一点出发，将specular的计算公式改一下就能解决这个问题：

原先：cavity \* Specular(gloss) \* Fresnel(reflectance)

改为：Specular(gloss) \* Fresnel(cavity \* reflectance)

以下是修改公式后的结果：

![](/images/Character/ec346b805e9c63641823a2b270cd5959.jpg)

(FAKING) GLOBAL ILLUMINATION
----------------------------

他们在这一节提出的观点是，如果使用了physically based的HDR light
probe的话，occlusion会很重要。下面先从没有occlusion的结果开始，逐渐增加各项occlusion来体验效果的差距。以下是没有任何occlusion的结果。

![](/images/Character/811aa436de99a9f3f9002698aa363b9c.jpg)

加了HBAO以后，高光太亮的地方变暗，特别是眼睛的部分。但是还不够。

![](/images/Character/80089d23ad833210890c4535efd65ebb.jpg)

于是他们又引入了baked
occlusion，效果又稍微变好，但是边缘的高光还是太亮了（因为Fresnel）。

![](/images/Character/5c6e0f47cf3a9c13211297851a15d4f6.jpg)

引入一个很强的specular
occlusion，边缘的不正常高光有所减弱，但效果提升不是很明显。

![](/images/Character/fe507487e74041702844d614871a813c.png)

用bent normal才解决了这个问题。

![](/images/Character/7fcd54db423d3560bd977e979836ed04.jpg)

最后引入了color-bleed
AO，使得occlusion部分的颜色不那么黑，与lighting的结果更一致。

![](/images/Character/42437eb2e40463415ff1dd81307bdf73.jpg)

下面开始讲一些实现细节。首先是SSAO和Baked AO的组合。

SSAO质量较低，但是可以用在会动的部分；Baked
AO质量较好，但是只能用于静态的部分。他们用了一个min操作将二者结合起来：

>   finalAO = min(BakedAO, SSAO)

这样一来，人像在动画状态下如果某处的SSAO的值特别小，就会替代掉BakedAO的作用。

另外，上面提到了作用到specular结果上的Power Specular AO。Power Specular
AO可以用一定经验公式从直接换算得到[3]，但是他们找到了一种更好的方法在实时绘制中进行换算，计算方法如下：

![](/images/Character/27dcda0f0c7b80875d773d55a2505c8b.png)

这么计算的理念是：specular的遮挡只会在从侧面看时发生，所以他们用NdotV相关的值对ambient
occlusion进行了一个指数运算作为Specular
AO的值，这样的效果更好。除此之外，为了提升specular项的occlusion效果，他们每个顶点都baked了bent
normal来代替原来顶点normal。

下一个问题是，一般的ambient
occlusion会让结果看起来太暗。这是因为occlusion的区域还会接受到遮挡物的间接光照，使得遮挡区域的颜色偏红。

![http://www.people-results.com/wp-content/uploads/111901291-ear-2.jpg](/images/Character/bd271f978084a7c8353d9bdd3f96dc5b.jpg)

他们使用Color-Bleed AO来解决这个问题。他们用了一个多pass ambient occlusion
render来计算Color-Bleed AO，计算流程如下：

![](/images/Character/532a860e2c9234983c860dd615d2a654.png)

下面是一般AO跟Color-Bleed AO的一个对比：

![](/images/Character/3f68ed36f252b6976e77407815c842b9.png)

可以看到Color-Bleed
AO的结果跟一般AO的结果相比，颜色偏红。于是他们直接将Color-Bleed
AO的计算合并到一般AO的gamma矫正计算中，计算公式如下：

>   colorBleedAO = pow(regularAO,1.0 - strength)

例如，某一个模型的strength为

>   strength = float3(0.4,0.15,0.13)

SUBSURFACE SCATTERING
---------------------

次表面散射Subsurface
Scattering（SSS）对人脸绘制的效果至关重要。下面分成两部分来讲诉SSS的实现，一是reflectance
subsurface scattering，使得皮肤表面的光照更加柔和；二是transmittance subsurface
scattering，使得光可以透过诸如耳朵这样的比较薄的区域。

下面是有无reflectance subsurface scattering的效果对比。

![](/images/Character/b4f00730bd9ed251a90d37c7d61b6958.png)

![](/images/Character/2f6891cce98310645da70e65d4761e14.png)

传统的绘制方法假设光在同一个点入射出射，即BRDF。然而对于皮肤这种通透性的材质，我们需要使用BSSRDF模型，考虑光在一个点入射后会在相邻点出射的情况。由于BSSRDF是八维的，一般会使用diffusion
profile简化计算。在BRDF的基础上，使用一个距离参数输入diffusion
profile便可以估算出一个点对于另一个点光照的影响。

![](/images/Character/028d47ad0906ecb064c892fcd1bf3e6b.png)

有了diffusion profile之后，以它为kernel在texture space对irradiance
map进行二维卷积便可得到考虑reflectance subsurface
scattering的光照效果。但是对于每一个物体，都要在texture
space进行一次二维卷积，这样的计算量太大了。于是，有人提出用多个Gaussian
filter对diffusion profile进行拟合，由于一个Gaussian
filter的计算可以拆成两个pass在不同单个方向上的卷积，所以可以减少计算量。但对于游戏来说，这样的计算量还是太大了。可以再在两个方面减少计算量，一是在screen
space进行卷积[4]，
二是使用比Gaussian拟合更高效的方法对profile进行两个方向上的拆解，[5]中提出了直接将profile的二维卷积直接拆解为两个水平方向上卷积的方法。

使用以上的优化能够带来很大的效率提升，但是在近处看会出现规律性的条纹artifact。

![](/images/Character/ffdb9b5e48585233c337319d022e612d.png)

为了解决这个问题，他们借鉴了前人在SSAO上的工作[6]，给逐像素的计算中的卷积加上随机的旋转，从而消除这种交叉pattern。

![](/images/Character/a880b67effbb7c2b6d89d334700616f7.png)

SSS中reflectance的部分到此结束，接下来是transmittance的部分。这部分是在他们已有的工作基础上实现的[7][8]。下面要讲到的方法分为三类：single-sample
transmittance[7]、改进的multi-sample transmittance以及用于球谐光照的spherical
harmonic transmittance。

![](/images/Character/29dccf4f486ae79fe1577d6a2ccbbb9c.png)

在之前的工作中[7]，光透过一个物体走过的距离使用shadow
mapping来计算。如上图所示，利用shadow
mapping可以计算出Zout和Zin之间的距离S，以此可以计算出散射的程度。光在皮肤中透过的距离越长，散射效果越强，所以到最后形成的光路不是一条直线，呈现出diffused特征。然而，由于我们只采了一个样本点计算透射距离，所以最终的光照结果不会那么diffused。如下图所示，耳朵处的透射结果出现明显的边界。

![](/images/Character/db0df2e9a8c3a55200a9fa6fa1cffc73.png)

解决方法是采更多的样本，计算multi-sample
transmittance。这样可以提升效果，但同时多次采样会带来更多计算量。对此，类似软阴影技术，我们可以用Poisson
disk sample和逐像素的jitter
rotate来减少需要采样的个数。尽管如此，要达到好的效果，需要16次shadow采样，计算量依然会很大，而过少的采样数又会带来noise的结果。但由于我们最后需要为reflectance做一次SSS
Blur，所以我们可以接受稍微noise的transmittance结果，在采样次数上采取折中的方案。下面是single-sample和multi-sample的结果对比，可以看出multi-sample的结果要柔和很多，看起来更加真实。

![](/images/Character/dfd7604b35684ea8762ec7eb75208232.png)

![](/images/Character/32c8a0d9ed526237f0e0242c07b9fdd8.png)

使用球谐光计算transmittance也是可行的，但是使用2个带（SH4）的球谐光可能会产生一定程度的漏光。他们将multi-sample
transmittance的结果bake到SH4，储存为8-bit的RGBA纹理。下面是各个方法的对比：

![](/images/Character/0c364590b8e335f29e51d66dc8a665f3.png)

EYE RENDERING
=============

眼睛绘制的特点有很多，下面会逐点进行分析。

![](/images/Character/faf5f594dd82b5546cf9b179ee20e676.png)

REFLECTION
----------

首先， 眼睛可以分为pupil、iris、sclera和caruncle几个部分。

![](/images/Character/3b77134ea46b32229666e3b7ad9da396.jpg)

眼睛表面的反射效果是主要的特性。根据观察，他们发现sclera部分的反射呈现出sine-like的扰动，而其余部分则接近于镜面反射。

![](/images/Character/68d62f6fa1e46b9753192410ec1a664f.png)

于是，跟 detail map的做法一样，他们用正弦波生成用于眼睛的normal
map，只是中间的圆形区域normal保持（0，0，1）从而表现镜面反射。

![](/images/Character/bf8ebc0ed72ca657854d22c1ea1d5fdb.tmp)

WETNESS
-------

另外，眼睛的湿润程度对反射效果的影响很大。他们采集并对比了眼睛在不同湿润程度下的照片，发现越是湿润，sine-like扰动对应的length/波长和amplitude/振幅越大。

![](/images/Character/eb07b1780b1894629e9f58439fa02aaf.jpg)

于是将第二张波长和振幅更大的正弦波生成的normal map用reoriented normal
map的方法与第一张normal做混合。这样一来，第二张normal
map的0到1之间的intensity就可以作为眼睛的wetness让美术人员进行调整。

但是随着wetness改变反射的效果还不够，eyelids/眼睑部分的高光也会随着wetness变化。为了表现这种变化，他们做了三档不同的geometry，根据wetness做blend得到最终使用的geometry。

![](/images/Character/555ad29a77f6bea702b741742fa85d4d.jpg)

REFKECTIONS OCCLUSION
---------------------

另一方面，眼睛的反射效果应该受到眼睑和睫毛的遮挡。为此他们bake了一张reflection
occlusion map用于表现这种遮挡效果。

![](/images/Character/5d8d8b5dd2a3e27d0447c2c1f832e9f7.jpg)

使用之后的效果还是比较明显的，会有真正的“嵌入”的感觉。

![](/images/Character/cfd4d25dd5ebe1057c7a74a5d525838a.png)

![](/images/Character/e5431ee74f3ad7171fc6650f75af4631.png)

SCREEN-SPACE REFLECTIONS
------------------------

在光照不太均匀的情况下，局部结果的反射效果很重要。他们尝试将screen-space
reflection[9]用在眼睛绘制上，发现能够使眼睛的光照结果变得与皮肤绘制的光照结果更加一致，进一步提升真实感。

![](/images/Character/36d8338c8b032797b212671be8b6570a.png)

![](/images/Character/d1df74d477d1fbebdf4926bba6dd2f87.png)

VIEW REFRACTION
---------------

眼睛的折射效果是另外一大重要特性。如果不考虑折射效果的话，会有像牛一样盯着前方的不自然的感觉。

![](/images/Character/9bc35ddc08cdeb894e4d22108e253c87.png)

![](/images/Character/702cc67a68f2026f93d1a48665f4d1c1.png)

他们用两种方法来表现折射效果，一是用parallax
mapping，二是基于物理的方法（看计算过程其实还是类似parallax
mapping，只是多考虑了折射带来的出射方向改变）。

![](/images/Character/cff4182e7a17de64248be27d752d5685.png)

LIGHT REFRACTION
----------------

但是，前面仅考虑了view/出射光的折射效果，还没有考虑light/入射光的折射效果。而如果考虑入射光的折射效果，iris周围的一圈会出现自然的阴影效果，中间的部分会更亮，看起来更加真实。

![](/images/Character/3e936bec2a8f4eb79af8cc7715e3ecf2.png)

![](/images/Character/60da1647f568a5c3e217b6688654f6cc.png)

为了表现这种效果，他们将不同角度入射光的折射结果用photon
mapping的方法预计算并存到3d
texture上，每个slice对应一个角度的结果。如果光源是light
probe的话，对应的就是各个方向的积分结果。

![](/images/Character/5ddb9f4b269a066ad43c1ba16cf34380.png)

REDNESS SHADING
---------------

在眼睛干枯或者流泪的时候，眼睛的上的血管和虹膜会泛红。为了动态地表现这种变化，他们使用veins
map将血管和虹膜区分开来，分别对颜色进行不同程度的红色偏移。

![](/images/Character/6586450fcbbe181624ae6b28dc575723.png)

POST FX
=======

近年来，建模精度和屏幕分辨率得到了很大的提升，画质因此实现了飞跃。然而这带来了问题——生成的画面太过精美了，反倒显得不太真实。后处理正是解决这个问题的办法。在现实世界中，光在传输过程中会因为散射、景深、动态模糊以及光晕等原因变得模糊，例如散射使得光到达眼睛前经历多次反射，使得光照更加diffuse。除了模糊，真实画面与生成画面的另一个差距是噪点。在光照比较弱的情况下，更少的光子到达相机的传感器，使得光传播的随机性更加明显，最终使得画面噪点增多。下面是有无后处理实现景深的对比：

![](/images/Character/b7d03ce2d7356232bdad466c25e9fd95.png)

![](/images/Character/ace95067ba2da7ab39c14f2a65ff3571.png)

而下面是有无后处理实现底片颗粒感的对比：

![](/images/Character/7119d3a62daebe8f3874d2f02441befe.png)

![](/images/Character/906cb3ef171d4f25f80aa43a1988a78d.png)

锯齿也是破坏真实感的因素之一，他们使用SMAA T2x[10]实现抗锯齿。SMAA
T2x是改进的形态学坑锯齿方法，利用时域上的超采样获得时域上的稳定性。由于SMAA并不是比较通用的AA技术，所以我不在这里详细说明，有兴趣可以看原ppt进行了解。

他们使用简单的后处理模糊来实现景深，模糊半径用像素的circle of
confusion/弥散圆来衡量。如果每个邻居点都以相同权重参与模糊计算，会产生bleeding/颜色渗出。通过对比深度值，可以避免不聚焦的背景像素渗出到前景中；在背景像素的计算中，可以根据circle
of
confusion来给予邻居点不同的权重，从而避免前景像素渗出到后景中。但是用上这两种优化方法，效果依然不够好。景深本应使得物体的边界往外扩，但是由于只是进行了颜色模糊，于是在边界上出现类似光晕的效果。为了解决这个问题，他们在计算过程中比较每个邻居点的circle
of confusion以及邻居点的offset，只有circle of
confusion大于offset才参与模糊计算，而这一步比较又可以简化为saturate的circle of
confusion与offset的差。

![](/images/Character/0920bea2445f0422b1a961bb4ff5c231.png)

![](/images/Character/ea796745a8c9b3b78e1e40be93208c11.png)

最后，他们发现大多数后处理效果都用到了深度和颜色，包括次表面散射、SMAA、景深以及动态模糊。于是可以将深度的倒数储存在color的alpha通道中，从而减少shader之间的数据传输。由于大部分时候用到的都是深度的倒数而不是深度本身，所以选择储存深度的倒数，这样就可以使用LDR的帧缓冲来储存（因为LDR帧缓冲储存的值的取值范围限制在0到1，无法储存取值更大的深度值）。

参考资料
========

1.  *http://blog.selfshadow.com/publications/blending-in-detail/*

2.  [Hill2012] Rock-Solid Shading: Image Stability Without Sacrificing Detail

3.  [Gotanda2011] Real-time Physically Based Rendering – Implementation

4.  [Jimenez2009] Screen-Space Perceptual Rendering of Human Skin

5.  [Jimenez2012] Separable Subsurface Scattering and Photorealistic Eyes
    Rendering

6.  [Huang2011] Separable Approximation of Ambient Occlusion

7.  [Jimenez2010] Real-Time Realistic Skin Translucency

8.  [Jimenez2012] Separable Subsurface Scattering and Photorealistic Eyes
    Rendering

9.  [Kasyan2011] Secrets of CryENGINE 3 Graphics Technology
