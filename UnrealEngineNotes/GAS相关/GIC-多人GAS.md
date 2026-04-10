油管GIC频道有关多人GAS的梳理干货满满
b站翻译：
https://www.bilibili.com/video/BV14RrbBYEB5/?spm_id_from=333.1387.favlist.content.click&vd_source=2d5f5b0c789f1187890235f6fa20045c
原视频：
https://www.youtube.com/watch?v=WyyUPqdZQfU

# Attribute
属性中当前值的计算公式，这里有两个属性：生命值和治疗点数（治疗药水的数量）
![[GAS相关/GIC-多人GAS（Media）/1.png]]

# GE
GE中的三种持续类型。分别在current value中充当的角色。
- Instant Effect是直接作用到BaseValue上的。
- Duration Effect和Infinite Effect都是作为GE Modifier的一部分和BaseValue一起构成CurrentValue。这两种ge可以添加和移除。移除后CurrentValue就会和BaseValue相等。
在执行一次治疗能力中我们可能会用到两个GE。治疗的值和治疗的消耗。
![[GAS相关/GIC-多人GAS（Media）/2.png]]

两个GE在编辑器中的样子，都使用Instant 类型。
![[GAS相关/GIC-多人GAS（Media）/3.png]]

# GA
负责封装所有技能逻辑，并把GE应用到属性上。
1. 先需要检查是否被block tag阻止激活该能力。
2. 再检查cost ge是否可以支付然后消耗它。
3. 跑AbilityTask播放蒙太奇。
4. 应用GE_Heal。
5. 执行Gameplay Cue。
![[GAS相关/GIC-多人GAS（Media）/4.png]]

# AT（AbilityTask）和 GC（Gameplay Cue）
通常游戏技能GA是原子操作，只在一帧内运行完。
如果你需要任何运行超过一帧的内容，就必须使用AbilityTask和游戏提示Gameplay Cue。Gameplay Cue是视听效果的容器，生成或者移除粒子发射器。下图是GA能做的所有功能。
![[GAS相关/GIC-多人GAS（Media）/5.png]]

下图是蓝图中实现生命回复的技能逻辑图。Commit节点会检查冷却和消耗是否满足。
![[GAS相关/GIC-多人GAS（Media）/6.png]]

# GAS中的额外步骤
我们可以在PreAttributeChange中给Health属性夹值。
可以在CanActivateAbility中判断能否激活能力，假如血量已经达到最大血量时应该不能激活能力。
![[GAS相关/GIC-多人GAS（Media）/7.png]]

# UE多人游戏架构
不使用GAS时多人网络通信，需要通过属性复制，RPC，和CMC。GAS中可以直接编写技能并处理网络复制。
![[GAS相关/GIC-多人GAS（Media）/8.png]]

## 不使用GAS的预测流程
**不使用GAS并且不预测** 的网络通信大概如下图所示。客户端检查先觉条件，把RPC发送给服务端。服务端再次检查先觉条件然后增加生命值，然后把生命值增加的信息发回给客户端。这套流程看似合理，但实际上没有预测是不可接受的。会产生严重的延迟。
![[GAS相关/GIC-多人GAS（Media）/9.png]]

**不使用GAS并且预测** 的网络通信大概如下图所示。利用ue中的的属性同步，如果客户端已经和服务器的值相等就不需要再同步了。
整体流程如下所示。
![[GAS相关/GIC-多人GAS（Media）/10.png]]

这看起来没问题但是如果服务器判定你不能治疗会发生什么？这涉及到很多问题。
我们需要等待这个确认吗？要等多久？
我们需要回滚这个操作吗？
如果玩家在治疗过程中受到伤害该怎么办？
如果玩家已在治疗过程中又尝试了一次此时服务器还在处理上次的治疗，客户端当前又进行了一次治疗。当前一次治疗实际上并未生效。
这种情况下会有很多复杂的问题你必须仔细考虑。而这还仅仅是治疗这一种情况。你本意只是想按个键喝药水回血而已。
![[GAS相关/GIC-多人GAS（Media）/11.png]]

## 使用GAS的预测流程
在GAS中机制是这样的
客户端：
1. 在客户端检查前置条件
2. 生成预测密钥
3. 它会添加分配了该预测密钥的GE
4. 在客户端增加生命值
5. 带着这个预测密钥PK向服务器发送RPC

服务端：
1. 检查前置条件
2. 应用这个带有预测密钥的治疗GE
3. 增加生命值，并将这个密钥发回给客户端

![[GAS相关/GIC-多人GAS（Media）/12.png]]

但是，如果服务器判定你不能治疗，即游戏效果生效失败。它会向客户端发送信息，告知该预测密钥已被拒绝。
于是客户端就知道必须在本地移除这个效果。它只是移除了之前的那些生命值。
![[UnrealEngineNotes/GAS相关/GIC-多人GAS（Media）/13.png]]

这套系统对于我们在示例中使用的所有内容都能完美运作。所有内容都开箱即用地支持客户端预测的网络同步。
- GA 激活
- 播放蒙太奇 （AT）
- 应用GE
- 生成GC
![[GAS相关/GIC-多人GAS（Media）/14.png]]

GAS的构建初衷就考虑到了Replication，专门为paragon开发的，并且也应用到了堡垒之夜上。

# GAS的正确初始化
使用GAS需要ASC和AttributeSet的协作，不要把这两者直接放在角色类中。应该放在PlayerState上，因为PlayerState是一个对所有客户端都可见的Actor，所以每个人都能知道彼此的状态。还有非常重要的一点如果你在定义属性集AttributeSet应该将其定义为瞬态（Transient）
![[GAS相关/GIC-多人GAS（Media）/15.png]]

初始化流程很简单，在构造函数中创建ASC和AttributeSet，注意设置复制模式为Mixed，提高网络更新频率（PS的网络更新频率是比较低的），属性集是在构造函数中创建的他是一个UObject不是一个Component，基本上在Actor的构造函数中创建这个对象是非常糟糕的主意，因为如果你决定复制一个蓝图，也就是继承自它的蓝图，由于它是在构造函数中创建的，它会被序列化（serialized），它会将所有发生的更改，甚至是在游戏运行期间的更改都复制到新生成的对象中。这会制造很多混乱。
这就是为什么属性集应该被设置为瞬态（Transient），被设置为瞬态的对象不会被序列化。
![[GAS相关/GIC-多人GAS（Media）/16.png]]

然后你必须运行InitAbilityActorInfo函数。
其参数是拥有者Owner，也就是我们创建该组件的对象PS。
化生Avator，也就是该组件所控制的Actor在这种情况下也就是玩家角色。
这是一个初始化检查的好地方，适用于任何玩家角色和玩家控制器相关的内容。不要在BeginPlay中这样做。
因为在玩家角色的BeginPlay阶段你无法保证PlayerController已经存在。
而在PlayerController的BeginPlay阶段也不能保证PlayerCharacter已经存在。并且你无法保证他们已经连接上了这里是指Character被PlayerController占有Possess。最糟糕的是当你在编辑器中测试游戏时很可能是正常的，但如果在Standalone（独立进程）下运行会怎么样？Standalone模式的初始化方式截然不同。
![[GAS相关/GIC-多人GAS（Media）/17.png]]

所以要把角色和控制器的初始化放在这里。这里是指PossessedBy函数，它应该在服务器上运行。
以及OnRep_PlayerState函数中，它应该在客户端中运行。
![[GAS相关/GIC-多人GAS（Media）/18.png]]

至于AI角色，因为AI角色也可以拥有技能系统。只需要在AI角色类上创建ASC和AttributeSet
![[GAS相关/GIC-多人GAS（Media）/19.png]]


同样在构造函数中创建，唯一的区别是把复制模式设置成Minimal，以获得最佳的网络带宽。
![[GAS相关/GIC-多人GAS（Media）/20.png]]

并且只需要在BeginPlay中初始化AbilityActorInfo。此时Owner和Avatar都是这个AICharacter。
![[GAS相关/GIC-多人GAS（Media）/21.png]]

当你在AttributeSet中定义属性时要将其设置为RepNotify_Always。如果你不这样设置，复制过来的值和当前值相同的话就不会触发通知。但是GAS需要这些信息，这样所有的预测功能才能正常工作。
![[GAS相关/GIC-多人GAS（Media）/22.png]]

# 激活技能
## GA的重要选项
**Replication Policy** 基本上不需要动它。演讲者认为这个名字具有误导性，认为只是遗留代码。Epic的人说过再未来可能会移除掉这个选项，因为GA已经会从服务端复制到主控客户端了。但是在实际的开发中我们可以在GA中使用自定义RPC事件的时候就需要这个选项。
![[GAS相关/GIC-多人GAS（Media）/23.png]]

**Instancing Policy** 当启动技能时技能实例就会被生成，被称为Gameplay Ability Spec。
- Instanced Per Execution：每次执行生成一个实例。
- Instanced Per Actor：只生成一个实例反复用，必要时要清理数据。
- Non-Instanced：不生成实例，使用CDO。
![[GAS相关/GIC-多人GAS（Media）/24.png]]