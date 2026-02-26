Animation Slot即动画插槽通常被用来和 [[Animation Montage]] 动画蒙太奇一起使用，动画插槽在unreal动画系统中如果想要播放蒙太奇，是必须要设置这个节点的。需要在动画图表的output前加上。在动画蒙太奇的左侧可以设置。
如官方文档[虚幻引擎中的动画插槽 | 虚幻引擎 5.7 文档 | Epic Developer Community](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/animation-slots-in-unreal-engine)

文档中提到几个比较关键的点在平时开发使用的时候可能会被忽视：
1. 动画插槽的数据保存在骨骼中
2. ![[Animation Montage#^7fad5d]]
3. 插槽支持输入源姿势链接，并且在没有插入动画的时候让输入动画直接通过。只有当插槽中播放自己的动画时才会覆盖输入的动画。输入动画被覆盖时，它将不再更新。你可以在插槽的 **细节（Details）** 面板中启用 **固定更新源姿势（Always Update Source Pose）** 来保证源姿势持续更新。
   ![固定更新源姿势](https://d1iv7db44yhgxn.cloudfront.net/documentation/images/06a4386c-c104-4af3-b302-25e5555f833d/usage3.png)
   
4.  在Sequencer中使用
   插槽也可以在[Sequencer](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/cinematics-and-movie-making-in-unreal-engine)中使用，Sequencer是虚幻引擎的过场动画和影视制作工具。可以在Sequencer中使用插槽将过场动画插入动画蓝图，从而在角色动画中添加更多逻辑。
   
   插槽用于[动画轨道](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/cinematic-animation-track-in-unreal-engine)中的动画。要使插槽中的动画播放，右键点击一个动画分段并找到 **属性（Properties） > 槽位名称（Slot Name）**。输入要插入的插槽的名称。
   ![Sequencer中使用插槽](https://d1iv7db44yhgxn.cloudfront.net/documentation/images/4a83869a-2d27-4b3b-af59-279989ca1eae/sequencer1.png)一个常见的在Sequencer中使用插槽的用例为将过场动画和游戏动画混合在一起。更多信息可以参考[混合Gameplay和Sequencer动画](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/blend-gameplay-animation-to-cinematic-animation-in-unreal-engine)页面。
5. 插槽组不仅仅用于分类。当一个插槽正在运行时，再播放使用同组插槽的蒙太奇，正在运行的那个插槽便会被中断。这种自动的模式可以用于打断蒙太奇。例如，武器重装填动画蒙太奇可以被使用技能或者进战攻击的动画蒙太奇所打断。如果多个蒙太奇播放同插槽组中插槽上的动画，最近的蒙太奇则会打断前一个。