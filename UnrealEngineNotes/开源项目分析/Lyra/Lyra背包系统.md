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

这个接口被ALyraWorldCollectable这个Actor所继承
