Gas中有些地方会创建新的预测key会导致逻辑的异常
[https://zhuanlan.zhihu.com/p/422480523](https://zhuanlan.zhihu.com/p/422480523)
![[UnrealEngineNotes/GAS相关/GA预测key/1.png]]

Prediction是GAS系统中自带的预测机制是为了解决网络延迟而诞生。

我找到的网络上的介绍Prediction的资料如下。
知乎文章：预测流程[https://zhuanlan.zhihu.com/p/647874338](https://zhuanlan.zhihu.com/p/647874338)

GD项目中的Prediction部分：[https://github.com/tranek/GASDocumentation?tab=readme-ov-file#concepts-p](https://github.com/tranek/GASDocumentation?tab=readme-ov-file#concepts-p)

Stephen的GAS课程中的Prediction部分：[https://www.bilibili.com/video/BV1JD421E7yC?spm_id_from=333.788.videopod.episodes&vd_source=2d5f5b0c789f1187890235f6fa20045c&p=117](https://www.bilibili.com/video/BV1JD421E7yC?spm_id_from=333.788.videopod.episodes&vd_source=2d5f5b0c789f1187890235f6fa20045c&p=117)

最新的UFSH漫威争锋GAS应用演讲中的Prediction介绍：[https://www.bilibili.com/video/BV1VgWJzDEM8/?spm_id_from=333.337.search-card.all.click&vd_source=2d5f5b0c789f1187890235f6fa20045c](https://www.bilibili.com/video/BV1VgWJzDEM8/?spm_id_from=333.337.search-card.all.click&vd_source=2d5f5b0c789f1187890235f6fa20045c)

服务器有撤销预测结果的能力。
![[UnrealEngineNotes/GAS相关/GA预测key/2.png]]

我们在写GA的时候，如果逻辑中只是应用GE的话我们通常不加条件分支。如：

if (Authority) Do X

else：Do Predicted Version ...

在GA中应用GE就是实现了自动Prediction，所以我们不需要写上面的条件分支。
![[UnrealEngineNotes/GAS相关/GA预测key/3.png]]

实现自动预测的部分有如下：
![[UnrealEngineNotes/GAS相关/GA预测key/4.png]]

Prediction系统基于Prediction Key实现
GAS中会通过RPC把客户端发送Prediction Key 到服务器的事件称为 Replicate，这个和原生的网络框架中的Replicate的只是从服务器同步到客户端的原则不一样。在服务端收到预测键后会根据当前游戏状态选择接受或拒绝，然后通过FPredictionKey::NetSerialize() 把服务器收到的Prediction Key再Replicate到发起预测的客户端，而其他客户端中的发起预测能力的角色将收到一个无效的PredictionKey它的id为0。
![[UnrealEngineNotes/GAS相关/GA预测key/5.png]]

在预测的时候GA中会开启一个Prediction Window，在这个窗口中可以执行一系列的预测行为。
![[UnrealEngineNotes/GAS相关/GA预测key/6.png]]

以下是从客户端激活GA的流程
激活GA的时候会生成一个新的预测键，这个预测键和本次预测中的所有GE产生关联。服务器通过client RPC和返回服务器接受还是拒绝。同时ASC中还有一个FReplicatedPredictionKey 的变量用来复制预测键到客户端。
![[UnrealEngineNotes/GAS相关/GA预测key/7.png]]

查看引擎的代码发现ASC中现在有这个复制的变量，ReplicatedPredictionKeyMap变量
![[UnrealEngineNotes/GAS相关/GA预测key/8.png]]

可能是引擎代码迭代了
这个FReplicatedPredictionKeyMap，是用了FastArray来进行优化，保存了一个FReplicatedPreditionKeyItem的容器。
![[UnrealEngineNotes/GAS相关/GA预测key/9.png]]

Item被同步到客户端的时候会手动调用这里的OnRep函数，这两个结构就是Stephen视频中说的无疑了。
![[UnrealEngineNotes/GAS相关/GA预测key/10.png]]

GE被预测性的应用时会执行以下的操作
这个ASC中ReplicatedPredictionKey的作用是让复制到客户端时移除原来预测时候添加的side effect用的。
![[UnrealEngineNotes/GAS相关/GA预测key/11.png]]