先附上官方文档的链接[虚幻引擎中的动画蒙太奇 | 虚幻引擎 5.7 文档 | Epic Developer Community](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/animation-montage-in-unreal-engine#montagesectionspanel)

几个重要的点如下：
1. 在动画蒙太奇中，动画序列以 **插槽组（Slot Groups）** 和 **插槽（Slots）** 分类。一个插槽中可以有多个动画序列来生成动画，还可以使用不同的插槽来创建额外的轨道。什么意思呢？通常我们会在动画蒙太奇左创建并使用动画插槽 [[Animation Slot]] , 几乎所有的动画蒙太奇都要使用。没有Slot就无法在动画系统中播放Montage。在一个插槽中拼接多段Animation Sequence，这是常见的做法。==**在动画蒙太奇中可以使用多个插槽，每个插槽对应一个轨道，每个轨道可以拼接多个Animation Sequence。**== 如下
   ![[Animations/LearnALSV4/Animation Montage Media/1.png]]
   <br>在蒙太奇编辑器中我们只能预览一个轨道的动画。可以切换预览各个轨道的动画序列，但也只是在编辑器中预览。
   ![[Animations/LearnALSV4/Animation Montage Media/2.png|600]]
   <br>在游戏中如何呢，创建个测试用的蒙太奇，就用第三人称的模版里的内容。在蒙太奇中使用两个动画插槽。都是DefaultGroup这个插槽组的。
   ![[Animations/LearnALSV4/Animation Montage Media/3.png]]
   <br>还需要在动画图表中加上两个Slot，和Slot直接覆盖掉Animation Sequence 的pose的行为不同的是，两个Slot串联并同时触发时，两个Slot的蒙太奇是叠加混合的。也就是上面两个轨道的Animation Sequence都同时在播。
   ![[Animations/LearnALSV4/Animation Montage Media/4.png]]
   <br>这个特性要如何使用，我的意思是我的使用方式是否正确？官方文档中有介绍正确的使用方式。我们可以把一些相似的动画，但在不同的Slot上播放的组织到一个蒙太奇中。
   ![[Animations/LearnALSV4/Animation Montage Media/5.png]]<br>
2.  一个蒙太奇可以同时播放多个插槽上的动画，但是它们必须在同一个[插槽组](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/animation-slots-in-unreal-engine#slotgroups)中。上面我们刚好使用的是同一个插槽组，所以正常播放动画。想在一个蒙太奇上使用两个不同的插槽组是吧？不好意思编辑器会直接弹出警告。 ^7fad5d
   
3. 动画蓝图和蒙太奇相关的默认值使用关联[[分层动画蓝图]]
   ![[Animations/LearnALSV4/Animation Montage Media/6.png]]

# 动画蒙太奇的插槽中可以是叠加动画

通常我们在动画蒙太奇的插槽中用的是AnimationSequence，但也可以是Additive Animation类型的AnimationSequence。
**Slot 节点本身已经内置了处理 Additive 动画的能力**，不需要额外连接 `Apply Additive` 节点。Slot 节点的输入是普通的 Local Space FK 数据，输出也是。当你把 Local Space Additive 或 Mesh Space Additive 动画丢给它，它会自动把这些 Additive 叠加到输入上再输出。如果混合了不同类型的数据，Slot 节点也会正确处理 Blend Out。

| Slot 输入  | Slot 内 Montage 类型 | 输出结果                                  |
| -------- | ----------------- | ------------------------------------- |
| 普通动画     | 普通 Montage        | Montage 覆盖输入（Blend）                   |
| 普通动画     | Additive Montage  | 输入 + Additive 叠加                      |
| Additive | Additive Montage  | Additive A + Additive B               |
| Additive | 普通 Montage        | 取决于 bAdditiveAnimationsOverrideSource |

**特殊情况：强制覆盖模式**
Slot 节点有一个 `bAdditiveAnimationsOverrideSource` 选项，设为 `true` 时，即使传入的是 Additive 动画，Slot 也会把它当成普通动画数据来处理，执行覆盖（Switch）而不是叠加。适合子树中全部都是 Additive 数据的特殊场景。