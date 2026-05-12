ALSV4中的相机管理其实相对简单，在下图中的Crouch，Roll，Aim，SwitchShoulders，CamPerspective，ToRagdoll
状态下相机会有一定的变化。其实就是一些偏移的变化不是很明显。
![[Camera/ALSV4中的相机系统Media/1.png]]

# Camera组件在哪？
ALS中默认角色ALS_AnimMan_CharacterBP中是没有挂载CameraComponent的。
通常在第三人称模板中CameraComponent会挂在角色蓝图身上，可能父节点还会挂一个SpringArmComponent充当相机臂，ALS中采用创建自定义PlayerCameraManager，这个相机管理器上同样没有CameraComponent，了解Unreal相机管理系统后[[Unreal默认相机系统]]，知道我们不一定要用到CameraComponent。但是这个PlayerCameraManager上有一个叫做CameraBehavior的骨骼网格体，这个骨骼网格体，主要是用来挂载ALS_PlayerCameraBehavior_C这个ABP的，这个ABP中主要有一个控制相机状态的状态机。然后是缓存成一个MainCameraStates的Pose。在这之后通过修改Pose的曲线值缓存成各种Pose。
![[Camera/ALSV4中的相机系统Media/2.png]]

## MainCameraStates
这个状态机中的状态就这几个：VelocityDirection，Aiming，LookingDirection。
![[Camera/ALSV4中的相机系统Media/3.png]]

这些状态中的逻辑基本上是这样的，通过ModifyCurve节点不指定SourcePose，只修改曲线参数。这种情况下输出的pose始终是个Reference Pose+设置的曲线值（绑定姿势，具体是T-Pose还是A-Pose由DCC工作的初始绑定姿势决定的）。
![[Camera/ALSV4中的相机系统Media/4.png]]

## 这些控制相机的曲线在什么时候使用值？
搜下引用可以知道在ALS_PlayerCameraManager的BlueprintUpdateCamera中的CustomCameraBehavior中会使用到之前修改的曲线的值。因为BlueprintUpdateCamera在蓝图中有实现所以不会跑C++那边的逻辑。
![[Unreal默认相机系统#^7eaf4f]]

![[Camera/ALSV4中的相机系统Media/5.png]]

虽然PlayerCameraManager中有APawn* GetViewTargetPawn()但是没有暴露到蓝图中
所以要拿到蓝图中的PlayerCameraManager关联的Pawn在ALSV4中是这样做的：在Controller占有Pawn的时候
调用自定义PlayerCameraManager的OnPossess方法。
![[Camera/ALSV4中的相机系统Media/7.png]]

该OnPossess函数中用ControlledPawn变量缓存下Controller中传过来的PossessedPawn。
![[Camera/ALSV4中的相机系统Media/8.png]]

CustomCameraBehavior中会使用之前动画图表中的曲线值。
- Step1：中用接口函数返回获取 **PivotTarget**（角色Mesh上的head和root的socket的中点的Transform），**FPTarget**（角色Mesh上的FP_Camera的Socket的WorldLocation，位置在Mesh的眼睛的位置中间），**TPFOV和FPFOV**（角色蓝图上甚设置的第三人称和第一人称的FOV浮点值）
- Step2：把PlayerCamera
![[Camera/ALSV4中的相机系统Media/6.png]]