# 类大纲
Lyra中的相机系统代码相关的目录都在Camera目录下：
![[Camera/Lyra中的相机系统Media/1.png]]

# LyraCameraAssistInterface（相机协助接口）
```cpp
UINTERFACE(BlueprintType)  
class ULyraCameraAssistInterface : public UInterface  
{  
    GENERATED_BODY()  
};  
  
class ILyraCameraAssistInterface  
{  
    GENERATED_BODY()  
  
public:  
    /**  
     * Get the list of actors that we're allowing the camera to penetrate. Useful in 3rd person cameras     * when you need the following camera to ignore things like the a collection of view targets, the pawn,     * a vehicle..etc.     */ 
     virtual void GetIgnoredActorsForCameraPentration(TArray<const AActor*>& OutActorsAllowPenetration) const { }  
  
    /**  
     * The target actor to prevent penetration on.  Normally, this is almost always the view target, which if     * unimplemented will remain true.  However, sometimes the view target, isn't the same as the root actor     * you need to keep in frame.  
     */    
     virtual TOptional<AActor*> GetCameraPreventPenetrationTarget() const  
     {  
        return TOptional<AActor*>();  
     }
     
    /** Called if the camera penetrates the focal target.  Useful if you want to hide the target actor when being overlapped. */  
     virtual void OnCameraPenetratingTarget() { }  
};
```
接口中ULyraCameraAssistInterface基本是个空实现
ILyraCameraAssistInterface中声明这几个接口：
- GetIgnoredActorsForCameraPentration（获取忽略Actors为相机穿透），项目中是空实现，调用的地方被注释了在`ULyraCameraMode_ThirdPerson::PreventCameraPenetration`中，有如下代码段：（我们可以在这个阶段为相机的穿透注入忽略Actors）
  ```cpp
  //TODO ILyraCameraTarget.GetIgnoredActorsForCameraPentration();  
  //if (IgnoreActorForCameraPenetration)  
  //{  
  //  SphereParams.AddIgnoredActor(IgnoreActorForCameraPenetration);  
  //}
  ```
- GetCameraPreventPenetrationTarget()（获取相机阻止穿透目标），返回一个TOptional<AActor*>()空对象，本质上就是返回一个Pawn上设定的Actor类型的阻挡相机目标，这个阻挡目标要实现LyraCameraAssistInterface接口需要有根组件，然后参与`ULyraCameraMode_ThirdPerson::PreventCameraPenetration`阻止相机穿透。同时调用这个Actor上的`OnCameraPenetratingTarget()`响应函数。
  在`ULyraCameraMode_ThirdPerson::UpdatePreventPenetration(float DeltaTime)`中有如下代码：
  ```cpp
	AActor* TargetActor = GetTargetActor();
	ILyraCameraAssistInterface* TargetActorAssist = Cast<ILyraCameraAssistInterface>(TargetActor);
	TOptional<AActor*> OptionalPPTarget = TargetActorAssist ? TargetActorAssist->GetCameraPreventPenetrationTarget() : TOptional<AActor*>();
	AActor* PPActor = OptionalPPTarget.IsSet() ? OptionalPPTarget.GetValue() : TargetActor;  
	ILyraCameraAssistInterface* PPActorAssist = OptionalPPTarget.IsSet() ? Cast<ILyraCameraAssistInterface>(PPActor) : nullptr;
  
	ILyraCameraAssistInterface* AssistArray[] = { TargetControllerAssist, TargetActorAssist, PPActorAssist };  
  
	PreventCameraPenetration(*PPActor, SafeLocation, View.Location, DeltaTime, AimLineToDesiredPosBlockedPct, bSingleRayPenetrationCheck);
	if (AimLineToDesiredPosBlockedPct < ReportPenetrationPercent)  
	{  
	    for (ILyraCameraAssistInterface* Assist : AssistArray)  
	    {       
		    if (Assist)  
	        {          
		        // camera is too close, tell the assists  
		        Assist->OnCameraPenetratingTarget();  
	        }    
	    }
	}
  ``` ^b5205d
- OnCameraPenetratingTarget()（相机正在穿透目标行为函数），在接口中无实现在`ALyraPlayerController`中重写如下：
  ```cpp
  void ALyraPlayerController::OnCameraPenetratingTarget()  
{  
    bHideViewTargetPawnNextFrame = true;  
}
  ```

# ULyraCameraComponent
## ULyraCameraComponent中的成员变量：
```cpp
public:
	// Delegate used to query for the best camera mode.  
	FLyraCameraModeDelegate DetermineCameraModeDelegate;
	
protected:
	// Stack used to blend the camera modes.  
	UPROPERTY()  
	TObjectPtr<ULyraCameraModeStack> CameraModeStack;  
	  
	// Offset applied to the field of view.  The offset is only for one frame, it gets cleared once it is applied.  
	float FieldOfViewOffset;
```
- DetermineCameraModeDelegate在ULyraHeroComponent::HandleChangeInitState中找到相机组件的时候绑定。在ULyraCameraComponent::UpdateCameraModes()中如果CameraModeStack是激活的调用DetermineCameraModeDelegate返回一个CameraMode然后Push到CameraModeStack中。
- CameraModeStack变量名字可以很明显的看出是个栈结构。内部维护两个`ULyraCameraMode`类型数组变量`CameraModeInstances`，`CameraModeStack`。以及内部实现的栈方法。
  **CameraModeInstances**使用的地方如下get的时候没有实例就Add进CameraModeInstances。`CameraModeInstances`充当对象池的角色。
	```cpp
	APlayerController::SpawnPlayerCameraManager()
		void APlayerCameraManager::InitializeFor(APlayerController* PC)
			void APlayerCameraManager::UpdateCamera(float DeltaTime)
							&&
	void UWorld::Tick( ELevelTick TickType, float DeltaSeconds )
		void APlayerController::UpdateCameraManager(float DeltaSeconds)
			void APlayerCameraManager::UpdateCamera(float DeltaTime)
				void APlayerCameraManager::DoUpdateCamera(float DeltaTime)
					void APlayerCameraManager::UpdateViewTarget(FTViewTarget& OutVT, float DeltaTime)
						void APlayerCameraManager::UpdateViewTargetInternal(FTViewTarget& OutVT, float DeltaTime)
							void AActor::CalcCamera(float DeltaTime, FMinimalViewInfo& OutResult)
								void ULyraCameraComponent::GetCameraView(float DeltaTime, FMinimalViewInfo& DesiredView)
									void ULyraCameraComponent::UpdateCameraModes()
										 ULyraCameraModeStack::PushCameraMode(TSubclassOf<ULyraCameraMode> CameraModeClass)
											 ULyraCameraMode* CameraMode = GetCameraModeInstance(CameraModeClass);
			 
	ULyraCameraMode* ULyraCameraModeStack::GetCameraModeInstance(TSubclassOf<ULyraCameraMode> CameraModeClass)
	{
		check(CameraModeClass);
	
		// First see if we already created one.
		for (ULyraCameraMode* CameraMode : CameraModeInstances)
		{
			if ((CameraMode != nullptr) && (CameraMode->GetClass() == CameraModeClass))
			{
				return CameraMode;
			}
		}
	
		// Not found, so we need to create it.
		ULyraCameraMode* NewCameraMode = NewObject<ULyraCameraMode>(GetOuter(), CameraModeClass, NAME_None, RF_NoFlags);
		check(NewCameraMode);
	
		CameraModeInstances.Add(NewCameraMode);
	
		return NewCameraMode;
	}
	```

  **CameraModeStack**是当前激活的相机模式的栈。使用到的地方就比较多了 ^66829e
	- `ULyraCameraModeStack::ActivateStack()`和`ULyraCameraModeStack::DeactivateStack()`中对Stack中的CameraMode遍历调用`OnActivation()`和`OnDeactivation()`。
	- `void ULyraCameraModeStack::PushCameraMode(TSubclassOf<ULyraCameraMode> CameraModeClass)`中涉及`CameraModeStack`的RemoveAt，Insert和Last()操作。
	- `ULyraCameraModeStack::UpdateStack(float DeltaTime)`移除BlendWeight>=1时元素上方的所有元素。
	- `void ULyraCameraModeStack::BlendStack(FLyraCameraModeView& OutCameraModeView) const` 中把`FLyraCameraModeView& OutCameraModeView`从`CameraModeStack`栈底（Array的底端）开始Blend。
	- `ULyraCameraModeStack::GetBlendInfo(float& OutWeightOfTopLayer, FGameplayTag& OutTagOfTopLayer) const`返回栈底（CameraModeStack.Last()）元素权重和tag。
- FieldOfViewOffset译为视野偏移，在`void ULyraCameraComponent::GetCameraView(float DeltaTime, FMinimalViewInfo& DesiredView)`中给经过CameraModeStack->EvaluateStack的FLyraCameraModeView CameraModeView.FieldOfView加上这个偏移。

## 比较重要的方法
- `virtual void UpdateCameraModes()`: 属于是Tick级别的方法前面以及提及了，主要是调用CameraModeStack->PushCameraMode(CameraMode);
- `virtual void GetCameraView(float DeltaTime, FMinimalViewInfo& DesiredView) override`: 也在上面函数的调用栈中，用来返回一个FMinimalViewInfo& DesiredView会从CameraModeStack->EvaluateStack中计算所需要的数据。
```cpp
  void AActor::CalcCamera(float DeltaTime, FMinimalViewInfo& OutResult)
	void ULyraCameraComponent::GetCameraView(float DeltaTime, FMinimalViewInfo& DesiredView)
		void ULyraCameraComponent::UpdateCameraModes()
```

- `virtual void OnRegister() override`: 重写ActorComponent中的组件注册事件，一开始CameraModeStack为空会New一个对象。

# ULyraCameraMode
这个文件中定义了几个类或结构体
- **ULyraCameraMode**：UObject的子类在CameraModeStack中作为栈元素（即一组位置旋转和混合相关的数据）
  **内部有几个比较重要的变量：**
  `FGameplayTag CameraTypeTag`：CameraMode对应的Tag。
  `FLyraCameraModeView View`：在下面
  `float BlendTime，float BlendAlpha，float BlendWeight`：
	  1. float BlendTime（在CameraMode构造函数中设置为0.5f），float BlendAlpha 在`void ULyraCameraMode::UpdateBlending(float DeltaTime)`中根据DeltaTime更新BlendWeight，调用栈如下：
	   ```cpp
		  void ULyraCameraComponent::GetCameraView(float DeltaTime, FMinimalViewInfo& DesiredView)
			  void ULyraCameraComponent::UpdateCameraModes()
			  //...
			  bool ULyraCameraModeStack::EvaluateStack(float DeltaTime, FLyraCameraModeView& OutCameraModeView)
				  void ULyraCameraModeStack::UpdateStack(float DeltaTime)
					  void ULyraCameraMode::UpdateCameraMode(float DeltaTime)
						  void ULyraCameraMode::UpdateBlending(float DeltaTime)
	   ```
	  2. `void ULyraCameraMode::SetBlendWeight(float Weight)`中根据传入的Weight设置BlendWeight并更新BlendAlpha，调用栈如下：
	    ```cpp
	    void ULyraCameraComponent::UpdateCameraModes()
		    void ULyraCameraModeStack::PushCameraMode(TSubclassOf<ULyraCameraMode> CameraModeClass)
			    void ULyraCameraMode::SetBlendWeight(float Weight)
	    ```
	    SetBlendWeight一定在UpdateBlending之前调用，计算出的BlendAlpha再用来计算UpdateBlending中的BlendWeight的。然后用这个UpdateBlending中的BlendWeight再计算下次SetBlendWeight的BlendAlpha。
		GetBlendTime()：
		1. 会在ULyraCameraModeStack::PushCameraMode中控制下次传入CameraMode中的BlendWeight。
		GetBlendWeight()：
		2. 会在ULyraCameraModeStack::PushCameraMode中提供下次传入CameraMode中的BlendWeight的数据。
		3. 在ULyraCameraModeStack::UpdateStack中控制要移除的栈元素的Index。
		4. 会在ULyraCameraModeStack::BlendStack中参与混合CameraModeView。
		5. 在ULyraRangedWeaponInstance::UpdateMultipliers中根据CameraComponent->GetBlendInfo拿到栈底元素的（外界称其为TopCameraWeight）BlendWeight。将其赋值给AimingAlpha，随后参与一些列计算赋值给ULyraRangedWeaponInstance中的CurrentSpreadAngleMultiplier：（这里有些数学计算后续再分析）[[Lyra中的AimingAlpha和CurrentSpreadAngleMultiplier]]
			1. 最终在LyraGameplayAbility_RangedWeapon中参与计算单个子弹的打击终点。
			2. ULyraReticleWidgetBase::ComputeSpreadAngle()中参与计算并返回ActualSpreadAngle。然后在ULyraReticleWidgetBase::ComputeMaxScreenspaceSpreadRadius()中通过接口PC->ProjectWorldLocationToScreen投影到屏幕空间.

  **几个比较重要的函数：**
  1. GetLyraCameraComponent()：返回Outer，虽然CameraMode是在ULyraCameraModeStack::GetCameraModeInstance中被NewObject出来的但是使用的Outer是ULyraCameraModeStack的Outer，也就是LyraCameraComponent。
  2. GetTargetActor()：返回Outer即LyraCameraComponent的Owner。
  3. GetPivotLocation()：返回相机位置默认是在Pawn的眼睛处的位置，这里处理了下蹲ACharacter::Crouch()后的相机位置调整为和站立时一致。
  4. GetPivotRotation()：返回TargetActor的rotation，如果是Pawn就是GetViewRotation，如果是Actor就是ActorRotation。
  5. UpdateView()：通过PivotLocation和PivotRotation更新View
  6. 其余的是在上面有提及的一些函数。
  
  
- **ULyraCameraModeStack**：UObject的子类前面介绍了被LyraCameraComponent所有[[Lyra中的相机系统#^66829e]]。前面没提到的几个方法介绍：
  `bool EvaluateStack(float DeltaTime, FLyraCameraModeView& OutCameraModeView)`：里面主要是依次调用UpdateStack(DeltaTime); 
  BlendStack(OutCameraModeView);
  `ActivateStack()和DeactivateStack()`：遍历调用CameraMode中的OnActivation和OnDeactivation函数。
- **FLyraCameraModeView**：一个简单的结构体内部有FVector Location，FRotator Rotation，FRotator ControlRotation，float FieldOfView这几个变量。

## ULyraCameraMode_ThridPerson
ULyraCameraMode的子类，从名字中可以看出来是专门用于第三人称的CameraMode。
## 重写的函数
- 重写了**UpdateView**函数：相比于基类中的函数主要改动就是对View.Location的赋值不再是简单的GetPivotLocation()的默认值。而是GetPivotLocation()+CurrentCrouchOffset+PivotRotation.RotateVector(TargetOffset)。
- 重写了**DrawDebug**函数：调用基类方法后，ENABLE_DRAW_DEBUG时（即非Shipping包时）多输出相机阻挡物的Name。

## 新添加的函数
- **UpdateForTarget**：在UpdateView中一开始调用，属于tick链路中的一个环节。主要是判断是否是Crouched状态，如果是的话调用SetTargetCrouchOffset应用下端的偏移量。
- **UpdatePreventPenetration**：[[Lyra中的相机系统#^b5205d]]
- **PreventCameraPenetration**：在UpdatePreventPenetration中调用返回一个AimLine到目标位置阻挡的百分比浮点值。如果AimLineToDesiredPosBlockedPct < ReportPenetrationPercent 时就触发Assist->OnCameraPenetratingTarget(); 在Lyra项目中只有在ALyraPlayerController中实现OnCameraPenetratingTarget让bHideViewTargetPawnNextFrame=true。会在ALyraPlayerController::UpdateHiddenComponents中让该PlayerController上的PlayerCameraManager的ViewTarget的所有组件在渲染线程中隐藏。这段逻辑用人话说就是阻挡物如果离相机镜头太近就会被隐藏，这个是镜头前的阻挡。这里计算量挺大的可以单独开一个坑了。[[Lyra的镜前阻挡物剔除Penetration]]

## 新加的变量
新加的变量还是相对比较多的，基本是和穿透Penetration和CrouchOffset相关的。

# LyraPenetrationAvoidanceFeeler（穿透避免探测器）
是个结构体，注释中提到：定义了一种用于避免相机穿透的探测射线的结构。
内部没有函数而是定义了一系列变量：
```cpp
/** FRotator describing deviance from main ray */  
UPROPERTY(EditAnywhere, Category=PenetrationAvoidanceFeeler)  
FRotator AdjustmentRot;  
  
/** how much this feeler affects the final position if it hits the world */  
UPROPERTY(EditAnywhere, Category=PenetrationAvoidanceFeeler)  
float WorldWeight;  
  
/** how much this feeler affects the final position if it hits a APawn (setting to 0 will not attempt to collide with pawns at all) */  
UPROPERTY(EditAnywhere, Category=PenetrationAvoidanceFeeler)  
float PawnWeight;  
  
/** extent to use for collision when tracing this feeler */  
UPROPERTY(EditAnywhere, Category=PenetrationAvoidanceFeeler)  
float Extent;  

/** minimum frame interval between traces with this feeler if nothing was hit last frame */  
UPROPERTY(EditAnywhere, Category=PenetrationAvoidanceFeeler)  
int32 TraceInterval;  
  
/** number of frames since this feeler was used */  
UPROPERTY(transient)  
int32 FramesUntilNextTrace;
```

探测器这个名词和这些变量注释都比较抽象不太好理解。看下这个结构体实际用到的地方。
Lyra项目中在第三人称的CameraMode中使用到。ULyraCameraMode_ThirdPerson中定义了探测器的数组
```cpp
/**
这些是用于确定摄像机放置位置的探测光束。
* 编号：0 ：这是我们常用的常规探测光束，用于避免碰撞。
* 编号：1+ ：如果“bDoPredictiveAvoidance=true”，则使用这些探测光束来扫描潜在的碰撞情况，假设玩家会朝那个方向旋转并与摄像机发生初步碰撞，从而在碰撞到遮挡物之前使摄像机收回。
*/
UPROPERTY(EditDefaultsOnly, Category = "Collision")  
TArray<FLyraPenetrationAvoidanceFeeler> PenetrationAvoidanceFeelers;
```

在构造函数中向数组中创建添加结构体对象
```cpp
ULyraCameraMode_ThirdPerson::ULyraCameraMode_ThirdPerson()
{
	TargetOffsetCurve = nullptr;

	PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(FRotator(+00.0f, +00.0f, 0.0f), 1.00f, 1.00f, 14.f, 0));
	PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(FRotator(+00.0f, +16.0f, 0.0f), 0.75f, 0.75f, 00.f, 3));
	PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(FRotator(+00.0f, -16.0f, 0.0f), 0.75f, 0.75f, 00.f, 3));
	PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(FRotator(+00.0f, +32.0f, 0.0f), 0.50f, 0.50f, 00.f, 5));
	PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(FRotator(+00.0f, -32.0f, 0.0f), 0.50f, 0.50f, 00.f, 5));
	PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(FRotator(+20.0f, +00.0f, 0.0f), 1.00f, 1.00f, 00.f, 4));
	PenetrationAvoidanceFeelers.Add(FLyraPenetrationAvoidanceFeeler(FRotator(-20.0f, +00.0f, 0.0f), 0.50f, 0.50f, 00.f, 4));
}

```

在ULyraCameraMode_ThirdPerson::PreventCameraPenetration中使用，核心逻辑如下
可以理解到这个探测器的概念就是定义了每一条从相机位置发出的射线的配置。打到的物体有LyraCameraMode_ThirdPerson_Statics::NAME_IgnoreCameraCollision 的tag会加入忽略，每个不可忽略的物体都会累加进最终的相机偏移。
```cpp
void ULyraCameraMode_ThirdPerson::PreventCameraPenetration(class AActor const& ViewTarget, FVector const& SafeLoc, FVector& CameraLoc, float const& DeltaTime, float& DistBlockedPct, bool bSingleRayOnly)  
{
#if ENABLE_DRAW_DEBUG
	DebugActorsHitDuringCameraPenetration.Reset();
#endif

	float HardBlockedPct = DistBlockedPct;
	float SoftBlockedPct = DistBlockedPct;

	FVector BaseRay = CameraLoc - SafeLoc;
	FRotationMatrix BaseRayMatrix(BaseRay.Rotation());
	FVector BaseRayLocalUp, BaseRayLocalFwd, BaseRayLocalRight;

	BaseRayMatrix.GetScaledAxes(BaseRayLocalFwd, BaseRayLocalRight, BaseRayLocalUp);

	float DistBlockedPctThisFrame = 1.f;

	int32 const NumRaysToShoot = bSingleRayOnly ? FMath::Min(1, PenetrationAvoidanceFeelers.Num()) : PenetrationAvoidanceFeelers.Num();
	FCollisionQueryParams SphereParams(SCENE_QUERY_STAT(CameraPen), false, nullptr/*PlayerCamera*/);

	SphereParams.AddIgnoredActor(&ViewTarget);

	//TODO ILyraCameraTarget.GetIgnoredActorsForCameraPentration();
	//if (IgnoreActorForCameraPenetration)
	//{
	//	SphereParams.AddIgnoredActor(IgnoreActorForCameraPenetration);
	//}

	FCollisionShape SphereShape = FCollisionShape::MakeSphere(0.f);
	UWorld* World = GetWorld();

	for (int32 RayIdx = 0; RayIdx < NumRaysToShoot; ++RayIdx)
	{
		FLyraPenetrationAvoidanceFeeler& Feeler = PenetrationAvoidanceFeelers[RayIdx];
		if (Feeler.FramesUntilNextTrace <= 0)
		{
			// calc ray target
			FVector RayTarget;
			{
				FVector RotatedRay = BaseRay.RotateAngleAxis(Feeler.AdjustmentRot.Yaw, BaseRayLocalUp);
				RotatedRay = RotatedRay.RotateAngleAxis(Feeler.AdjustmentRot.Pitch, BaseRayLocalRight);
				RayTarget = SafeLoc + RotatedRay;
			}

			// cast for world and pawn hits separately.  this is so we can safely ignore the 
			// camera's target pawn
			SphereShape.Sphere.Radius = Feeler.Extent;
			ECollisionChannel TraceChannel = ECC_Camera;		//(Feeler.PawnWeight > 0.f) ? ECC_Pawn : ECC_Camera;

			// do multi-line check to make sure the hits we throw out aren't
			// masking real hits behind (these are important rays).

			// MT-> passing camera as actor so that camerablockingvolumes know when it's the camera doing traces
			FHitResult Hit;
			const bool bHit = World->SweepSingleByChannel(Hit, SafeLoc, RayTarget, FQuat::Identity, TraceChannel, SphereShape, SphereParams);
#if ENABLE_DRAW_DEBUG
			if (World->TimeSince(LastDrawDebugTime) < 1.f)
			{
				DrawDebugSphere(World, SafeLoc, SphereShape.Sphere.Radius, 8, FColor::Red);
				DrawDebugSphere(World, bHit ? Hit.Location : RayTarget, SphereShape.Sphere.Radius, 8, FColor::Red);
				DrawDebugLine(World, SafeLoc, bHit ? Hit.Location : RayTarget, FColor::Red);
			}
#endif // ENABLE_DRAW_DEBUG

			Feeler.FramesUntilNextTrace = Feeler.TraceInterval;

			const AActor* HitActor = Hit.GetActor();

			if (bHit && HitActor)
			{
				bool bIgnoreHit = false;

				if (HitActor->ActorHasTag(LyraCameraMode_ThirdPerson_Statics::NAME_IgnoreCameraCollision))
				{
					bIgnoreHit = true;
					SphereParams.AddIgnoredActor(HitActor);
				}

				// Ignore CameraBlockingVolume hits that occur in front of the ViewTarget.
				if (!bIgnoreHit && HitActor->IsA<ACameraBlockingVolume>())
				{
					const FVector ViewTargetForwardXY = ViewTarget.GetActorForwardVector().GetSafeNormal2D();
					const FVector ViewTargetLocation = ViewTarget.GetActorLocation();
					const FVector HitOffset = Hit.Location - ViewTargetLocation;
					const FVector HitDirectionXY = HitOffset.GetSafeNormal2D();
					const float DotHitDirection = FVector::DotProduct(ViewTargetForwardXY, HitDirectionXY);
					if (DotHitDirection > 0.0f)
					{
						bIgnoreHit = true;
						// Ignore this CameraBlockingVolume on the remaining sweeps.
						SphereParams.AddIgnoredActor(HitActor);
					}
					else
					{
#if ENABLE_DRAW_DEBUG
						DebugActorsHitDuringCameraPenetration.AddUnique(TObjectPtr<const AActor>(HitActor));
#endif
					}
				}
				
				if (!bIgnoreHit)
				{
					float const Weight = Cast<APawn>(Hit.GetActor()) ? Feeler.PawnWeight : Feeler.WorldWeight;
					float NewBlockPct = Hit.Time;
					NewBlockPct += (1.f - NewBlockPct) * (1.f - Weight);

					// Recompute blocked pct taking into account pushout distance.
					NewBlockPct = ((Hit.Location - SafeLoc).Size() - CollisionPushOutDistance) / (RayTarget - SafeLoc).Size();
					DistBlockedPctThisFrame = FMath::Min(NewBlockPct, DistBlockedPctThisFrame);

					// This feeler got a hit, so do another trace next frame
					Feeler.FramesUntilNextTrace = 0;

#if ENABLE_DRAW_DEBUG
					DebugActorsHitDuringCameraPenetration.AddUnique(TObjectPtr<const AActor>(HitActor));
#endif
				}
			}

			if (RayIdx == 0)
			{
				// don't interpolate toward this one, snap to it
				// assumes ray 0 is the center/main ray 
				HardBlockedPct = DistBlockedPctThisFrame;
			}
			else
			{
				SoftBlockedPct = DistBlockedPctThisFrame;
			}
		}
		else
		{
			--Feeler.FramesUntilNextTrace;
		}
	}

	if (bResetInterpolation)
	{
		DistBlockedPct = DistBlockedPctThisFrame;
	}
	else if (DistBlockedPct < DistBlockedPctThisFrame)
	{
		// interpolate smoothly out
		if (PenetrationBlendOutTime > DeltaTime)
		{
			DistBlockedPct = DistBlockedPct + DeltaTime / PenetrationBlendOutTime * (DistBlockedPctThisFrame - DistBlockedPct);
		}
		else
		{
			DistBlockedPct = DistBlockedPctThisFrame;
		}
	}
	else
	{
		if (DistBlockedPct > HardBlockedPct)
		{
			DistBlockedPct = HardBlockedPct;
		}
		else if (DistBlockedPct > SoftBlockedPct)
		{
			// interpolate smoothly in
			if (PenetrationBlendInTime > DeltaTime)
			{
				DistBlockedPct = DistBlockedPct - DeltaTime / PenetrationBlendInTime * (DistBlockedPct - SoftBlockedPct);
			}
			else
			{
				DistBlockedPct = SoftBlockedPct;
			}
		}
	}

	DistBlockedPct = FMath::Clamp<float>(DistBlockedPct, 0.f, 1.f);
	if (DistBlockedPct < (1.f - ZERO_ANIMWEIGHT_THRESH))
	{
		CameraLoc = SafeLoc + (CameraLoc - SafeLoc) * DistBlockedPct;
	}
}
```

# ALyraPlayerCameraManager
继承于引擎的APlayerCameraManager
## 新增加的变量
后面介绍
```cpp
private:  
    /** The UI Camera Component, controls the camera when UI is doing something important that gameplay doesn't get priority over. */  
    UPROPERTY(Transient)  
    TObjectPtr<ULyraUICameraManagerComponent> UICamera;
```

## 重写的函数
- UpdateViewTarget：World的Tick链路中的一环，如果UICamera需要更新ViewTarget，会在基类Super::UpdateViewTarget(OutVT, DeltaTime)后调用UICamera->UpdateViewTarget(OutVT, DeltaTime); ^862edc
- DisplayDebug：先调用基类APlayerCameraManager的DisplayDebug，然后调用CameraComponent的DrawDebug；

## 新增加的函数
- GetUICameraComponent：返回UICamera。

# ULyraUICameraManagerComponent
作为ALyraPlayerCameraManager上的成员变量，在其构造函数中被构造
```cpp
ALyraPlayerCameraManager::ALyraPlayerCameraManager(const FObjectInitializer& ObjectInitializer)  
    : Super(ObjectInitializer)  
{  
    DefaultFOV = LYRA_CAMERA_DEFAULT_FOV;  
    ViewPitchMin = LYRA_CAMERA_DEFAULT_PITCH_MIN;  
    ViewPitchMax = LYRA_CAMERA_DEFAULT_PITCH_MAX;  
  
    UICamera = CreateDefaultSubobject<ULyraUICameraManagerComponent>(UICameraComponentName);  
}
```
## 新增内部变量
```cpp
UPROPERTY(Transient)  
TObjectPtr<AActor> ViewTarget;  
  
UPROPERTY(Transient)  
bool bUpdatingViewTarget;
```

- ViewTarget：在ULyraUICameraManagerComponent::SetViewTarget中设置成传入的参数，接着再传给Owner，也就是ALyraPlayerCameraManager，然后再SetViewTarget。
- bUpdatingViewTarget：在IsSettingViewTarget中返回，但是用到这个函数的地方在项目中没有找到。

## 新增的函数
- NeedsToUpdateViewTarget()：默认返回一个false。为真的时候会![[Lyra中的相机系统#^862edc]]
- UpdateViewTarget(struct FTViewTarget& OutVT, float DeltaTime)：空实现，会在上面流程中调用。需要我们根据项目去拓展了。
- OnShowDebugInfo(AHUD* HUD, UCanvas* Canvas, const FDebugDisplayInfo& DisplayInfo, float& YL, float& YPos)：空实现，
  在构造函数中绑定到AHud的OnShowDebugInfo委托上
  ```cpp
    ULyraUICameraManagerComponent::ULyraUICameraManagerComponent()
	{
		bWantsInitializeComponent = true;
	
		if (!HasAnyFlags(RF_ClassDefaultObject))
		{
			// Register "showdebug" hook.
			if (!IsRunningDedicatedServer())
			{
				AHUD::OnShowDebugInfo.AddUObject(this, &ThisClass::OnShowDebugInfo);
			}
		}
	}
  ```

- 剩下三个函数虽然定义了但是没有调用的地方：
  ```cpp
    bool IsSettingViewTarget() const { return bUpdatingViewTarget; }  
	AActor* GetViewTarget() const { return ViewTarget; }  
	void SetViewTarget(AActor* InViewTarget, FViewTargetTransitionParams TransitionParams = FViewTargetTransitionParams());
  ```