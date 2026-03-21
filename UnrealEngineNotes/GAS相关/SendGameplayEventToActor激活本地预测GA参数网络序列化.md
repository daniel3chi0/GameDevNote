我们想给GA传参，比较好的解法应该是用GameplayEvent，触发GA的激活。利用FGameplayEventData Payload 的事件负载参数传递外部参数。（UAbilitySystemBlueprintLibrary::SendGameplayEventToActor）

如果要传入自定义的数据，我们认为比较好的方式就是利用FGameplayEventData中的OptionalObject
![[GAS相关/SendGameplayEventToActor激活本地预测GA参数网络序列化/1.png]]

我们要定义自己的UObject来包裹我们的数据
![[GAS相关/SendGameplayEventToActor激活本地预测GA参数网络序列化/2.png]]

在通过事件激活本地预测的能力时，也会在服务器激活一次
![[GAS相关/SendGameplayEventToActor激活本地预测GA参数网络序列化/3.png]]

如果我们只是简单用UObject包装我们的数据，在本地激活的能力是能拿到UObject中的数据的。但是在服务器侧激活的能力时在拿OptionalObject时拿到的是空的。这是因为我们自定义的UObject是不支持网络序列化的。

实现网络序列化UObject需要重写
- NetSerialize（自定义序列化函数）
- IsSupportedForNetworking（让UObject支持复制）
- TStructOpsTypeTraits（告诉引擎这个类支持序列化）

![[GAS相关/SendGameplayEventToActor激活本地预测GA参数网络序列化/4.png]]

实现自定义uobject的序列化
![[GAS相关/SendGameplayEventToActor激活本地预测GA参数网络序列化/5.png]]

如果在我们的UObject中存在自定义的表行数据
因为虚幻默认的FTableRowBase是不支持网络序列化的
所以我们同样要给我们自己派生的表行实现序列化函数
![[GAS相关/SendGameplayEventToActor激活本地预测GA参数网络序列化/6.png]]

这样我们通过事件激活的本地预测的能力的时候，在服务器侧生成的能力的时候就能在EventData 中的OptionalObject中拿到客户端侧通过网络序列化来的自定义Uobject参数了。

如果觉得UObject类型的参数开销太大，还可以该UObject中的数据包装成一个自定义的FGameplayAbilityTargetData的派生类（这是更好的方案）比起UObject少了一步IsSupportedForNetworking的重写。

![[GAS相关/SendGameplayEventToActor激活本地预测GA参数网络序列化/7.png]]

![[GAS相关/SendGameplayEventToActor激活本地预测GA参数网络序列化/8.png]]

![[GAS相关/SendGameplayEventToActor激活本地预测GA参数网络序列化/9.png]]

使用时如下
![[GAS相关/SendGameplayEventToActor激活本地预测GA参数网络序列化/10.png]]