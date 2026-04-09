油管GIC频道有关多人GAS的梳理干货满满
b站翻译：
https://www.bilibili.com/video/BV14RrbBYEB5/?spm_id_from=333.1387.favlist.content.click&vd_source=2d5f5b0c789f1187890235f6fa20045c
原视频：
https://www.youtube.com/watch?v=WyyUPqdZQfU

# Attribute
属性中当前值的计算公式，这里有两个属性：生命值和治疗点数（治疗药水的数量）
![[UnrealEngineNotes/GAS相关/GIC-多人GAS（Media）/1.png]]

# GE
GE中的三种持续类型。分别在current value中充当的角色。
- Instant Effect是直接作用到BaseValue上的。
- Duration Effect和Infinite Effect都是作为GE Modifier的一部分和BaseValue一起构成CurrentValue。这两种ge可以添加和移除。移除后CurrentValue就会和BaseValue相等。
在执行一次治疗能力中我们可能会用到两个GE。治疗的值和治疗的消耗。
![[UnrealEngineNotes/GAS相关/GIC-多人GAS（Media）/2.png]]

两个GE在编辑器中的样子，都使用Instant 类型。
![[UnrealEngineNotes/GAS相关/GIC-多人GAS（Media）/3.png]]

# GA
负责封装所有技能逻辑，并把GE应用到属性上。
1. 先需要检查是否被block tag阻止激活该能力。
2. 再检查cost ge是否可以支付然后消耗它。
3. 跑AbilityTask播放蒙太奇。
4. 应用GE_Heal。
5. 执行Gameplay Cue。
![[UnrealEngineNotes/GAS相关/GIC-多人GAS（Media）/4.png]]

# AT（AbilityTask）和 GC（Gameplay Cue）
通常游戏技能GA是原子操作，只在一帧内运行完。
如果你需要任何运行超过一帧的内容，就必须使用AbilityTask和游戏提示Gameplay Cue。Gameplay Cue是视听效果的容器，生成或者移除粒子发射器。下图是GA能做的所有功能。
![[UnrealEngineNotes/GAS相关/GIC-多人GAS（Media）/5.png]]

下图是蓝图中实现生命回复的技能逻辑图。Commit节点会检查冷却和消耗是否满足。
![[UnrealEngineNotes/GAS相关/GIC-多人GAS（Media）/6.png]]

# GAS中的额外步骤
我们可以在PreAttributeChange中给Health属性夹值。
可以在CanActivateAbility中判断能否激活能力，假如血量已经达到最大血量时应该不能激活能力。
![[UnrealEngineNotes/GAS相关/GIC-多人GAS（Media）/7.png]]

# UE多人游戏架构
不使用GAS时多人网络通信，需要通过属性复制，RPC，和CMC。GAS中可以直接编写技能并处理网络复制。
![[UnrealEngineNotes/GAS相关/GIC-多人GAS（Media）/8.png]]
