GD是github上GasDocument项目中的工程文件
# AbilityTask
这个工程中主要写了两个AT分别是：
- UGDAT_PlayMontageAndWaitForEvent
- UGDAT_WaitReceiveDamage
UGDAT_PlayMontageAndWaitForEvent和action rpg中的没啥区别
UGDAT_WaitReceiveDamage看名字就知道是用来接受伤害的。
实现如下
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/1.png]]

具体实现就是在AT激活的时候给GDASC上接收伤害的委托绑定回调，销毁的时候把asc上绑定上的回调清理了就行。一般是在ATEnd或是AT的owner也就是GA被End的时候执行这个AT的销毁逻辑。
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/2.png]]

追溯ASC的调用这个委托的源头是在伤害的计算类中的
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/3.png]]

action rpg的伤害计算类中没有给asc转发这个接收伤害委托的操作，直接给角色去处理伤害了，可能是工程没有需要在GA处理伤害回调的需求把

我们看下使用到的ga用这个回调来做什么需求，在蓝图中有一个能力叫GA_PassiveArmor_BP是被动护甲的能力。而在action rpg中是没有被动护甲这样的能力的，而是只有一个防御力的属性在属性集中。

这个被动能力就是在每次接收伤害的时候移除一层护甲ge的Tag
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/4.png]]
然后再结束ga的时候end一下task，虽然引擎会自动完成这步操作。

# AttributeSet
先看下AS中的PreAttributeChange和PostGameplayEffectExecute
PreAttributeChange中是一些属性集的夹值操作和根据最大值进行整体调整和action rpg中一样，和GAS源码中的的注释一致推荐在这里夹值。
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/5.png]]

PostGameplayEffectExecute太长了直接贴源码
```cpp
void UGDAttributeSetBase::PostGameplayEffectExecute(const FGameplayEffectModCallbackData & Data)
{
    Super::PostGameplayEffectExecute(Data);

    FGameplayEffectContextHandle Context = Data.EffectSpec.GetContext();
    UAbilitySystemComponent* Source = Context.GetOriginalInstigatorAbilitySystemComponent();
    const FGameplayTagContainer& SourceTags = *Data.EffectSpec.CapturedSourceTags.GetAggregatedTags();
    FGameplayTagContainer SpecAssetTags;
    Data.EffectSpec.GetAllAssetTags(SpecAssetTags);

    // Get the Target actor, which should be our owner
    AActor* TargetActor = nullptr;
    AController* TargetController = nullptr;
    AGDCharacterBase* TargetCharacter = nullptr;
    if (Data.Target.AbilityActorInfo.IsValid() && Data.Target.AbilityActorInfo->AvatarActor.IsValid())
    {
       TargetActor = Data.Target.AbilityActorInfo->AvatarActor.Get();
       TargetController = Data.Target.AbilityActorInfo->PlayerController.Get();
       TargetCharacter = Cast<AGDCharacterBase>(TargetActor);
    }

    // Get the Source actor
    AActor* SourceActor = nullptr;
    AController* SourceController = nullptr;
    AGDCharacterBase* SourceCharacter = nullptr;
    if (Source && Source->AbilityActorInfo.IsValid() && Source->AbilityActorInfo->AvatarActor.IsValid())
    {
       SourceActor = Source->AbilityActorInfo->AvatarActor.Get();
       SourceController = Source->AbilityActorInfo->PlayerController.Get();
       if (SourceController == nullptr && SourceActor != nullptr)
       {
          if (APawn* Pawn = Cast<APawn>(SourceActor))
          {
             SourceController = Pawn->GetController();
          }
       }

       // Use the controller to find the source pawn
       if (SourceController)
       {
          SourceCharacter = Cast<AGDCharacterBase>(SourceController->GetPawn());
       }
       else
       {
          SourceCharacter = Cast<AGDCharacterBase>(SourceActor);
       }

       // Set the causer actor based on context if it's set
       if (Context.GetEffectCauser())
       {
          SourceActor = Context.GetEffectCauser();
       }
    }

    if (Data.EvaluatedData.Attribute == GetDamageAttribute())
    {
       // Try to extract a hit result
       FHitResult HitResult;
       if (Context.GetHitResult())
       {
          HitResult = *Context.GetHitResult();
       }

       // Store a local copy of the amount of damage done and clear the damage attribute
       const float LocalDamageDone = GetDamage();
       SetDamage(0.f);
    
       if (LocalDamageDone > 0.0f)
       {
          // If character was alive before damage is added, handle damage
          // This prevents damage being added to dead things and replaying death animations
          bool WasAlive = true;

          if (TargetCharacter)
          {
             WasAlive = TargetCharacter->IsAlive();
          }

          if (!TargetCharacter->IsAlive())
          {
             //UE_LOG(LogTemp, Warning, TEXT("%s() %s is NOT alive when receiving damage"), TEXT(__FUNCTION__), *TargetCharacter->GetName());
          }

          // Apply the health change and then clamp it
          const float NewHealth = GetHealth() - LocalDamageDone;
          SetHealth(FMath::Clamp(NewHealth, 0.0f, GetMaxHealth()));

          if (TargetCharacter && WasAlive)
          {
             // This is the log statement for damage received. Turned off for live games.
             //UE_LOG(LogTemp, Log, TEXT("%s() %s Damage Received: %f"), TEXT(__FUNCTION__), *GetOwningActor()->GetName(), LocalDamageDone);

             // Play HitReact animation and sound with a multicast RPC.
             const FHitResult* Hit = Data.EffectSpec.GetContext().GetHitResult();

             if (Hit)
             {
                EGDHitReactDirection HitDirection = TargetCharacter->GetHitReactDirection(Data.EffectSpec.GetContext().GetHitResult()->Location);
                switch (HitDirection)
                {
                case EGDHitReactDirection::Left:
                   TargetCharacter->PlayHitReact(HitDirectionLeftTag, SourceCharacter);
                   break;
                case EGDHitReactDirection::Front:
                   TargetCharacter->PlayHitReact(HitDirectionFrontTag, SourceCharacter);
                   break;
                case EGDHitReactDirection::Right:
                   TargetCharacter->PlayHitReact(HitDirectionRightTag, SourceCharacter);
                   break;
                case EGDHitReactDirection::Back:
                   TargetCharacter->PlayHitReact(HitDirectionBackTag, SourceCharacter);
                   break;
                }
             }
             else
             {
                // No hit result. Default to front.
                TargetCharacter->PlayHitReact(HitDirectionFrontTag, SourceCharacter);
             }

             // Show damage number for the Source player unless it was self damage
             if (SourceActor != TargetActor)
             {
                AGDPlayerController* PC = Cast<AGDPlayerController>(SourceController);
                if (PC)
                {
                   PC->ShowDamageNumber(LocalDamageDone, TargetCharacter);
                }
             }

             if (!TargetCharacter->IsAlive())
             {
                // TargetCharacter was alive before this damage and now is not alive, give XP and Gold bounties to Source.
                // Don't give bounty to self.
                if (SourceController != TargetController)
                {
                   // Create a dynamic instant Gameplay Effect to give the bounties
                   UGameplayEffect* GEBounty = NewObject<UGameplayEffect>(GetTransientPackage(), FName(TEXT("Bounty")));
                   GEBounty->DurationPolicy = EGameplayEffectDurationType::Instant;

                   int32 Idx = GEBounty->Modifiers.Num();
                   GEBounty->Modifiers.SetNum(Idx + 2);

                   FGameplayModifierInfo& InfoXP = GEBounty->Modifiers[Idx];
                   InfoXP.ModifierMagnitude = FScalableFloat(GetXPBounty());
                   InfoXP.ModifierOp = EGameplayModOp::Additive;
                   InfoXP.Attribute = UGDAttributeSetBase::GetXPAttribute();

                   FGameplayModifierInfo& InfoGold = GEBounty->Modifiers[Idx + 1];
                   InfoGold.ModifierMagnitude = FScalableFloat(GetGoldBounty());
                   InfoGold.ModifierOp = EGameplayModOp::Additive;
                   InfoGold.Attribute = UGDAttributeSetBase::GetGoldAttribute();

                   Source->ApplyGameplayEffectToSelf(GEBounty, 1.0f, Source->MakeEffectContext());
                }
             }
          }
       }
    }// Damage
    else if (Data.EvaluatedData.Attribute == GetHealthAttribute())
    {
       // Handle other health changes.
       // Health loss should go through Damage.
       SetHealth(FMath::Clamp(GetHealth(), 0.0f, GetMaxHealth()));
    } // Health
    else if (Data.EvaluatedData.Attribute == GetManaAttribute())
    {
       // Handle mana changes.
       SetMana(FMath::Clamp(GetMana(), 0.0f, GetMaxMana()));
    } // Mana
    else if (Data.EvaluatedData.Attribute == GetStaminaAttribute())
    {
       // Handle stamina changes.
       SetStamina(FMath::Clamp(GetStamina(), 0.0f, GetMaxStamina()));
    }
}
```

其实gd中PostGameplayEffectExecute和action rpg差别不大
在SetHealth之前也要做一些夹值操作。gd文档中提过在这里夹值是在同步给客户端之前，所以也是可以的。
这里在伤害应用给玩家健康值后有一步给角色受击反馈的操作
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/6.png]]

这里有个显示伤害数值和给照成伤害的人赏金的操作，给赏金是动态生成ge然后应用上去的。
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/7.png]]

后面就是一些属性的夹值操作很常规

# 蓝图异步行为
UAsyncTaskAttributeChanged，
UAsyncTaskCooldownChanged
UAsyncTaskEffectStackChanged

UAsyncTaskAttributeChanged这个节点中实现了两种类型的参数的函数：单个属性和属性数组。
两个核心方法的实现，主要还是依赖asc上的GetGameplayAttributeValueChangeDelegate这个委托
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/8.png]]

这在HUD中使用到，这也是gd文档中8.2中提到的点
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/9.png]]

然后是UAsyncTaskCooldownChanged中的核心函数这里主要使用了asc上的OnActiveGameplayEffectAddedDelegateToSelf和RegisterGameplayTagEvent(CooldownTag, EGameplayTagEventType::NewOrRemoved)返回的委托
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/10.png]]

冷却开始的回调
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/11.png]]

冷却tag被移除的时候触发的回调
下面的函数和action rpg中差不多
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/12.png]]

蓝图中的使用，冷却开始的时候，剩余时间小于0说明是本地预测的，大于0是服务器开始冷却的时刻
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/13.png]]

这是代码中写死的
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/14.png]]
然后开一个timer在客户端递减剩余时间在客户端小于0的时候点亮技能图标
这只是ui的显示实际控制ga的冷却还是在ga的cooldown中的ge。

接着是UAsyncTaskEffectStackChanged
要监听ge的stack改变主要是依赖ASC上的OnActiveGameplayEffectAddedDelegateToSelf和OnAnyGameplayEffectRemovedDelegate()委托
堆栈只在持续和无限的ge上发生。
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/15.png]]

ge add后再绑定asc上的OnGameplayEffectStackChangeDelegate监听ge堆栈改变。
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/16.png]]

也是在蓝图的UI_HUD中用到
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/17.png]]
总结这种和asc相关的回调事件都可以封装成异步节点，但是普通蓝图用不了AT类型的异步节点，只能封装成蓝图异步节点。

# ASC
ASC的设计很少只有接收伤害然后调用下委托
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/18.png]]

![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/19.png]]

# 伤害传递
大致和Action RPG上差不多但是多了一步接收外部传入的伤害数值
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/20.png]]

看下枪的ge没设置任何modifier，也没有setbycaller的选项
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/21.png]]

是在GA基类中动态赋予的
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/22.png]]

在蓝图中动态赋予
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/23.png]]

gd是没在PostGEExecute中执行通知角色接收伤害的
只有处理一些受击的反馈，角色伤害的接收是在geec中直接处理
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/24.png]]

这个项目中大量直接使用outgoing ge而不是像action rpg那样使用ge container

# GA
跳跃能力扩展的地方很少
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/25.png]]
这里有个预测key的概念
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/26.png]]

![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/27.png]]

这里有个域锁的概念这里有段注释
官方注释
取消非实例化能力很棘手。现在，这适用于跳跃，因为如果您还没有跳跃，那么调用StopJumping()不会出错。如果我们有一个蒙太奇播放非实例化能力，它需要确保它播放的蒙太奇仍在播放，如果是，则取消它。如果这是我们需要支持的东西，我们可能需要一些轻量级的数据结构来表示“非实例化的能力正在执行中”，并有一种取消/结束它们的方法。
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/28.png]]

ga的基类
加了同样是EGDAbilityInputID类型的AbilityInputID和AbilityID
这里是用InputID和输入相关联，而AbilityID是和插槽相关联。
加了ActivateAbilityOnGranted标识来代表是别动技能，被赋予时直接激活。
重写了OnAvatarSet，因为会在ga被赋予后调用所以在这里判断是否可以激活被动技能。
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/29.png]]

给ga加上默认阻挡标签，死亡和眩晕。
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/30.png]]

火枪GA这里AbilityTag和ActivationOwnedTag用的是同一个tag
blocktag 是加了所有技能
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/31.png]]

这里通过匹配asc上的标签判断是否使用下蹲的动画。
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/32.png]]

C++中构造AT要手动激活
蓝图中会被编辑器自动激活
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/33.png]]

这里有一处和我们平时使用不同的地方
PlayMontageAndWaitForEvent的参数传入的FGameplayTagContainer是空构造
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/34.png]]

接收montage事件的回调
我们在这里过滤tag，而不是在AT的参数中过滤
这里也是动态创建ge spec但是和之前动态赋予的方式不太一样。spec和ge赋予的区别
```cpp
void UGDGA_FireGun::EventReceived(FGameplayTag EventTag, FGameplayEventData EventData)
{
    // Event.Montage.EndAbility=蒙太奇告诉我们在蒙太奇结束之前结束这个能力。
    // 蒙太奇被设置为在技能结束后继续播放动画，所以这是可以的。
    if (EventTag == FGameplayTag::RequestGameplayTag(FName("Event.Montage.EndAbility")))
    {
       EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, false);
       return;
    }

    // Event.Montage.SpawnProjectile=生成子弹，仅在服务器上生成子弹
    if (GetOwningActorFromActorInfo()->GetLocalRole() == ROLE_Authority && EventTag == FGameplayTag::RequestGameplayTag(FName("Event.Montage.SpawnProjectile")))
    {
       AGDHeroCharacter* Hero = Cast<AGDHeroCharacter>(GetAvatarActorFromActorInfo());
       //如果获取角色失败，就结束能力
       if (!Hero)
       {
          EndAbility(CurrentSpecHandle, CurrentActorInfo, CurrentActivationInfo, true, true);
       }
       //找到枪口，用于发射子弹的起始点
       FVector Start = Hero->GetGunComponent()->GetSocketLocation(FName("Muzzle"));
       //根据射击范围确定子弹终点
       FVector End = Hero->GetCameraBoom()->GetComponentLocation() + Hero->GetFollowCamera()->GetForwardVector() * Range;
       FRotator Rotation = UKismetMathLibrary::FindLookAtRotation(Start, End);

       FGameplayEffectSpecHandle DamageEffectSpecHandle = MakeOutgoingGameplayEffectSpec(DamageGameplayEffect, GetAbilityLevel());
       
       // 通过GameplayEffectSpec上的SetByCaller值将伤害传递给伤害执行计算
       DamageEffectSpecHandle.Data.Get()->SetSetByCallerMagnitude(FGameplayTag::RequestGameplayTag(FName("Data.Damage")), Damage);

       FTransform MuzzleTransform = Hero->GetGunComponent()->GetSocketTransform(FName("Muzzle"));
       MuzzleTransform.SetRotation(Rotation.Quaternion());
       MuzzleTransform.SetScale3D(FVector(1.0f));

       FActorSpawnParameters SpawnParameters;
       SpawnParameters.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
       //生成子弹
       AGDProjectile* Projectile = GetWorld()->SpawnActorDeferred<AGDProjectile>(ProjectileClass, MuzzleTransform, GetOwningActorFromActorInfo(),
          Hero, ESpawnActorCollisionHandlingMethod::AlwaysSpawn);
       Projectile->DamageEffectSpecHandle = DamageEffectSpecHandle;
       Projectile->Range = Range;
       Projectile->FinishSpawning(MuzzleTransform);
    }
}
```

# 输入绑定
在GD角色类中先是构造FTopLevelAssetPath拿到EGDAbilityInputID这个枚举
这个是对应ue旧的输入系统中的配置，这步是绑定技能的确认和取消按键
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/35.png]]

在角色在服务器被占有的时候初始化能力
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/36.png]]

在赋予能力的时候绑定输入
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/37.png]]

# 监听Tag的变化
在监听冷却改变的蓝图异步任务中
监听冷却改变
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/38.png]]

小兵初始化的时候监听眩晕的标签
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/39.png]]

触发后会取消某些能力
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/40.png]]

PlayerState初始化的时候也是监听眩晕的标签
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/41.png]]

同样触发的时候取消些能力
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/42.png]]

# GE资源Tag
有的ge会有ge的资源tag，就是在死亡的时候移除这个ge
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/43.png]]

GD角色类中有这样一段代码
用RemoveActiveEffectsWithTags这个接口
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/44.png]]

asset tag也也会被纳入考虑
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/45.png]]

# C++中监听属性改变

![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/46.png]]

![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/47.png]]

# CharacterMovementComponent
在GD的CMC中定义了自定义的FSavedMove_Character
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/48.png]]

和自定义的FNetworkPredictionData_Client_Character
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/49.png]]

GD CMC中在服务器一直更新flag
需要看这篇文章[https://zhuanlan.zhihu.com/p/694601656](https://zhuanlan.zhihu.com/p/694601656)
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/50.png]]

savemove的一些操作
清理和比对标识
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/51.png]]

判断是否需要合并savemove
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/52.png]]

重写FGDNetworkPredictionData_Client::AllocateNewMove为返回我们自定义的savedmove
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/53.png]]

# 处理Scoped Prediction Key 无效的情况

sprint的GA使用了wait net sync节点
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/55.png]]

主要为了解决wait delay后Scoped Prediction Key 无效的情况
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/54.png]]

看了sprint的ga中应用的ge是没有更改移动的属性的，主要是加sprint的tag
主要都是在移动组件中做修改
![[UnrealEngineNotes/开源项目分析/GD中的GAS设计/56.png]]