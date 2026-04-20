我们在编写GAS相关的代码如AbilityTask这种，我们通常会创建预测窗口。一开始CurrentPredictionKey和ActivatedPredictionKey是相等的，创建新的预测窗口后CurrentPredictionKey就会更新。
ActivatedPredictionKey则是不变还是激活Ability时在客户端侧生成的预测键。服务端上则是从客户端传过来的的预测键。

```cpp
FScopedPredictionWindow ScopedPrediction(AbilitySystemComponent.Get());
```

创建预测窗口的目的是生成同步点，如果我们用了WaitDelay这种延迟回调的节点，且内部没有创建新的预测窗口，在服务器侧的预测键就会和客户端不一致，导致后续需要预测键的操作失效。
![[GIC-多人GAS#Prediction工作原理]]

在Aura中发送光标命中的目标时，SendMouseCursorData()中就创建了预测窗口。因为当服务器收到数据并调用ValidData.Broadcast(DataHandle);时已经和客户端的ValidData.Broadcast(DataHandle);不在同一个原子组内了。
另外获取激活的预测键的方式为：GetActivationPredictionKey()
获取当前的预测键方式为：AbilitySystemComponent->ScopedPredictionKey
![[Aura客户端向服务器发送TargetData#^295ae3]]