在我的插件SkillGasSupport中，实现了一个在客户端侧监听任意按键的，任意触发事件的AbilityTask。
```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "Abilities/Tasks/AbilityTask.h"
#include "AbilityTask_ListenKeyAndWaitInputEvent.generated.h"

DECLARE_DYNAMIC_MULTICAST_DELEGATE(FListenKeyInputEventDelegate);
/**
 * 
 */
UCLASS()
class WESTERN_MAP_API UAbilityTask_ListenKeyAndWaitInputEvent : public UAbilityTask
{
	GENERATED_BODY()

public:
	
	UAbilityTask_ListenKeyAndWaitInputEvent(const FObjectInitializer& ObjectInitializer);

	UFUNCTION(BlueprintCallable, Category = "Ability|Tasks", meta = (HidePin = "OwningAbility", DefaultToSelf = "OwningAbility", BlueprintInternalUseOnly = "TRUE"))
	static UAbilityTask_ListenKeyAndWaitInputEvent* ListenKeyAndWaitInputEvent(UGameplayAbility* OwningAbility, FKey ListenKeys, EInputEvent InputEvent);

protected:
	
	virtual void Activate() override;

	virtual void OnDestroy(bool bInOwnerFinished) override;

private:

	FKey ListenKey;

	EInputEvent InputEvent;

	FInputKeyBinding InputKeyBinding;

	int32 CacheIndex;

	static int32 RemoveIndex;

public:

	UPROPERTY(BlueprintAssignable)
	FListenKeyInputEventDelegate OnKeyInputEventCallBack;

private:

	void OnICBindKeyInputEventCallBack();
};

```

```cpp
// Fill out your copyright notice in the Description page of Project Settings.


#include "AbilitySystem/AbilityTasks/AbilityTask_ListenKeyAndWaitInputEvent.h"

int32 UAbilityTask_ListenKeyAndWaitInputEvent::RemoveIndex = 0;

UAbilityTask_ListenKeyAndWaitInputEvent::UAbilityTask_ListenKeyAndWaitInputEvent(const FObjectInitializer& ObjectInitializer)
{
	ListenKey = EKeys::AnyKey;
	InputEvent = EInputEvent::IE_Pressed;
}

UAbilityTask_ListenKeyAndWaitInputEvent* UAbilityTask_ListenKeyAndWaitInputEvent::ListenKeyAndWaitInputEvent(UGameplayAbility* OwningAbility, FKey ListenKey,
	EInputEvent InputEvent)
{
	UAbilityTask_ListenKeyAndWaitInputEvent* Task = NewAbilityTask<UAbilityTask_ListenKeyAndWaitInputEvent>(OwningAbility);
	Task->ListenKey = ListenKey;
	Task->InputEvent = InputEvent;
	return Task;
}

void UAbilityTask_ListenKeyAndWaitInputEvent::Activate()
{
	if(!Ability) return;

	AActor* Avatar = Ability->GetAvatarActorFromActorInfo();
	if(Avatar)
	{
		if(Avatar->InputComponent)
		{
			InputKeyBinding = Avatar->InputComponent->BindKey(ListenKey, InputEvent, this, &UAbilityTask_ListenKeyAndWaitInputEvent::OnICBindKeyInputEventCallBack);
			CacheIndex = Avatar->InputComponent->KeyBindings.Num() - 1;
		}
	}
}

void UAbilityTask_ListenKeyAndWaitInputEvent::OnDestroy(bool bInOwnerFinished)
{
	if(Ability)
	{
		AActor* Avatar = Ability->GetAvatarActorFromActorInfo();
		if(Avatar)
		{
			if(Avatar->InputComponent)
			{
				if(RemoveIndex > 0 && RemoveIndex < CacheIndex)
					CacheIndex = RemoveIndex;
				Avatar->InputComponent->KeyBindings.RemoveAt(CacheIndex, EAllowShrinking::No);
				RemoveIndex = CacheIndex;
			}
		}
	}
	
	Super::OnDestroy(bInOwnerFinished);
}

void UAbilityTask_ListenKeyAndWaitInputEvent::OnICBindKeyInputEventCallBack()
{
	if(ShouldBroadcastAbilityTaskDelegates())
	{
		if(OnKeyInputEventCallBack.IsBound())
		{
			OnKeyInputEventCallBack.Broadcast();
		}
	}

	EndTask();
}
```

但是存在缺点只能在客户端侧监听到触发按键的回调，没实现在服务器侧触发按键的回调。其实在GASShooter中有个UGSAT_WaitInputPressWithTags的AbilityTask中就有同样的需求，采用的是方案是使用GAS中的**GenericReplicatedEvent**，只不过这个InputPressed的事件的枚举是引擎中刚好有的，在GameplayAbilityTargetType中有如下代码：
```cpp
UENUM()  
namespace EAbilityGenericReplicatedEvent  
{  
    enum Type : int  
    {    
/** A generic confirmation to commit the ability */  
       GenericConfirm = 0,  
       /** A generic cancellation event. Not necessarily a canellation of the ability or targeting. Could be used to cancel out of a channelling portion of ability. */  
       GenericCancel,  
       /** Additional input presses of the ability (Press X to activate ability, press X again while it is active to do other things within the GameplayAbility's logic) */  
       InputPressed,    
/** Input release event of the ability */  
       InputReleased,  
       /** A generic event from the client */  
       GenericSignalFromClient,  
       /** A generic event from the server */  
       GenericSignalFromServer,  
       /** Custom events for game use */  
       GameCustom1,  
       GameCustom2,  
       GameCustom3,  
       GameCustom4,  
       GameCustom5,  
       GameCustom6,  
       MAX  
    };  
}
```

前面几个有命名的枚举基本上都是用于其他AbilityTask的专用复制事件：
- GenericConfirm，GenericCancel用于TargetActor和相关的AbilityTask
- InputPressed，InputReleased用于Ga的WaitInputPressed任务，WaitInputReleased任务。
- GenericSignalFromClient，GenericSignalFromServer用于WaitNetSync任务。

我们只能使用后面的GameCustom的事件来充当我们自定义类型的ReplicateEvent。这个结构体是写在引擎的代码中的，在正式项目中使用可读性很差。**GameCustom1 这种名字在多个 Task 共用时完全不知道谁是谁**。
`EAbilityGenericReplicatedEvent` 是一个 `namespace` 内的普通 C++ `enum`（非 `enum class`），**C++ 语言层面无法继承或扩展 enum**。
# 解法：Alias 映射层

不扩展枚举本身，而是在你自己的项目代码里建一个**语义映射层**，把晦涩的 `GameCustom1~6` 映射为有业务含义的名字：
```cpp
// MyGameReplicatedEvents.h
// 项目内的语义别名，集中管理 GameCustom 槽位占用情况
namespace EMyReplicatedEvent
{
    // 清晰记录每个槽位被哪个 Task 占用，防止多个 Task 抢同一个槽位
    
    // GameCustom1 -> 监听任意键输入
    constexpr EAbilityGenericReplicatedEvent::Type ListenKeyInput
        = EAbilityGenericReplicatedEvent::GameCustom1;

    // GameCustom2 -> 技能蓄力完成信号
    constexpr EAbilityGenericReplicatedEvent::Type ChargeComplete
        = EAbilityGenericReplicatedEvent::GameCustom2;

    // GameCustom3 -> 瞄准确认
    constexpr EAbilityGenericReplicatedEvent::Type AimConfirmed
        = EAbilityGenericReplicatedEvent::GameCustom3;

    // GameCustom4~6 未占用，按需分配
}
```

使用时：
```cpp
// AbilityTask 中使用语义名，完全不出现 GameCustom1
ASC->ServerSetReplicatedEvent(
    EMyReplicatedEvent::ListenKeyInput,   // 一眼看懂
    GetAbilitySpecHandle(),
    GetActivationPredictionKey()
);

AbilitySystemComponent->GenericReplicatedEventDelegate(
    EMyReplicatedEvent::ListenKeyInput,   // 对应
    GetAbilitySpecHandle(),
    GetActivationPredictionKey()
).AddUObject(this, &UAbilityTask_ListenKeyAndWaitInputEvent::OnReplicatedEventReceived);
```

# 优化OnDestroy中的解绑事件

用静态索引来解绑，在多实例中可能存在互相污染的情况。可以考虑用这种方式解绑。
```cpp
void UAbilityTask_ListenKeyAndWaitInputEvent::OnDestroy(bool bInOwnerFinished)
{
    if(Ability)
    {
        AActor* Avatar = Ability->GetAvatarActorFromActorInfo();
        if(Avatar && Avatar->InputComponent)
        {
            // 通过 Delegate 对象指针匹配，找到自己注册的那条精确移除
            Avatar->InputComponent->KeyBindings.RemoveAll(
                [this](const FInputKeyBinding& Binding)
                {
                    return Binding.KeyDelegate.IsBoundToObject(this);
                }
            );
        }
    }
    Super::OnDestroy(bInOwnerFinished);
}
```