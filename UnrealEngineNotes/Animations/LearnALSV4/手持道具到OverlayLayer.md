[[Outline 从Character解析ALSV4结构|BackToOutline]]
角色蓝图子类ALS_AnimMan_CharacterBP中，HeldObjectRoot，SkeletalMesh，StaticMesh 这个三个组件是用来做手持道具的。

在Tick事件，先执行基类中的逻辑，然后运行UpdateColoringSystem和UpdateHeldObjectAnimations，这两个函数。UpdateHeldObjectAnimations比较重要是我们手持道具的入口。

![[Animations/LearnALSV4/手持道具Media/8.png]]

## 弓
### 根据拉弓动画表现设置动画曲线的值带动弓的形变
如果Overlay State是弓状态，会提取主动画实例（Mesh上的ABP，在角色基类Beginplay的时候赋值）中的Enable_SpineRotation曲线的值然后设置给SkeletalMesh上的弓的ABP上的变量。

![[Animations/LearnALSV4/手持道具Media/9.png]]

![[Animations/LearnALSV4/手持道具Media/10.png]]


我们可以思考下为什么这样做，通常角色在拉弓的时候会有一个拉的程度，因为弓是SkeletalMesh可以有动画弓就可以接受这个形变的值从而表现动画。
弓的ABP中的逻辑很简单，把传入的Draw值给到SequenceEvaluator中，Draw的值越大代表弓的弧度越大。

![[Animations/LearnALSV4/手持道具Media/11.png]]


这个曲线是在ALS_Props_Bow_Poses这个AnimationSequence中定义的，当然别的AnimationSequence中也有定义。

一条曲线在不同的AnimationSequence中的值可以用户自己编辑。这里最相关的就是ALS_Props_Bow_Poses。

曲线名为Enable_SpineRotation（脊柱的旋转），拉弓的程度和脊柱的旋转正相关。

![[Animations/LearnALSV4/手持道具Media/12.png]]


这个Sequence是和正常的的Sequence不太一样，帧与帧之间没有过渡而是跳变。正如名字所述更像是一组Pose。

![[13.avif]]


因为这个Sequence的动画插值方式是用Step

![[Animations/LearnALSV4/手持道具Media/14.png]]


改成Linear后就会像正常的AnimationSequence一样平滑过渡了

![[15.avif]]
这条曲线在这个Sequence中的值是在[0,1]内跳变的。0代表脊柱基本无旋转（但在实际动画中某些时刻是有旋转的）1代表脊柱的最大旋转状态。


特殊情况如下这里Enable_SpineRotation的值是0，Enable_HandIK_R从0跳变到1（右手的IK变了）。
Layering_Arm_R_Add从1跳变到0（右手臂的变化，引起侧身的产生）

![[Animations/LearnALSV4/手持道具Media/16.png]]

### 拉弓动画在哪使用？
这段Sequence是怎么使用的？很明显不可能用SequencePlayer。
这段Sequence在主ABP中分别被Bow，BowRelaxed，BowAiming，BowReady这几个State通过SequenceEvaluator使用。
我这里先给出结论：用单帧动画和同步组叠加动画Apply Additive形成特色的Pose序列


先看下Bow(State)，这个Bow(State)位于OverlayLayer（AnimGraph）中的OverlayStates（StateMachine）

![[Animations/LearnALSV4/手持道具Media/17.png]]

![[Animations/LearnALSV4/手持道具Media/18.png]] ^4f4249

#### 1.Bow状态姿势混合
内部使用Int混合姿势节点进行混合

![[Animations/LearnALSV4/手持道具Media/19.png]] ^a0db39


##### 1. Bow状态Pose0——BowStates(State Machine)[[#^a0db39]]
Pose0是Bow的状态机（OverlayStates中又套了个BowStates），状态机内部是Bow的准备，松开，瞄准三个状态之间的过渡。这几个状态也就是使用到ALS_Props_Bow_Poses的SequenceEvaluator的地方了。

![[Animations/LearnALSV4/手持道具Media/20.png]] ^cf61bf

- 几个状态的过渡规则：

==BowRelaxed——>BowReady== : 需要满足RotationMode == Aiming[[#4.RotationMode]]

==BowReady——>BowRelaxed== : 需要满足（BowReady的CurrentStateTime > 3s && RotationAmount曲线值 == 0）||（BowReady的CurrentStateTime > 3s && IsMoving[[#1.IsMoving变量]]）||（Gait == Sprinting || MovementState == InAir[[#2.Gait和Movement变量]]）

==BowReady——>BowAiming== : 需要满足RotationMode == Aiming[[#4.RotationMode]]

==BowAiming——>BowReady== : 需要满足RotationMode != Aiming[[#4.RotationMode]]

- 状态内部的表现逻辑 ：

###### 1.IsMoving变量
>IsMoving从字面上看就是判断角色是否在移动的一个bool变量。是在ABP的Tick事件中通过UpdateCharacterInfo这个函数进行更新的。
<br>![[Animations/LearnALSV4/手持道具Media/21.png]]<br>
展开这个函数，IsMoving是从Character上实现的接口函数中拿到数据的。这里有两个接口函数分别是BPIGetEssentialValues（获取基本值）和BPIGetCurrentStates（获取当前状态）。
<br>![[Animations/LearnALSV4/手持道具Media/22.png]]<br>
BPIGetEssentialValues（获取基本值）和BPIGetCurrentStates（获取当前状态）这两个接口函数在ALS_AnimMan_CharacterBP（小蓝人）中是没实现的，是在基类中实现的（火柴人）中实现的。
<br>![[Animations/LearnALSV4/手持道具Media/23.png]]<br>
<br>![[Animations/LearnALSV4/手持道具Media/24.png]]<br>
这里BPIGetEssentialValues中的IsMoving是在Character的Tick事件中通过SetEssentialValues判断角色的Velocity的值是否大于0后设置。
<br>![[Animations/LearnALSV4/手持道具Media/25.png]]<br>
###### 2.Gait和Movement变量
>BPIGetCurrentStates中有我们最后一个条件中需要用到的变量Gait和MovementState的获取。Gait和MovementState变量都是在ALS_Base_Character中实现的接口函数中赋值的，分别是BPISetMovementState和BPISetGait。
<br>![[Animations/LearnALSV4/手持道具Media/27.png]]<br>
这里用到项目用BlueprintMarcoLibrary分装的宏ML_IsDifferent(Byte)，这个宏只是起到一个条件语句的作用。
<br>![[Animations/LearnALSV4/手持道具Media/28.png]]<br>
真正改变值用到的都是后面的OnXXChanged函数。
<br>![[Animations/LearnALSV4/手持道具Media/29.png]]<br>
后面的逻辑不是重点，但我还是简单介绍下吧。
如果MovementState是在空中就需要再判断MovementAction，如果是Rolling就需要做布娃娃运动。[[#3.Ragdoll流程]]
如果不是Rolling就需要判断站姿，如果是下蹲的要取消下蹲，Uncrouch是引擎中函数。如果Movement State是Ragdoll，检查前一个状态（PreviousMovementState）如果是Mantling就Stop掉这个MantleTimeline。(避免这个timeline中的数值影响到Ragdoll)
![[Animations/LearnALSV4/手持道具Media/30.png]]

###### 3.Ragdoll流程
>布娃娃运动分成3步：1.把MovementMode设置成None。2.Capsule的碰撞关闭，Mesh的碰撞对象类型设置成PhysicsBody，碰撞起用类型设置成查询和物理启用，设置盆骨以下模拟物理。3.动画蓝图停止蒙太奇。
<br>![[Animations/LearnALSV4/手持道具Media/31.png]]<br>
<br>![[Animations/LearnALSV4/手持道具Media/32.png]]<br>
OnXXChanged函数都用到了这个ML_SetPreviousAndNewValue宏，设置最新值和缓存旧的值。
<br>![[Animations/LearnALSV4/手持道具Media/33.png]]<br>

###### 4.RotationMode
>RotationMode的改变是在角色蓝图的基类（ALS_Base_CharacterBP）中的根据输入来反馈调用类中实现的BPI_SetRotationMode接口赋值RotationMode，比如把RotationMode设置成Aiming如下
<br>![[Animations/LearnALSV4/手持道具Media/37.png]]<br>
这里用的是旧版的输入系统，对应的是鼠标右键。
<br>![[Animations/LearnALSV4/手持道具Media/38.png]]<br>

##### 2. Bow状态Pose1——Mantle 1M Pose[[#^a0db39]]
Pose1是我们的曲线作用序列ALS_Props_Bow_Poses的SequenceEvaluator，ExplicitTime是0所以只是一个默认的站立姿势。作者给了个Mantle 1M Pose的注释，Mantle是指一种常见的动作捕捉术语，描述角色抓取边缘并翻越障碍物的动作（如从低矮物体上爬起或翻越围栏），Mantle 1M Pose 则是针对1M高的障碍物的的攀爬动作。

![[Animations/LearnALSV4/手持道具Media/34.png]]


##### 3. Bow状态Pose2——LandRollPose[[#^a0db39]]
Pose2 是在ALS_Props_Bow_Poses的0.11s的Pose的基础上应用对Layering_Arm_L_Add这条曲线的修改，数值为0.5。应该是类似0.9s的这个Pose一样，对应的是站立拿弓，但作者给出的名字是LandRollPose。

![[Animations/LearnALSV4/手持道具Media/35.png]]


##### 4. Bow状态Pose3——GetUpPose[[#^a0db39]]
Pose3是ALS_Props_Bow_Poses中0.28s的Pose，对应的是蹲拿弓正准备起身的Pose。

![[Animations/LearnALSV4/手持道具Media/36.png]]

#### 2.Bow Relaxed (State)
该状态位于[[#^a0db39]]中的Pose0中。如[[#^cf61bf]]意为弓箭放松状态。
内部逻辑如下。

![[Animations/LearnALSV4/手持道具Media/39.png]]

##### TwoWayBlend
注意到这里多次使用了Two Way Blend这个混合节点。要理解这个Two Way Blend节点可以看引擎中的这段代码。

![[Animations/LearnALSV4/手持道具Media/40.png]]

![[Animations/LearnALSV4/手持道具Media/41.png]]

![[Animations/LearnALSV4/手持道具Media/59.png]]

我们TwoWayBlend节点的Setting如下

![[Animations/LearnALSV4/手持道具Media/42.png]]

那么这里连续使用ALS_Props_Bow_Poses的Sequence Evaluator加上TwoWayBlend并配合Weight_Gait这个曲线的含义是什么呢？查看ALS_Props_Bow_Poses这个Anim Sequence可以知道其实在这个动画片段中是没有做Weight_Gait这个曲线的。查看这个Weight_Gait曲线使用到的动画序列也有挺多的。基本上是在走跑冲的动画中。

![[Animations/LearnALSV4/手持道具Media/43.png]]

跑中的Weight_Gait的曲线值是一条直线 2 ，走是1，冲是3。
如果当前角色没有播放走跑冲动画，曲线的是0。可以在引擎源码中看到。 ^ee475d

```cpp
float UAnimInstance::GetCurveValue(FName CurveName) const  
{  
    float Value = 0.f;  
    GetCurveValue(CurveName, Value);  
    return Value;  
}
```

这个状态是Bow的Relaxed状态。加上走跑冲专用曲线。就能知道这是Bow Relaxed和走跑冲上下身混合的时候用的。这里TwoWayBlend节点就起到一个分支选择的作用。

##### BlendMulti
接着看后面的代码，把前面根据Weight_Gait曲线选择的ALS_Prop_Bow_Pose中对应时间节点的Pose和ALS_Prop_Bow_Pose 0.28s的Pose[[#^0f792c]]进行Blend Multi。

![[Animations/LearnALSV4/手持道具Media/44.png]] ^4c2121

![[Animations/LearnALSV4/手持道具Media/45.png]] ^0f792c

Blend Multi 根据 BasePoseN 和 Base Pose CLF 两个值进行混合。根据如下基本同名的曲线提取值，这个UpdateLayerValues函数，是在EventBlueprintUpdateAnimation中实时更新的。 ^0ef9f7

![[Animations/LearnALSV4/手持道具Media/46.png]]

使用到BasePose_N曲线的动画如下

![[Animations/LearnALSV4/手持道具Media/47.png]]

可以看到BasePose_N为0的时候为蹲下状态，BasePose_N为1的时候是站立状态。BasePose_CLF曲线则是刚好与之相反。[[ALSV4站立和蹲下状态]]，这里我可以实时打印出曲线的值来进行调试，站立后BasePose_N一直都是1，BasePose_CLF一直是0，即使进入走跑冲跳。 ^65bd8a

![[Animations/LearnALSV4/手持道具Media/48.png]]

此代码段[[#^4c2121| BlendMulti]]，的作用就是根据两个曲线的值进行多pose的多权重混合。即Pose0 \* weight0 + Pose1 \* weight1输出的pose。

##### SecondaryMotion叠加动画
判断完是站立的各个状态还是蹲下的状态后，接下来是和叠加动画进行叠加[[叠加动画]]

![[Animations/LearnALSV4/手持道具Media/49.png]]

这个动画的名字是ALS_N_SecondaryMotion，意为二次运动的叠加。

![[52.avif]]

进入叠加动画内部查看相应的设置。比较重要的就属这部分。叠加动画类型是LocalSpace。BasePoseType是Selected Animation Frame当前为ALS_N_Pose的第0帧。

![[Animations/LearnALSV4/手持道具Media/50.png]]

不过这个ALS_N_Pose也就是一个两帧动画。保持这个姿势没变过。

![[Animations/LearnALSV4/手持道具Media/51.png]]

在使用这个动画序列的时候把此序列设置成同步组。[[同步组设置]]

![[Animations/LearnALSV4/手持道具Media/53.png]]
<br>
同步角色是CanBeLeader，只要这个动画有更高的混合权重就能成为领导者。

![[Animations/LearnALSV4/手持道具Media/54.png]]

而这个叠加动画ALS_N_SecondaryMotion里并没有设置同步组标记[[52.avif]]，从表现上看只是个站立呼吸的叠加动画。
所以这个BowRelaxed(States)的输出姿势本质上就是一个走/跑/冲上半身的定格动画Evaluator+一个呼吸的loop叠加动画。
我将这个叠加动画的同步组设置成不同步，实测没有什么变化，可能是不怎么明显。没有同步组标记的同步组动画只是让动画时长能自动对齐领导者动画。

上面提到的动画Evaluator+Addtive的同步组动画，输出的会是一段新的Pose动画，就像动画序列（sequence player）那样。
在上面的案例中就是呼吸的动画叠加到走/跑/冲/停/蹲的单帧Pose上，形成走/跑/冲/停/蹲的呼吸动画序列。 ^b72a56

最终合成的动画序列使用到的地方如下就是这个紫色的OverlayLayer的AnimationLayer[[AnimationLayer动画层]]，这个动画蓝图ALS_AnimBP中没有使用动画层接口[[动画层接口]]，而是直接创建AnimationLayers的。 ^1aed19

![[Animations/LearnALSV4/手持道具Media/55.png]]

点开LayerBlending的动画层我们可以看到OverlayLayer Input在这里只是作为一个姿势缓存来使用。然后动态生成叠加动画，再用于后面的分层混合逻辑。[[LayeredBlendPerBone分层混合]]

![[Animations/LearnALSV4/手持道具Media/56.png]]

SyncGroup的叠加动画如这个呼吸的叠加动画的同步组（SecondaryMotion）。经过简单搜索都在OverlayLayer中，所以这个无同步标记的同步组叠加动画的含义就是让在这个OverlayLayer中的OverlayStates这个状态机中的各个状态及其子状态的循环动画的时长一致，因为用的同步组叠加动画都是同一个如ALS_N_SecondaryMotion，所以不需要同步标记。
![[Animations/LearnALSV4/手持道具Media/57.png]]

#### 3.Bow Ready (State)
基本上和前面的BowRelaxed状态的逻辑都差不多，这里Idle用单独一个Evaluator，走跑用单独一个Evaluator，蹲用一个Evaluator，然后还是用呼吸动画进行叠加。

![[Animations/LearnALSV4/手持道具Media/58.png]]

#### 4.Bow Aiming (State)
同样和前面类似，这里把Idle和Walk的Evaluator的Pose按Weight_Gait[[#^ee475d]]归类为StandingPose，然后输出的pose和ALS_Props_Bow_Aim_Sweep的叠加Pose进行MeshSpace的Additive，因为叠加动画Evaluator里的叠加类型是基于MeshSpace的。
![[Animations/LearnALSV4/手持道具Media/60.png]] ^42f43b

这里有一个变量AimSweepTime是在Update函数中动态根据瞄准方向的y值（-90.90）映射成（1,0）
![[Animations/LearnALSV4/手持道具Media/61.png]]

这里的AimingRotation是获取PlayerController的旋转
![[Animations/LearnALSV4/手持道具Media/62.png]]

回到[[#^42f43b]]把Evaluator叠加Pose（Pitch方向的俯仰Pose）和Idle/走的拉弓Evaluator的Pose叠加后生成一个新的单帧Pose，然后同样和后面的呼吸叠加动画进行叠加。
下面的分支是下蹲Pose的混合分支。