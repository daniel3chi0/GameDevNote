[[叠加动画]] [[骨骼动画变换的不同空间的含义]]
瞄准偏移是一种特殊的混合空间（ue4里也有把瞄准偏移封装成单独的资源）：使用到叠加动画资源的混合空间
ALSV4中瞄准偏移是用混合空间做的

![[Animations/LearnALSV4/AimOffset瞄准偏移Media/1.png]]

![[Animations/LearnALSV4/AimOffset瞄准偏移Media/2.png]]
<br>
进入这个叠加动画资源中查看

![[Animations/LearnALSV4/AimOffset瞄准偏移Media/3.png]]
<br>
再进入ue4官方的动画示例资源中的瞄准偏移资源中看下

![[Animations/LearnALSV4/AimOffset瞄准偏移Media/4.png]]

看下用到的混合采样的 叠加动画资源

![[Animations/LearnALSV4/AimOffset瞄准偏移Media/5.png]]


![[Animations/LearnALSV4/AimOffset瞄准偏移Media/6.png]]
<br>
虽然ue4有给瞄准偏移定义资产，而不使用混合空间那瞄准偏移究竟是否是混合空间呢？
看下代码就知道，瞄准偏移混合空间就是混合空间的子类

![[Animations/LearnALSV4/AimOffset瞄准偏移Media/7.png]]
<br>
这两个函数是分别判断，附加类型和混合采样集是不是旋转偏移网格空间

![[Animations/LearnALSV4/AimOffset瞄准偏移Media/8.png]]
<br>
旋转偏移网格空间在编辑器中显示的就是 简单的网格空间

![[Animations/LearnALSV4/AimOffset瞄准偏移Media/9.png]]
<br>
就是叠加动画的网格体空间

![[Animations/LearnALSV4/AimOffset瞄准偏移Media/10.png]]
<br>
看下局部空间和网格体空间在官方文档中的描述
这在射击游戏中的应用就是让角色瞄准鼠标方向而不是动作的方向
如图中瞄准的时候按侧偏键，不至于角色瞄准的方向被侧偏方向带偏。 ^ed75e8

![[Animations/LearnALSV4/AimOffset瞄准偏移Media/11.png]]

在编辑器中新创建瞄准偏移

![[Animations/LearnALSV4/AimOffset瞄准偏移Media/12.png]]

添加混合采样集
![[Animations/LearnALSV4/AimOffset瞄准偏移Media/13.png]]

由上所述，用瞄准偏移混合空间执行以上的操作和混合空间的效果其实一样
只是用瞄准偏移使用的蓝图结点ue4封装过使用起来更简练。
这个预览基础动作是动画序列，混合采样集中的是叠加动画。

<br>
使用瞄准偏移的结点

![[Animations/LearnALSV4/AimOffset瞄准偏移Media/14.png]]

使用混合空间节点实现瞄准偏移需要如下

![[Animations/LearnALSV4/AimOffset瞄准偏移Media/15.png]]