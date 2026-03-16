事件的起因是我在做技能相关需求的时候，需要在GA中获取WorldContextObject，最开始的GA在运行时是不实例化的，只是作为一个模版类来新生成一个GA实例。GA的运行时数据是在AbilitySpec中的，如果使用AbilitySpec中的Ability其实就是一个CDO。

# 是否需要在CDO中获取WorldContextObject来调用相关子系统的方法？因为子系统中的方法的通用性。
获取子系统需要有WorldContextObject但是CDO不会存在于一个世界中。所以在CDO中获取WorldContextObject是不可行的。

# 为什么会遇到这个问题？
想判断单个GA实例能否被激活

# 预判断GA能否激活实现步骤
直接Spec.Ability获取到的不是实例而是CDO，所以使用的应该是GASpec的GetPrimaryInstance，而不是一开始的GASpec.Ability。
![[GAS相关/在GA中获取WorldContextObject （Media）/1.png]]

我重写了Ability的GetCostGameplayEffect方法
![[GAS相关/在GA中获取WorldContextObject （Media）/2.png]]

我会对默认的CostGE的CDO进行些额外操作。
![[GAS相关/在GA中获取WorldContextObject （Media）/3.png]]

我们可以查看一下Spec.GetPrimaryInstance()中的逻辑
在Ability->GetInstancingPolicy() == EGameplayAbilityInstancingPolicy::InstancedPerActor时返回NonReplicatedInstances 或 ReplicatedInstances中的元素
![[UnrealEngineNotes/GAS相关/在GA中获取WorldContextObject （Media）/4.png]]

NonReplicatedInstances 或 ReplicatedInstances中的元素在什么时候会被填充
查看源码知道会在CreateNewInstanceOfAbility中被填充
CreateNewInstanceOfAbility，就是创建GA实例的方法。
如果是EGameplayAbilityInstancingPolicy::InstancedPerActor的GA会在GiveAbility的时候就实例化，InstancedPerExecution的话会在激活这个能力的时候实例化。
而只有实例化的GA 的 CurrentActorInfo才不会为空
![[UnrealEngineNotes/GAS相关/在GA中获取WorldContextObject （Media）/5.png]]

如果我们在CDO中跑逻辑（GASpec.Ability）
GetAvatarActorFromActorInfo()就有可能为空
CurrentActorInfo会在UGameplayAbility::OnGiveAbility和UGameplayAbility::PreActivate的时候被赋值，但是需要实例化的能力。
所以实例化的能力才能在非激活时，访问GetAvatarActorFromActorInfo()等WorldContextObject.

end...