# Aura中的光标目标数据
在GAS中我们在客户端激活AbilityTask后如果是本地预测的话会在服务器上也激活AbilityTask。这个中间时间是设为ε。
然后在客户端侧接着执行把本地的TargetData发送给服务器的RPC，发送的时间设为δ。
现在分为两种情况：
- ε<δ：服务端激活AbilityTask的时候可能TargetData还没到。
- δ>ε：服务端激活AbilityTask的时候TargetData已经到了，但是绑定委托回调晚了。已经错过TargetData的广播的那一刻了。
这两种情况是无法确定的。
![[GAS相关/客户端向服务器发送TargetData（Media）/1.png]]

GAS中有一个内置的目标数据系统。可以通过ServerSetReplicatedTargetData()发送TargetData到服务器，通过服务器的FAbilityTargetDataSetDelegate去广播。
所以服务器上要绑定这个委托当收到的时候可以呼叫响应。服务器上还维护着一个AbilityTargetDataMap。key对应AbilitySpec，value对应的是TargetData。
![[GAS相关/客户端向服务器发送TargetData（Media）/2.png]]

现在我们使用这种方式：
如果ε<δ，那么在服务端Bind TargetSet Delegate 等待TargetData到来就没有问题。
![[GAS相关/客户端向服务器发送TargetData（Media）/3.png]]

如果ε>δ，那么TargetData先复制到服务器然后广播掉了，此后再绑定就没有意义了。此时使用这种方法：CallReplicatedTargetDataDelegateIfSet，这将强制服务器再次广播，并检索目标数据。
![[GAS相关/客户端向服务器发送TargetData（Media）/4.png]]

Aura中默认不向服务器发送TargetData数据的鼠标射线命中目标的实现如下（这里委托的参数暂时还是设置成const FVector& Data）。
服务器侧没办法获得光标所以没有HitResult。
![[GAS相关/客户端向服务器发送TargetData（Media）/6.png]]

![[GAS相关/客户端向服务器发送TargetData（Media）/5.png]]