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
  ```
- OnCameraPenetratingTarget()（相机正在穿透目标行为函数），在接口中无实现在`ALyraPlayerController`中重写如下：
  ```cpp
  void ALyraPlayerController::OnCameraPenetratingTarget()  
{  
    bHideViewTargetPawnNextFrame = true;  
}
  ```

# LyraCameraComponent
## LyraCameraComponent中的成员变量：
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

# LyraCameraMode
这个文件中定义了几个类或结构体
- **ULyraCameraMode**：UObject的子类在CameraModeStack中作为栈元素（即一组位置旋转混合数据），内部有几个比较重要的变量
  `FGameplayTag CameraTypeTag`：CameraMode对应的Tag。
  `FLyraCameraModeView View`：在下面
  `float BlendTime，float BlendAlpha，float BlendWeight`：
  - float BlendTime（在CameraMode构造函数中设置为0.5f），float BlendAlpha 在`void ULyraCameraMode::UpdateBlending(float DeltaTime)`中根据DeltaTime更新BlendWeight，调用栈如下：
   ```cpp
	  void ULyraCameraComponent::GetCameraView(float DeltaTime, FMinimalViewInfo& DesiredView)
		  void ULyraCameraComponent::UpdateCameraModes()
		  //...
		  bool ULyraCameraModeStack::EvaluateStack(float DeltaTime, FLyraCameraModeView& OutCameraModeView)
			  void ULyraCameraModeStack::UpdateStack(float DeltaTime)
				  void ULyraCameraMode::UpdateCameraMode(float DeltaTime)
					  void ULyraCameraMode::UpdateBlending(float DeltaTime)
   ```
  - `void ULyraCameraMode::SetBlendWeight(float Weight)`中根据传入的Weight设置BlendWeight并更新BlendAlpha，调用栈如下：
    ```cpp
    void ULyraCameraComponent::UpdateCameraModes()
	    void ULyraCameraModeStack::PushCameraMode(TSubclassOf<ULyraCameraMode> CameraModeClass)
		    void ULyraCameraMode::SetBlendWeight(float Weight)
    ```
    SetBlendWeight一定在UpdateBlending之前调用，计算出的BlendAlpha再用来计算UpdateBlending中的BlendWeight的。然后用这个UpdateBlending中的BlendWeight再计算下次SetBlendWeight的BlendAlpha。
- **ULyraCameraModeStack**：UObject的子类前面介绍了被LyraCameraComponent所有[[Lyra中的相机系统#^66829e]]。前面没提到的几个方法介绍：
  `bool EvaluateStack(float DeltaTime, FLyraCameraModeView& OutCameraModeView)`：里面主要是依次调用UpdateStack(DeltaTime); 
  BlendStack(OutCameraModeView);
  `ActivateStack()和DeactivateStack()`：遍历调用CameraMode中的OnActivation和OnDeactivation函数。
- **FLyraCameraModeView**：一个简单的结构体内部有FVector Location，FRotator Rotation，FRotator ControlRotation，float FieldOfView这几个变量。

## LyraCameraMode_ThridPerson
