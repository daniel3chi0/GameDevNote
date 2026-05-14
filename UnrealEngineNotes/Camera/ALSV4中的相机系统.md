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
- Step2：把PlayerCamera的Rotation旋转插值到控制器的Rotation，观察了下使用RotationLagSpeed的曲线值在各个状态中都设置为恒定20，后面的DebugViewRotation是在ALS_PlayerCameraManager中设定的一个固定旋转值供我们测试的时候，强制把相机转到角色的正面，Override_Debug曲线值是作为一个开关来打开这个功能。在ALS_Player_Controller中操作开关，通过BPI_GetDebugInfo在ALS_PlayerCameraBehavior的EventBlueprintUpdateAnimation的UpdateCharacterInfo中赋值，缓存成DebugView变量，最终在其动画图表的DebugViewOverride注释块中根据缓存的DebugView变量修改OverrideDebug曲线值为1或0。
  **GetCameraRotation**：这是PCM中的函数调用的堆栈如下，这里不做过多的探究，目前表现上来看在这一步Camera的Location代表Controller控制的Pawn的眼睛处的Location和APawn::GetViewRotation()。
  ```cpp
  ULocalPlayer::GetViewPoint(FMinimalViewInfo& OutViewInfo)
	  if (PlayerController->PlayerCameraManager != NULL)  
		{  
		    OutViewInfo = PlayerController->PlayerCameraManager->GetCameraCacheView();  
		    OutViewInfo.FOV = PlayerController->PlayerCameraManager->GetFOVAngle();  
		    PlayerController->GetPlayerViewPoint(/*out*/ OutViewInfo.Location, /*out*/ OutViewInfo.Rotation);
			    if (IsInState(NAME_Spectating) && HasAuthority() && !IsLocalController())  
				{  
				    // Server uses the synced location from clients. Important for view relevancy checks.  
				    out_Location = LastSpectatorSyncLocation;  
				    out_Rotation = LastSpectatorSyncRotation;  
				}  
				else if (PlayerCameraManager != NULL &&   
				PlayerCameraManager->GetCameraCacheTime() > 0.f) // Whether camera was updated at least once)  
				{  
				    PlayerCameraManager->GetCameraViewPoint(out_Location, out_Rotation);  
				}
			    else  //第一次走这里
				{  
					//GetViewTarget内部是返回PCM的ViewTarget.Target(Target可以是PC，Pawn，PS)，如果是空的会默认用传入的PC。
				    AActor* TheViewTarget = GetViewTarget();  
				  
				    if( TheViewTarget != NULL )  
				    {       
					    out_Location = TheViewTarget->GetActorLocation();  
					    out_Rotation = TheViewTarget->GetActorRotation();  
				    }    
				    else  
				    {  
					    //找到控制的Pawn的ViewPoint
				        Super::GetPlayerViewPoint(out_Location,out_Rotation);  
				    }  
				    out_Location.DiagnosticCheckNaN(*FString::Printf(TEXT("APlayerController::GetPlayerViewPoint: out_Location, ViewTarget=%s"), *GetNameSafe(TheViewTarget)));  
				    out_Rotation.DiagnosticCheckNaN(*FString::Printf(TEXT("APlayerController::GetPlayerViewPoint: out_Rotation, ViewTarget=%s"), *GetNameSafe(TheViewTarget)));  
				}
		}
  
  APlayerCameraManager::GetCameraRotation()
	  GetCameraCacheView().Rotation
		  CameraCachePrivate.POV
  ```
  
![[Camera/ALSV4中的相机系统Media/6.png]]

- Step3：通过自定义函数CalculateAxisIndependentLag（计算轴的独立滞后），输出一个Location结果。
  这里把相机的Rotation（欧拉角）只保留Yaw分量，然后把CurrentLocation和TargetLocation这两个世界坐标反旋转相机的Rotation（即转成相机坐标系中表示这两个向量，如果不这样Lag的时候`SmoothedPivotTarget（平滑的支点目标）.Location`会和我们想要的结果有细微偏差，因为是和LagSpeeds这个相对于相机空间的向量的xyz轴向的分量进行FInterp）**这点还是很细的！！**
  ![[Camera/ALSV4中的相机系统Media/10.png]]
  参数：
  1. `CurrentLocation`取自ALS_PlayerCameraManager中的`SmoothedPivotTarget（平滑的支点目标）.Location`。
  2. `TargetLocation`取自ALS_PlayerCameraManager中的`PivotTarget（支点目标）.Location`。
  3. `CameraRotation`取自ALS_PlayerCameraManager中的`TargetCameraRotation`。
  4. `LagSpeeds`取自根据曲线PivotLagSpeed_X，PivotLagSpeed_Y，PivotLagSpeed_Z的值组成的vector值。
  注释中提到`SmoothedPivotTarget（平滑的支点目标）`是OrangeSphere。`PivotTarget（支点目标）`是GreenSphere。可以在debug模式下查看这些Sphere：OrangeSphere其实是和我们的runtime中的相机位置正相关，因为其是滞后点和相机同步滞后。复原时OrangeSphere和GreenSphere重叠，相机也和OrangeSphere一起复位。
  ![[Camera/ALSV4中的相机系统Media/9.png]]
- Step4：引入了一个新概念叫Pivot Location（支点位置）注释中写到这个位置代表debug中的BlueSphere，在Step3中计算得出`SmoothedPivotTarget.Rotation`其实是`PivotTarget的Rotation`也就是ALS_AnimMan_CharacterBP的`ActorRotation`。接着用`SmoothedPivotTarget.Rotation`的xyz基向量乘PivotOffset_X，PivotOffset_Y，PivotOffset_Z的曲线值附加到`SmoothedPivotTarget.Location`上最后的结果赋值到PivotLocation上。
- Step5：根据TargetCameraRotation的xyz基向量乘CameraOffset_X，CameraOffset_Y，CameraOffset_Z并附加到PivotLocation并设置为TargetCameraLocation（目标相机位置），后面lerp是看是否开启debug模式。
![[Camera/ALSV4中的相机系统Media/11.png]]

- Step6：根据ALS_AnimMan_CharacterBP的接口BPI_Get_3P_TraceParams，返回左右肩的TraceOrigin。接着做SphereTrace射线检测End位置为TargetCameraLocation，根据被阻挡的位置动态调整TargetCameraLocation的位置。这步的意思就是模拟SpringArmComponent的效果。（注释里写的很明显了）
- Step7：这步就是绘制`debug sphere`和`debug line`。
![[Camera/ALSV4中的相机系统Media/12.png]]

- Step8：根据Weight_FirstPerson曲线的值Lerp第一人称或者第三人称的相机位置，旋转和FOV。
![[Camera/ALSV4中的相机系统Media/13.png]]

# 冲刺时的相机震动
直接在冲刺的AS资源中添加CameraShake_Notify。

![[Camera/ALSV4中的相机系统Media/14.png]]

这个CameraShake通知中比较核心的方法就是用PlayerController上的ClientStartCameraShake。
关于Shake的配置在ShakeClass中。
![[Camera/ALSV4中的相机系统Media/15.png]]