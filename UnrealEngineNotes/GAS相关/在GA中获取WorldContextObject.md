事件的起因是我在做技能相关需求的时候，需要在GA中获取WorldContextObject，最开始的GA在运行时是不实例化的，只是作为一个模版类来新生成一个GA实例。GA的运行时数据是在AbilitySpec中的，如果使用AbilitySpec中的Ability其实就是一个CDO。

需要在CDO中获取WorldContextObject来调用相关子系统的方法？因为子系统中的方法的通用性。
而获取子系统需要有WorldContextObject但是CDO不会存在于一个世界中。所以在CDO中获取WorldContextObject是不可行的。

为什么会遇到这个问题？
想判断当个GA实例能否被激活

直接Spec.Ability获取到的不是实例而是CDO，所以使用的应该是GASpec的GetPrimaryInstance，而不是一开始的GASpec.Ability。
![[GAS相关/在GA中获取WorldContextObject （Media）/1.png]]

我重写了Ability的GetCostGameplayEffect方法
![[GAS相关/在GA中获取WorldContextObject （Media）/2.png]]

我会对默认的CostGE的CDO进行些额外操作。
![[GAS相关/在GA中获取WorldContextObject （Media）/3.png]]