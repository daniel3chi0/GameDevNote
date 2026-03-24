Lyra项目中背包系统主要由三个模块构成分别是Inventory，Equipment，Weapons。
先看下C++侧源码目录的结构
Inventory：
![[开源项目分析/Lyra/Lyra背包系统/1.png|300]]
Equipment：
![[开源项目分析/Lyra/Lyra背包系统/2.png|300]]
Weapons：
![[开源项目分析/Lyra/Lyra背包系统/3.png|300]]

可以看到Lyra这个项目的是没有区分Public和Private目录的，因为Lyra是个游戏项目，通常不是作为插件提供给第三方使用的，要区分目录带来的收益很小，提高开发效率，缺点就是不利于维护和后续拆分成插件。

可以看到这三个模块都使用几个相似的类：Fragment，Definition，Instance，ManagerComponent，武器侧则是StateComponent比较特殊一点。