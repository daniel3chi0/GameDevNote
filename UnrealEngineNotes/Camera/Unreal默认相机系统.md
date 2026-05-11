引用文章：[虚幻引擎相机系统原理机制源码剖析-你丫才建模-SurfCG](https://www.surfcg.com/article/835644)
[【UEGamePlay】- 3C篇（三） : Camera - 丨桐 - 博客园](https://www.cnblogs.com/Tong0115/p/19201040)
[https://zhuanlan.zhihu.com/p/34897458](https://zhuanlan.zhihu.com/p/34897458)

相机的本质是封装和维护一套具有视口目标的位置和旋转等信息的数据结构，本体无法直接进入渲染流程，在Runtime下主要由PlayerCameraManager提供当前激活的相机，利用其中的数据提交至渲染器(RHI)。编辑器中看到的相机实体则代表了玩家的视角。

# Camera
相机（摄像机，Camera） 代表了玩家的视角，比如玩家如何查看世界。因此， 相机只和玩家控制的人物有关。PlayerController 会指定一个PlayerCameraManager类以此计算玩家从哪个位置和角度 观察世界。在服务器上可以通过GetAutoActivateCameraForPlayer()来获取与PlayerIndex对应的CameraActor（只支持0~7号PlayerIndex）。

# CameraActor
本质上就是一个默认带有CameraComponent的Actor，有关相机的所有属性和行为均在 CameraComponent 中设置。CameraActor 类主要用作 CameraComponent 的包装器，以使相机可以被直接放置在该关卡内，而非另一个类中。

# CameraComponent
CameraComponent 代表相机视角和设置，比如投射类型（Projection Type）、视野（Field Of View） 和 后期处理覆盖（Post-Process Overrides）。如果 ViewTarget 是一个 CameraActor 或者包含 CameraComponent 并且 bFindCameraComponentWhenViewTarget 设为 true 的Actor，bTakeCameraControlWhenPossessed 是一个可以为任何 Pawn 设置的相关属性，Pawn 会在 PlayerController 占有时自动变为 ViewTarget。

# Actors 和 PlayerControllers
PlayerControllers 和 Actors 都含有 CalcCamera 函数。如果 bFindCameraComponentWhenViewTarget 为 true，而且 CameraComponent 存在，Actor 的 CalcCamera 函数返回 Actor 中的首个 CameraComponent 的相机视图，否则，它获取 Actor 的位置和旋转方向。在 PlayerController 类中，CalcCamera 函数的行为方式与第二种情况类似，如果存在占有 Pawn ，则返回其位置以及 PlayerController 的控制旋转。

# PlayerCameraManager
PlayerCameraManager 用于为一个特定的玩家管理相机。它定义最终查看属性，供渲染器这类的其它系统使用。PlayerCameraManager 的功能类似于一个用来查看世界的 "虚拟眼球"。 它可以直接计算最终相机属性，也能够在其它物体或者影响相机的Actor之间混合（从一个 CameraActor 混合到另外一个）。 PlayerCameraManager 的主要外部职责是要可靠地对各种 Get() 函数做出响应，比如 GetCameraViewPoint 。 默认情况下，PlayerCameraManager 保留一个查看目标，为相机所关联的主要 Actor 。它可以向最终查看状态应用各种后期效果，比如相机动画、后期处理效果或者特殊效果（比如相机镜头上的尘土）。
PlayerCameraManager 中的 UpdateViewTarget 函数查询ViewTarget，并返回该ViewTarget 的视角。

## ViewTarget
ViewTarget 结构体在PlayerCameraManager中定义，负责向 PlayerCameraManager提供理想的视角(POV)。ViewTarget包含有关 Actor目标、以及 PlayerState的信息（用于在观看时跟随同一个玩家完成Pawn 过渡和其它变更）。
相机信息被通过视角（POV）属性以 "FMinimalViewInfo" 结构体的形式传给 PlayerCameraManager。该结构包含来自 CameraComponent 的基本摄像机信息，包括 位置（Location）、旋转（Rotation）、 投射模式（Projection Mode）（透视或正交）、视场（FOV）、投影宽度（Orthographic Width）、宽高比（Aspect Ratio） 和 后期处理效果（Post Process Effects）。让 PlayerCameraManager访问这些值使PlayerCameraManager能在摄像机管理期间混合两种摄像机模式。

### FMinimalViewInfo
结构包含渲染所需的重要信息:
- Location 位置
- Rotation 旋转
- FOV 视场
- AspectRatio 宽高比
- OrthoWidth 投影宽度(正交视图下)
- ProjectionMode 投影模式(正交/透视)
- PostProcesSettings 后期处理设置

# 调试
- 可在命令行输入 ShowDebug CAMERA 来展示相机相关属性状态。可在自定义的 UCameraModifier 的 DisplayDebug 方法中添加自己的信息。
  ![[Camera/Unreal默认相机系统/1.png]]

- 可在 Outline 里取消 CameraComponent 的 HiddenInGame 以达到在 PIE 下显示相机位置的目的。

# 类图示

![[Camera/Unreal默认相机系统/2.png]]

![[Camera/Unreal默认相机系统/9.png]]

相机系统的关键类和对应的函数如上图所示。
其中 APlayerCameraManager 是枢纽，其中缓存着包含相机位置和旋转等信息的 FCameraCacheEntry CameraCachePrivate ，相机管理器内部通过 SetCameraCachePOV 对其进行修改调整，外部通过 GetCameraViewPoint 对其进行获取。
相机的位置和旋转等信息主要受到 UCameraComponent 及 APlayerController 等的影响。实际上根据是否启用 bUsePawnControlRotation ，相机的主要影响者在相机组件及其父节点（如 USpringArmComponent 以及其最上层的玩家 Actor 等）与玩家控制器之间进行切换。APlayerCameraManager 内部也会通过 CameraStyle 和 UCameraModifier 对相机相关属性进行调整。

# 更新流程

![[Camera/Unreal默认相机系统/3.png]]

# 时序图
- 通过 APlayerCameraManager 调整 APlayerController 的旋转等
  ![[Camera/Unreal默认相机系统/4.png]]

- 基于 APlayerController 等的旋转来控制 USpringArmComponent 与 UCameraComponent 的旋转等
  ![[Camera/Unreal默认相机系统/5.png]]

- 基于 UCameraComponent 或 APlayerController 设置 APlayerCameraManager 中的 CameraCachePrivate.POV
  ![[Camera/Unreal默认相机系统/6.png]]

- 拿到 APlayerCameraManager 中的 CameraCachePrivate.POV 初始化 FViewInfo 等
  ![[Camera/Unreal默认相机系统/7.png]]

- 利用 FViewInfo 进行绘制等操作
  ![[Camera/Unreal默认相机系统/8.png]]

# GamePlay流程

GamePlay流程中，Camera负责用来根据GamePlay中的状态收集并且更新关键数据

由APlayerCameraManager更新FOV流程(更新渲染所需的关键数据FMinimalViewInfo)

```cpp
UWorld::Tick()
	APlayerController::UpdateCameraManager()
		APlayerCameraManager::UpdateCamera()
			APlayerCameraManager::DoUpdateCamera()
				APlayerCameraManager::UpdateViewTarget()
					APlayerCameraManager::UpdateViewTargetInternal() //更新重要数据FTViewTarget ViewTarget;
				FMinimalViewInfo NewPOV = ViewTarget.POV //更新重要数据FMinimalViewInfo;
				APlayerCameraManager::FillCameraCache(NewPOV)
					SetCameraCachePOV(NewInfo) //更新重要数据CameraCachePrivate.POV
					SetCameraCacheTime(CurrentGameTime);
```

![[Camera/Unreal默认相机系统/10.png]]

# Render流程

Runtime下的渲染流程(FMinimalViewInfo被渲染流程调用)：

```cpp
UGameEngine::Tick()
	UGameEngine::RedrawViewports() //渲染一切
		FViewport::Draw() //更新游戏逻辑后更新渲染
			UGameViewportClient::Draw(FViewport* InViewport ...)
				ULocalPlayer::CalcSceneView(FViewport* Viewport ...)
					ULocalPlayer::GetViewPoint(FMinimalViewInfo& OutViewInfo)
						//设置FMinimalViewInfo
```

```cpp
ULocalPlayer::GetViewPoint(FMinimalViewInfo& OutViewInfo)
	APlayerController::GetPlayerViewPoint(out_Location,out_Rotation)
		APlayerCameraManager::GetCameraViewPoint(&OutCamLoc,&OutCamRot)
			const FMinimalViewInfo& CurrentPOV = GetCameraCacheView(); //return CameraCachePrivate.POV;
			OutCamLoc = CurrentPOV.Location;
			OutCamRot = CurrentPOV.Rotation;
```

### 总结

UE的摄像机系统框架的主要流程如下：

- 每个APlayerController对象都会在PostInitializeComponents方法里创建APlayerCameraManager对象，并**初始化ViewTarget**；
- 当Pawn/Character上挂载相机臂组件时，开启bUsePawnControlRotation,此时的相机臂组件的期望旋转会与Pawn::GetViewRotation(通常为控制器旋转)同步.由于相机臂中的GetSocketTransform,会使其相机旋转同步。
- UWorld::Tick 调用APlayerController的UpdateCameraManager，调用Controller绑定的APlayerCameraManager的UpdateViewTargetInternal方法里不断**更新ViewTarget**；
- 更新完游戏逻辑之后，在调用UGameViewportClient的Draw方法渲染画面时，会通过ULocalPlayer的CalcSceneView方法，最终从APlayerCameraManager中**获取ViewTarget中的POV数据用于渲染**；
- 外界主要通过 APlayerCameraManager::GetCameraCacheView 来获取 CameraCachePrivate.POV
- FMinimalViewInfo 来执行视口判定等逻辑，如窗口绘制、关卡的流式加载、世界分区、音频、界面、Niagara 等。
- DoUpdateCamera中的更新流程：先更新FTViewTarget然后转入更新FMinimalViewInfo ; 最后利用FMinimalViewInfo更新FCameraCacheEntry CameraCachePrivate;CameraCachePrivate则暴露给外界使用
- APlayerCameraManager::UpdateViewTarget 计算给定/有效的ViewTarget去更新POV
- APlayerCameraManager::UpdateViewTargetInternal中会判断BlueprintUpdateCamera是否被蓝图重写，如果没有则调用ViewTarget.Target->CalcCamera();