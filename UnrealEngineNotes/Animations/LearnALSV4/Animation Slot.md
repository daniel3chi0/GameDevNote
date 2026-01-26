Animation Slot即动画插槽通常被用来和 [[Animation Montage]] 动画蒙太奇一起使用，动画插槽在unreal动画系统中如果想要播放蒙太奇，是必须要设置这个节点的。需要在动画图表的output前加上。在动画蒙太奇的左侧可以设置。
如官方文档[虚幻引擎中的动画插槽 | 虚幻引擎 5.7 文档 | Epic Developer Community](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/animation-slots-in-unreal-engine)

文档中提到几个比较关键的点在平时开发使用的时候可能会被忽视：
1. 动画插槽的数据保存在骨骼中
2. ![[Animation Montage#^7fad5d]]
3.  在Sequencer中使用
   插槽也可以在[Sequencer](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/cinematics-and-movie-making-in-unreal-engine)中使用，Sequencer是虚幻引擎的过场动画和影视制作工具。可以在Sequencer中使用插槽将过场动画插入动画蓝图，从而在角色动画中添加更多逻辑。
   
   插槽用于[动画轨道](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/cinematic-animation-track-in-unreal-engine)中的动画。要使插槽中的动画播放，右键点击一个动画分段并找到 **属性（Properties） > 槽位名称（Slot Name）**。输入要插入的插槽的名称。

![Sequencer中使用插槽](https://d1iv7db44yhgxn.cloudfront.net/documentation/images/4a83869a-2d27-4b3b-af59-279989ca1eae/sequencer1.png)

一个常见的在Sequencer中使用插槽的用例为将过场动画和游戏动画混合在一起。更多信息可以参考[混合Gameplay和Sequencer动画](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/blend-gameplay-animation-to-cinematic-animation-in-unreal-engine)页面。