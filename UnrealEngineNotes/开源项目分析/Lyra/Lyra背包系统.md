Lyra项目中背包系统主要由三个模块构成分别是Inventory，Equipment，Weapons。
先看下C++侧源码目录的结构
Inventory：
![[开源项目分析/Lyra/Lyra背包系统/1.png|300]]
Equipment：
![[开源项目分析/Lyra/Lyra背包系统/2.png|300]]
Weapons：
![[开源项目分析/Lyra/Lyra背包系统/3.png|300]]

可以看到Lyra这个项目的是没有区分Public和Private目录的，因为Lyra是个游戏项目，通常不是作为插件提供给第三方使用的，要区分目录带来的收益很小，提高开发效率，缺点就是不利于维护和后续拆分成插件。

可以看到这三个模块都使用几个相似的类：Fragment，Definition，Instance，Entry，ManagerComponent，武器侧则是StateComponent比较特殊一点。

# Fragment
Fragment是片段的意思，是UObject的子类，被Definition所持有。
# Definition
Definition是定义的意思，是UObject的子类，被Instance所持有，作为Instance中的数据填充。
# Instance
Instance是实例，是UObject的子类，Instance通常是**Entry**中的一部分，并且**Entry**被EntryList所包裹且两者组合成FastArray的结构，然后EntryList被ManagerComponent所持有，Instance代表元素条目中的实体。
# ManagerComponent
作为管理器管理EntryList这个列表：
- 在Inventory部分中作为ActorComponent的子类。
- 在Equipment部分中作为ModularGameplay的PawnComponent的子类。
- 在Weapons部分中作为ModularGameplay的ControllerComponent的子类（Controller组件顾名思义就是要挂在controller上的）。Weapons部分比较特殊，内部没FastArray结构的EntryList，取而代之的是两个结构体数组：屏幕空间的打击点数组，服务器侧打击点标记合批数组。

# 一次实例生成过程
在Inventory部分有个接口Pickupable，里面有个纯虚函数GetPickupInventory。
![[开源项目分析/Lyra/Lyra背包系统/4.png]]

## ALyraWorldCollectable
这个接口被ALyraWorldCollectable这个Actor所继承实现如下:        内部结构如——>[[#FInventoryPickup StaticInventory]]
```cpp
FInventoryPickup ALyraWorldCollectable::GetPickupInventory() const  
{  
    return StaticInventory;  
}
```

同时这个Actor也继承了IInteractableTarget这个接口，这个类是世界上的可收集物，它可以被拾取可以被交互。其中有两个结构体变量：
### FInteractionOption Option
其中的主要是交互提示的，UI，Text，该交互物的ASC，以及交互时的表现能力，
其中的TScriptInterface\<IInteractableTarget\> InteractableTarget是在AbilityTask中~~由FInteractionOptionBuilder来传入的~~InteractableTarget字面意思是可交互的目标，可这是个收集物，它可交互的目标？指玩家还是它自己呢？（看下下面的调用堆栈知道是ALyraWorldCollectable这个对象）
看一下涉及到的AbilityTask有：
1. UAbilityTask_WaitForInteractableTargets其中的InteractableTarget是由UAbilityTask_WaitForInteractableTargets_SingleLineTrace中开了个定时器做射线检测传入的。
	 ```cpp
     UAbilityTask_WaitForInteractableTargets_SingleLineTrace
     {
	     SetTimer();
		     TArray<TScriptInterface<IInteractableTarget>> InteractableTargets;  
			 UInteractionStatics::AppendInteractableTargetsFromHitResult(OutHitResult, InteractableTargets);
		     PerformTrace(InteractionQuery, InteractableTargets);
			 Base::UpdateInteractableOptions(InteractionQuery, InteractableTargets);
				 TArray<FInteractionOption> TempOptions;  
				 FInteractionOptionBuilder InteractionBuilder(InteractiveTarget, TempOptions);  
				 InteractiveTarget->GatherInteractionOptions(InteractQuery, InteractionBuilder);    //这里的GatherInteractionOptions是ALyraWorldCollectable中的函数
					 InteractionBuilder.AddInteractionOption(Option);
	 }
     ```
  
2. UAbilityTask_GrantNearbyInteraction中的InteractableTarget中也是大致的流程

回到正题理了一遍代码发现这个Option不是在代码中赋值的而是在编辑器中赋值的，InteractableTarget主要用来在交互的GA中最为指示器的目标这个目标是ALyraWorldCollectable。或是作为GameplayEvent负载数据传入Option上的ASC激活交互能力。
### FInventoryPickup StaticInventory
看下结构体内部有两个数组：实例和模板
```cpp
UPROPERTY(EditAnywhere, BlueprintReadOnly)  
TArray<FPickupInstance> Instances;  
  
UPROPERTY(EditAnywhere, BlueprintReadOnly)  
TArray<FPickupTemplate> Templates;
```

实例中是Item实例，模板中是definition数据和Stack数
```cpp
USTRUCT(BlueprintType)  
struct FPickupTemplate  
{  
    GENERATED_BODY()  
  
public:  
    UPROPERTY(EditAnywhere)  
    int32 StackCount = 1;  
  
    UPROPERTY(EditAnywhere)  
    TSubclassOf<ULyraInventoryItemDefinition> ItemDef;  
};  
  
USTRUCT(BlueprintType)  
struct FPickupInstance  
{  
    GENERATED_BODY()  
  
public:  
    UPROPERTY(EditAnywhere, BlueprintReadOnly)  
    TObjectPtr<ULyraInventoryItemInstance> Item = nullptr;  
};
```

### 通过GetPickupInventory添加到库存中
当我们把一个可拾取物加入库存中时会获取拾取物上的FInventoryPickup这个结构其实是个数据对象，里面存默认实例和definition。用来生成最终的物品实例。
把可拾取物加入背包有两部操作：加定义和加实例。
```cpp
void UPickupableStatics::AddPickupToInventory(ULyraInventoryManagerComponent* InventoryComponent, TScriptInterface<IPickupable> Pickup)  
{  
    if (InventoryComponent && Pickup)  
    {       
	   const FInventoryPickup& PickupInventory = Pickup->GetPickupInventory();  
       for (const FPickupTemplate& Template : PickupInventory.Templates)  
       {          
	       InventoryComponent->AddItemDefinition(Template.ItemDef, Template.StackCount);  
       }  
       for (const FPickupInstance& Instance : PickupInventory.Instances)  
       {          
	       InventoryComponent->AddItemInstance(Instance.Item);  
       }    
    }
}
```

我们看下两个AddItem的具体实现
```cpp
ULyraInventoryItemInstance* ULyraInventoryManagerComponent::AddItemDefinition(TSubclassOf<ULyraInventoryItemDefinition> ItemDef, int32 StackCount)  
{  
    ULyraInventoryItemInstance* Result = nullptr;  
    if (ItemDef != nullptr)  
    {       
	   Result = InventoryList.AddEntry(ItemDef, StackCount);  
       if (IsUsingRegisteredSubObjectList() && IsReadyForReplication() && Result)  
       {          
	       AddReplicatedSubObject(Result);  
       }    
    }    
    return Result;  
}  
  
void ULyraInventoryManagerComponent::AddItemInstance(ULyraInventoryItemInstance* ItemInstance)  
{  
    InventoryList.AddEntry(ItemInstance);  
    if (IsUsingRegisteredSubObjectList() && IsReadyForReplication() && ItemInstance)  
    {       
	    AddReplicatedSubObject(ItemInstance);  
    }
}
```