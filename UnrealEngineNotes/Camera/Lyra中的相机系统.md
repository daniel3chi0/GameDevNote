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
LyraCameraComponent中的成员变量：
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
- CameraModeStack变量名字可以很明显的看出是个栈结构。内部维护两个`ULyraCameraMode`类型数组变量CameraModeInstances，CameraModeStack。