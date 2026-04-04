Lyra的动画蓝图中比较有亮点的设计就是，使用了动画层接口进行复用其他动画蓝图中的逻辑。同骨架的两个动画蓝图只要继承了相同的动画层接口就能使用通过使用LinkAnimClassLayers这个接口给原ABP中的动画层接口的LinkedAnimLayer指定上对应的InstanceClass。这样会使用新指定上的AnimInstanceClass中的LinkedAnimLayer的逻辑。

![[AnimationLayer动画层#动画层的类型：]]

Lyra中LinkAnimClassLayers的操作是在激活武器的时候。

![[开源项目分析/Lyra/Lyra动画蓝图的结构/1.png]]

# 动画蓝图的结构拆分
## ABP_ManneQuin_Base
把这个Base动画蓝图称作主动画蓝图，这个ABP中的动画层都使用LinkedAnimLayer而不是普通的AnimLayer。上半部分是在这个ABP中调用的LinkedAnimLayer。下半部分是动画层接口中所有的方法，点开全部都是空实现。

![[开源项目分析/Lyra/Lyra动画蓝图的结构/2.png]]

## ABP_ItemAnimLayerBase
把这个动画蓝图称为动画层ABP，在这个动画蓝图中和主动画蓝图ABP_ManneQuin_Base继承同一个动画层接口。ABP_ManneQuin_Base中没实现的接口方法在这个动画层ABP中实现。但是在这个ABP中只有逻辑实现，没有指定具体的动画资产，而是将其作为变量。
![[开源项目分析/Lyra/Lyra动画蓝图的结构/3.png]]

## 动画层ABP的派生动画蓝图
动画层ABP的派生动画蓝图类分布在下面几个文件夹中。在这些动画蓝图分别代表不同持枪的动画蓝图
![[开源项目分析/Lyra/Lyra动画蓝图的结构/4.png]]

在这些动画蓝图中会给基类中的动画蓝图实际指定上资源。
![[开源项目分析/Lyra/Lyra动画蓝图的结构/5.png]]

整体流程如知乎上这张图。
![[开源项目分析/Lyra/Lyra动画蓝图的结构/6.png]]