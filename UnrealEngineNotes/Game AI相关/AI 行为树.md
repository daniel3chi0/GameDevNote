ue4时期主要用行为树的方式写些ai逻辑。
相关的类主要有这些：AIController，BehaviorTree，BlackBoard。（AlController是AI的大脑，行为树是思维，黑板是记忆）当然还有每个大类下面需要关联的类。
当然还有感知的Actor组件UAIPerceptionComponent，UPawnSensingComponent(属于历史遗留Legacy，现在更推荐前者)
# AIController
作为AI大脑我们通常会在AIController中的BeginPlay中RunBehaviorTree来激活行为树。
同时监听AI的死亡事件，死亡的时候我们要用StopLogic停止掉BrainComponent上跑的行为树逻辑。
AI被占有On Possess和On Unposses的时候要调用BrainComponent上的StartLogic和StopLogic来激活和停止行为树。
AIController在构造的时候会自动创建PathFollowingComponent和UDEPRECATED_PawnActionsComponent（未来会废弃）
在Lyra中还会在On Possess的时候监听队伍是否切换，切换后会调用UAIPerceptionComponent::RequestStimuliListenerUpdate来重新感知

AIController能获得行为树和AI的角色类已经黑板组件，AIController的权限是很高的。
## LineOfSightTo
AController本身有LineOfSightTo这个方法用来判断另一个Actor是不是在视点范围内，AIController也重写了这个函数。
可以在服务中用来判断，然后更新黑板值。

# BehaviorTree
本质是一个持续运行的循环，当跑完所有分支上的任务的时候会开启下一次的循环，节点除了返回“成功”（Success）和“失败”（Failure）外，还有一个关键状态：“**运行**”（Running）。当一个节点（例如一个“移动到某点”或“等待”任务）需要持续多帧才能完成时，它会返回“运行”状态。在此状态下，行为树在下一帧会**直接继续执行该节点**，而不是回溯到父节点或树的更上层。从外部观察，执行流就像“阻塞”在了这个运行中的节点上，其父节点会等待它完成（返回成功或失败）后，才继续管理后续节点。

## 行为树节点
在Unreal Engine的AI模块中，行为树所有节点的最顶层基类是`UBTNode`。`UBTNode`定义了行为树节点的通用接口和生命周期，例如初始化、执行请求、终止等。从这个基类出发，衍生出几种不同类型的节点：
- `UBTCompositeNode`：复合节点（如Sequence、Selector）的基类，用于控制子节点的执行流程。
- `UBTAuxiliaryNode`：辅助节点的基类，其下又派生出`UBTDecorator`（装饰器）和`UBTService`（服务）。
- **`UBTTaskNode`：任务节点的基类，这是开发者实现具体AI行为（如移动、攻击、等待）时最常直接继承的类**。

因此，当我们在C++中创建一个自定义任务（例如`UBTTask_MyCustomTask`）时，标准的做法是让它继承自`UBTTaskNode`，并重写其`ExecuteTask`等方法。在蓝图中，对应的基类是`BTTask_BlueprintBase`，它同样构建在`UBTTaskNode`体系之上。这是行为树任务开发的主流和标准路径。

### UGameplayTask和UBTTask
部分节点会使用到**GameplayTask**（如BTTask_MoveTo）：`UGameplayTask`是一个更通用、更底层的异步任务框架，它并不专属于AI模块。官方文档指出，其应用场景至少包括**GAS（Gameplay Ability System）和AI**两个主要领域。`UGameplayTask`提供了任务激活、暂停、恢复、资源互斥、基于优先级的打断等核心机制。在非AI模块和非GAS中同样可以使用GameplayTask，如这篇文章：[[https://www.zhihu.com/search?type=content&q=gameplaytask]]

在AI模块中，`UGameplayTask`框架的具体体现是`UAITask`类。**`UAITask`直接继承自`UGameplayTask`**，并集成了AI相关的上下文（如`AAIController`）。然而，`UAITask`通常用于实现那些需要复杂异步管理、资源竞争或精确打断逻辑的**底层AI操作**，而不是暴露给行为树编辑器的通用任务节点。

理解`UBTTaskNode`和`UAITask`（及背后的`UGameplayTask`）是如何协作的。它们并非替代关系，而是**上层应用与底层支撑**的关系。
- **执行机制整合**：一个`UBTTaskNode`在其内部执行逻辑中，可以创建并管理一个或多个`UAITask`实例。例如，引擎内置的**BTTask_MoveTo**任务，其内部封装了一个`UAITask_MoveTo`来实际处理寻路移动的异步过程。`UBTTaskNode`的`ExecuteTask`函数负责启动这个`UAITask`，并监听其完成或中断事件，然后据此向行为树返回成功（`Succeeded`）、失败（`Failed`）或运行中（`InProgress`）的状态。
- **中断与资源管理**：`UGameplayTask`框架提供的基于优先级的打断和资源互斥机制，为行为树中装饰器（`Decorator`）的“观察者中止”（`ObserverAborts`）功能提供了底层支持。当行为树因条件变化需要中断一个正在`InProgress`的`UBTTaskNode`时，该中断请求会向下传递，最终可能导致其内部驱动的`UAITask`被`Pause`或终止，从而实现平滑的行为切换。
- **生命周期映射**：`UBTTaskNode`中需要重写的`ReceiveExecute`（开始）、`ReceiveTick`（持续）、`ReceiveAbort`（中断）等事件，与`UGameplayTask`的`Activate`、`TickTask`、`Pause`/`Resume`等生命周期函数形成了对应关系，共同管理着任务从开始到结束的完整过程。

对于开发者来说，应该遵守以下原则：
- **创建行为树中的可配置任务节点时，应继承`UBTTaskNode`（C++）或使用`BTTask_BlueprintBase`（蓝图）**。这是将自定义行为集成到行为树可视化编辑器中的标准且唯一的方式。您需要关注的是任务逻辑本身以及如何通过`FinishExecute`等函数向行为树返回结果。
- **当需要实现一个跨AI模块复用、或需要精细控制异步、资源与优先级的底层原子操作时，可以考虑继承`UAITask`或直接使用`UGameplayTask`**。但这通常属于更高级的模块扩展，例如为AI创建一套新的、可被多种任务节点复用的移动或动画系统。

### Composites
合成（Composites） 节点是流控制的一种形式，决定了与其相连的子分支的执行方式。
1. 选择器（Selector） 从左到右执行分支，通常用于在子树之间进行选择。当选择器找到能够成功执行的子树时，将停止在子树之间移动，并且节点本身返回succeed。举例而言，如果AI正在有效地追逐玩家，选择器将停留在那个分支中，直到它的执行结束，然后转到选择器的父合成节点，继续决策流此时对应返回succeed。
2. 序列（Sequence） 从左到右执行分支，通常用于按顺序执行一系列子项。与选择器节点不同，序列节点会持续执行其子项，直到它遇到失败的节点。举例而言，如果我们有一个序列节点移动到玩家，则会检查他们是否在射程内，然后旋转并攻击。如果检查玩家是否在射程内便已失败，则不会执行旋转和攻击动作，只要一个任务返回失败，这个Sequence就返回失败。
3. 简单平行（Simple Parallel） 简单平行节点有两个"连接"。第一个是主任务，它只能分配一个任务节点（意味着没有合成节点）。第二个连接（后台分支）是主任务仍在运行时应该执行的活动，这个并行的分支可以是复合节点或者是另一个简单平行。简单平行节点可能会在主任务完成后立即结束，或者等待后台分支的结束，具体依属性而定，这个节点的返回结果和并行任务没有任何关系，主任务返回succeed后这个节点就返回succeed。

### Service
当一个分支挂载Service的时候，当行为树在进入一个分支直到分支结束的时候，服务上会负责这个分支时间周期的Tick逻辑。可以用户设定这个Tick的时间周期。可以理解为行为树的眼睛，实时计算外部环境变化并更新AI的记忆——即黑板。可以通过定义FBlackboardKeySelector类型的变量获取黑板值。

### Decorator
装饰器通常附加在行为树中的某个节点上。充当条件判断这个节点能否继续执行，或是打断正在执行的任务。
#### Flow Control（观察打断个人理解）
![[Game AI相关/AI 行为树/1.png|600]]
当装饰器上级的 **Composites** 节点是 Selector，则 **ObserverAborts** 有4个可选值：
- None
- Self
- LowerPriority
- Both
 当装饰器上级的 **Composites** 节点是 **Sequence**，则 ObserverAborts 有2个可选值：
- None
- Self

**None**：默认值选了None的装饰器无作用。

**Self**：（可以立刻停止自身的）若装饰器的计算结果为True，则执行修饰的任务，如果装饰器修饰Composites节点，则允许继续查找下级节点。如果装饰器结果是false，则不执行修饰的任务，如果这个任务修饰正在持续执行的任务，当装饰器计算结果由True改变为False时，当前任务会停止；如果装饰器修饰Composites节点，则不会继续查找下级节点。

**LowerPriority**：（不立刻停止自身，可以立刻停止低优先级的节点旁的数字越小代表的优先级越高）若当前装饰器计算结果为False，则不能执行当前修饰的节点，要执行比这个优先级低的节点。若装饰器计算结果True，会执行当前节点，并且会停止比当前节点低优先级的各种节点。进一步理解，当有一个任务是持续任务正在执行，修饰这个任务的 LowerPriority 装饰器，从True变成False，则不会停止持续任务，因为不是Self，但是如果这个任务结束了，下次再想执行，就不能执行了，因为装饰器是False。

**Both**：（既可以立刻停止自身的，又可以立刻停止低优先级的）单词意思是两者，哪两者？Self和Lower Priority。当这种装饰器为True，则可以执行当前节点，停止低优先级的节点。如果这装饰器为False，则停止当前修饰的节点，执行低优先级的节点。

NotifyObserver：
- On Result Change：装饰器自身的结果发生变动，就重新评估是否打断
- On Value Change：装饰器修饰的值变动，重新评估是否打断。

节点被中断后会从这个节点的父节点开始重新进行评估。

使用Blackboard装饰器可以检查黑板上的值加以判断。is set对应true，is not set对应false，如果是对象类型则对应是否为空。

我们能加的装饰器的种类很多，并且可以在一个节点上叠加：自定义黑板装饰器，冷却装饰器，循环装饰器。
这种情况不对loop第一次后会进入冷却
![[Game AI相关/AI 行为树/2.png]]

应该改成这种
![[Game AI相关/AI 行为树/3.png]]
# BTTask
继承UBTTask_BlackboardBase的子类task节点可以指定一个任意类型的黑板值作为执行这个行为所需的参数如UBTTask_MoveTo节点支持两种类型的参数FVector和AActor。可以通过定义FBlackboardKeySelector类型的变量获取黑板值。在C++侧我们通常继承UBTTaskNode节点。

# EQS
环境查询系统使用前需要在插件中开启。

## Generator
比如我们可以创建一个环形生成器
![[Game AI相关/AI 行为树/4.png]]

## Test
在一个generator上我们可以添加tests，test也有多种类型的，如一定距离或是在一个体积的内部或是外部。
![[Game AI相关/AI 行为树/5.png]]
添加一个距离测试这是默认与载体的距离。载体就是调用这个EnvironmentQuery文件的对象。
![[Game AI相关/AI 行为树/9.png]]

如在AI的行为树中调用这个文件，载体就是那个AI，行为树中通过RunEQSQuery来返回EQS数据。query template选上我们创建的environment query文件。下面的黑板key是我们通过这个节点通过EQS计算出的结果最后要填充的黑板值。
![[Game AI相关/AI 行为树/6.png]]

计算后的黑板值拱其他节点使用
![[Game AI相关/AI 行为树/8.png]]

RunEQSQuery任务节点的运行模式有下面这些
![[Game AI相关/AI 行为树/7.png]]

在debug中，绿色的球是通过测试的点，如果我们用single best item就是取分数最高的点。
![[Game AI相关/AI 行为树/10.png]]

## 环境查询上下文
默认查询者是这个AI自身，我们可以自定义查询者
![[Game AI相关/AI 行为树/11.png]]
选择EnvQueryContext_BlueprintBase作为基类，在蓝图中制作这个context。

使用provide single Actor
编写一下获取玩家对象的逻辑。EQS在功能层面上已经完善了，但是在显示层面上可能并不完善。这里的querier object和querier actor其实是一个东西都是AI，即调用环境查询的对象（BT）的本身的actor(AI)。
![[Game AI相关/AI 行为树/12.png]]

指定给EQS中的的generator
![[Game AI相关/AI 行为树/13.png]]

测试中还有评分和筛选机制
![[Game AI相关/AI 行为树/14.png]]

# 感知组件
旧的感知组件Pawn Sensing 应该被AI Perception所取代，但是现在它们两者都还存在于虚幻引擎里，还都能正常工作。
Lyra中使用AI Perception，但是只有一处被简单的使用了。SensesConfig是注册的感知配置，DominantSense是多个感知同时感知到目标后右边使用哪个感知。蓝图中只调用了RequestStimuliListenerUpdate这个函数。在队伍变换时手动强制AI感知系统更新指定目标刺激监听器的属性。但是Lyra中好像没有什么后续操作了。

![[Game AI相关/AI 行为树/15.png]]