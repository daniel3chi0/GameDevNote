[[Outline 从Character解析ALSV4结构|BackToOutline]]
手持道具的动画主要是在OverlayLayer这个动画层实现的，OverlayLayer输出的姿势会用于LayerBlending的输入。之前在[[手持道具到OverlayLayer#^1aed19]]OverlayLayer的部分有提及到。

# LayerBlending的输入
LayerBlending有三个输入：BaseLayer，OverlayLayer，BasePoses。

1. OverlayLayer的输出可以看做是一段动画序列Pose。[[手持道具到OverlayLayer#^b72a56]]
   
2. BasePose的内容就是一些Locomotion，所以逻辑上输出也可以看做是一段序列Pose。
   ![[Animations/LearnALSV4/LayerBlending层混合Media/1.png]]

3. BasePose这个Layer输出的是个静态单帧Pose，根据曲线来显示选择Pose[[手持道具到OverlayLayer#^0ef9f7]]
   ![[Animations/LearnALSV4/LayerBlending层混合Media/2.png]]

# LayerBlending内部结构
1. Input Cache
   进入LayerBlending节点后我们可以看到BaseLayer，OverlayLayer，BasePoses先是被缓存一个CachePose以便后面使用。
   
   ![[Animations/LearnALSV4/LayerBlending层混合Media/3.png|500]]

2. Make Dynamic Additives from Base Layer
   这里使用MakeDynamicAdditive节点动态生成一个Additive动画[[叠加动画]]，输入的Base是静态单帧Pose，BaseLayerInput则是序列帧Pose。
   ![[Animations/LearnALSV4/LayerBlending层混合Media/4.png]]
   这里用同样的输入做成两个叠加动画然后缓存起来，并加上后缀（MS）和（LS）代表MeshSpaceAdditive和非MeshSpaceAdditive（也就是LocalSpaceAdditive）区别如[[Aim Offset瞄准偏移#^ed75e8]]定义见[[骨骼动画变换的不同空间的含义]]
   
   虽然我们生成的叠加动画是是用一个（非叠加的动画的）序列帧Pose和一个静态单帧Pose相减得来的。但是一个叠加动画的输出其实也是Pose如果用叠加动画的序列帧Pose和一个静态单帧Pose相减的结果会是什么呢？
   结论是无论是ABP直接输出叠加动画 or 输出（叠加动画-单帧Pose)并应用（apply additive）到单帧Pose输出的表现都是有问题的，我自己的验证是角色模型直接在场景中消失了。这种错误用法可以写段测试代码看看。
   ![[Animations/LearnALSV4/LayerBlending层混合Media/5.png]]

> [!TIP] MS和LS使用
> 在注释部分作者补充MS和LS叠加动画用在身体的不同部分，LS类型最好使用在如Arms这样的地方以至于尊重Spine引起的旋转。
   
 3. Body Section Layer
    1. Legs Layer
       根据Layering_Legs曲线选择是把Base Layer Input还是Overlay Layer Input (+Slot Legs) + MS 缓存到Legs Pose。Layering_Legs曲线相关的动画序列或者蒙太奇都是腿部有动作的动画，腿部运动的时候曲线值更趋近-1，不运动的时候趋近于0。走跑都是-1。也有些值是1的动画序列，但都是差值类型为Step的。
       
       其实两条路径相比之下就多了个Overlay Layer Input，MS其实就是Base Layer Input 的运动量（BaseLayerInput - BasePosesInput）。
       ![[Animations/LearnALSV4/LayerBlending层混合Media/6.png]]
    2. Pelvis Layer
       和之前同理（骨盆运动的时候曲线值更趋近-1，不运动的时候趋近于0。差值类型为step的都是-1）区别是Slot名和曲线的名字。
       ![[Animations/LearnALSV4/LayerBlending层混合Media/7.png]]
    3. Spine Layer
       和之前同理区别是Slot名和曲线的名字。应用叠加动画的时候用SpineAdd的值作为权重。
       ![[Animations/LearnALSV4/LayerBlending层混合Media/8.png]]
    4. Head Layer
       和之前同理区别是Slot名和曲线的名字。应用叠加动画的时候用HeadAdd的值作为权重。
       ![[Animations/LearnALSV4/LayerBlending层混合Media/9.png]]
    5. Arm L Layer
       和之前同理区别是Slot名和曲线的名字并且使用LS叠加。应用叠加动画的时候用Arm L Add的值作为权重。
       ![[Animations/LearnALSV4/LayerBlending层混合Media/10.png]]
    6. Arm R Layer
       和之前同理区别是Slot名和曲线的名字并且使用LS叠加。应用叠加动画的时候用Arm R Add的值作为权重。
       ![[Animations/LearnALSV4/LayerBlending层混合Media/11.png]]
    Body Section Layer 做的事情就是把各个身体部位的相关的Montage片段提取出来保存成Cache Pose。此时还是带动全身的还没开始做分层混合，Slot总是加在Overlay Layer Input后面相关身体部位的Montage总是覆盖掉Overlay Layer Input这个输出序列。
    
    我看了这个骨架的大部分蒙太奇动画（其实是全部）的默认slot其实是MovementActionGroup.BaseLayer。如果要在上图对应的slot处插入蒙太奇我们要手动的去改Montage资产的Slot。
  
4. Blend Layer together
    1. Legs+Pelvis+Spine+Head
       把之前的缓存的身体片段的动画用Layered blend per bone节点进行分层混合。[[LayeredBlendPerBone分层混合]]
       ![[Animations/LearnALSV4/LayerBlending层混合Media/12.png]]
       <br>可以看下这个分层混合节点的设置，这里相当于pelvis骨骼和它的子骨骼参与混合使用BlendPose0，thigh_l左大腿骨骼和thigh_r右大腿骨骼以下使用BasePose（腿骨本身是参与混合的即BlendPose0），并且启用MS旋转，这里CurveBlendOption选择的是Override。
       ![[Animations/LearnALSV4/LayerBlending层混合Media/13.png]]
       <br>可以看一下骨骼的结构Pelvis的子骨骼有左右大腿和脊柱，从之前的混合深度得知盆骨以下全部用BlendPose0也就是CachePosePelvis，但是在左右大腿骨的时候被阻断子骨骼混合了，所以从大腿骨的子骨骼起使用BasePose也就是CachePoseLeg，而没阻断的部分就是spine_01和它的子骨骼即使用CachePosePelvis。
       ![[Animations/LearnALSV4/LayerBlending层混合Media/16.png]]
	   <br>依次检查后面两个同名节点的设置，类似上诉的实现就不一一描述了。
       ![[Animations/LearnALSV4/LayerBlending层混合Media/14.png]]
       ![[Animations/LearnALSV4/LayerBlending层混合Media/15.png]]
    
    2. Arm L (Mesh / Local Space)
       和上面差不多，多了用曲线值控制是使用MS还是LS的CachePose参与混合。多了用曲线的值控制混合的权重。
       ![[Animations/LearnALSV4/LayerBlending层混合Media/17.png]]
       
    3. Arm R (Mesh / Local Space)
       和上面差不多，多了用曲线值控制是使用MS还是LS的CachePose参与混合。多了用曲线的值控制混合权重。
       ![[Animations/LearnALSV4/LayerBlending层混合Media/18.png]]
       
    4. Hands
       正常手指骨有5个但这里多了个虚拟骨骼[[Virtual Bones 虚拟骨骼]]
       ![[Animations/LearnALSV4/LayerBlending层混合Media/19.png]]
       
    5. Blend AnimCurves from both layers together and override previous layer interference.
       这个节点没有填写混合的骨骼，代表混合不会应用到任何骨骼，但是曲线会。
       ![[Animations/LearnALSV4/LayerBlending层混合Media/20.png]]
       
       这里是根据这个vb_curve骨骼混合，最终输出姿势给后面的Layered blend per bone节点，但pose又被这个节点的设置不参与混合，只有曲线参与混合。这里的动画Slot的命名也是Curves代表输出只有曲线。[[Animation Slot]]
       ![[Animations/LearnALSV4/LayerBlending层混合Media/21.png]]