# 什么是状态树？
比起状态机，状态树更简洁直观，并且可以在不同的叶子节点的状态间跳转，这在状态机的状态中要新加过渡规则。
比起行为树也是少了composite节点的分流逻辑，更直观。同样多了控制不同叶子状态间的跳转，排版也更直观。
![[Game AI相关/状态树 StateTree Media/1.png]]

# 为什么要开发状态树？
在ue5中需要足够的通用的轻量的一个工具或者框架去表示系统的逻辑状态。
behavior tree和状态机只能用在Pawn或Character上的AI Controller中。
ue5中有许多功能需要通用的状态机来支持。
比如mass ai里每一个mass entity 不再是一个actor，所以没办法直接套用behavior tree那套东西。
比如city sample 中的smart object和AI的交互不能很方便的使用behavior tree交互等。

如果在通用蓝图模版中实现状态机可行吗？
- 蓝图虽然很强大但是很快就会复杂到你无法去管理。所有状态间的切换可能需要我们手动去写。布线的问题可读性会影响。

使用行为树的模版怎么样？
- 行为树的界面相对于蓝图或者是动画状态机来说，不管是界面的整洁还是可读性都相对好点。
- 行为树的过渡表现在这个ui界面中不是很明显的表现出来的。你必须理解composite节点的运行逻辑以及装饰器的打断逻辑。
- 最重要一点还是BT只能用在AI Controller上

使用动画状态机模版怎么样？
- 动画状态机的动画之间的相互切换其实是非常清楚的，但是因为非常清楚导致过渡线非常的凌乱。
- 还有一个问题就是他的过渡规则和表现逻辑你必须要点到他的下一层界面中去。
- 另外现在ue中的动画状态机写的是非常specific动画系统的。直接porting过去是有一定的工作量的。

**状态树就解决了上面很多问题**
- 界面简洁明了，没有很多过渡线。通常情况不需要进入到下层界面（比如进入到他的task中看有什么逻辑），只需要类似BT一样在同一个界面中就能完成大部分操作。

# 介绍一下状态树的核心概念
![[Game AI相关/状态树 StateTree Media/2.png]]

# States
主面板中的每一个蓝色方块代表一个state，点击状态后有细节面板中会有Enter /Transition Condition，Task， Transition


Schema，Evaluator，Parameter是在左侧面板表示全局的，5.0文档中Evaluator是关联于每一个node的，也就是每一个状态，5.0后官方改成全局了。
state tree是一个通用的状态机框架，可以替代BT，做mass ai的编写，做smart object的ai行为。
# Schema
我们看一下这个schema的行为
这里我们选中了MassBehavior这个schema，这样他只会把和mass ai相关的task和condition显示出来。
![[Game AI相关/状态树 StateTree Media/3.png]]

schema下面还有一个context代表在不同的case下有不同的支持外部参数的导入比如一些worldsubsystem，actor, handle等。比如你选了一个state tree component的一个schema的话，它的context就可能是他的哪个controller是拥有这个component，或者是哪个pawn现在正被运行着这个StateTree AI。不同的应用场景会被绑定不同的上下文相关的一些外部参数。
![[Game AI相关/状态树 StateTree Media/4.png]]

# Evaluator
evaluator就是在每个task之前去执行这个evaluator的逻辑。

# Parameter
parameter就是你可以通过这个参数去暴露给其他的UI，比如说暴露给StateTreeComponent，用户可以在StateTreeComponent中调整一些参数，去影响到整个statetree在执行时候的各种逻辑行为。

# 数据传递
接下来我们来看一下参数是如何传递的。
用户自己定义的参数parameter可以暴露给外部的component或者是外部引用他的owner界面上。你可以去调整这些参数。调整 statetree在运行时的一些行为。
还有context可以绑定运行这个StateTree的Controller,或者运行这个StateTree的Character或者Pawn,或者通过类型自动获得某一个你要的worldSubSystem的指针。
![[Game AI相关/状态树 StateTree Media/5.png]]

而每一个Task还可以使用它的ParentTask，就是他所有的前置运行的Task output它也可以去使用的。evaluators是优先执行的。这里的got hit就是它的output，这个output可以作为后续task的transition的input进行绑定。

下面的Annotation Tags也是同理。
![[Game AI相关/状态树 StateTree Media/6.png]]

在task中也是把一个task中output作为它的下级task的input，每个参数默认的情况下是通过copy的方式传到它下一级的task中。避免很多情况下的误操作导致的，像原来的blackboard的时期一些误操作把一些状态的值给修改掉，导致逻辑出现错误。而且每个task只会去引用它所需要的相关的参数。使得这个task不会像原来blackboard一样它所有的相关的行为树所需要的变量统统塞在一个blackboard中，这个blackboard会变 的非常庞大。参数在task直接传递默认情况下是通过copy的方式，如果你的参数是一个大的object或者是结构体是允许通过引用或是指针方式传递的。

在整个StateTreeComponent运行的初始化的阶段，就会调用当前所有加在这个StateTreeComponent上的Global Evaluator那个Section下面的所有Evalutor 的TreeStart函数。

你再去实现自己的Evaluator的时候，你会重载这些函数。接着会调用Tick函数确保output的值是合法的。

接着去调用SelectState函数,初始化的时候并不知它最终会落在哪个状态上。所以他传进去的target State会是当前的根状态root。这里有一个很重要的概念，在statetree中只有叶子节点才能作为最终的一个激活状态。

所以你传进去的root最终不能作为一个稳定的激活状态，这时候StateTree会去进行深度优先遍历他下面的子节点。然后根据子节点提供的enter condition，如果enter condition 能过能够return true它就会掉到那个子节点的下一个节点直到找到叶子节点。

# 运行逻辑
![[Game AI相关/状态树 StateTree Media/7.png]]

比如reacting这个state它前面没有问号所以它的enter condition的输出是true，然后判断flee的enter condition
![[Game AI相关/状态树 StateTree Media/8.png]]

假如flee，handleobstacle，hit，的enter condition都不满足
会进入平级的后面一个节点中判断。
![[Game AI相关/状态树 StateTree Media/9.png]]

比如进入这个状态后，先是claim这个smart object，然后到下一个状态，执行三个task：target，move和look。就是移动到这个smart object然后转向这个object然后reach这个state会做为它的最终的Active State。
![[Game AI相关/状态树 StateTree Media/10.png]]

从root节点到激活的叶子节点，这条链路都属于激活状态。
激活状态意味着在后面的每帧运行时，所有激活状态中的task的tick函数都会被调用。
evaluator的tick首先会被调用，以保证后面的task可能会引用到evaluator输出的output。所以现在的evaluator一定是每帧首先被tick掉。
之后就是从根节点到激活的叶子节点调用每个state里的task的tick，task是按顺序从左到右调用。
如果在task成功或者失败就会停止后续task的tick，这个时候会从叶子节点到根节点逆序去调用每个state中task的StateComplete函数。
![[Game AI相关/状态树 StateTree Media/11.png]]

比如在这个state中第一个task执行失败了，后面两个task虽然没有tick到，但是会从最后一个task开始逆序的调用state的complete函数然后到上一层的task
通知这些task说这个state已经结束了。
![[Game AI相关/状态树 StateTree Media/12.png]]

紧接着是做一个过渡的操作
![[Game AI相关/状态树 StateTree Media/13.png]]
虽然逆序调用task的ExitState和上面提到的task的StateComplete类似
但是调用ExitState的时候StateComplete不一定会被调用。
因为过渡的条件不一定依赖状态完成StateComplete。比如说你有一个ticked transition 如果满足里面的条件后它就会跳到另一个状态，而不是因为task。
调用成功后会把transition 中指定的next state设定成下一个SelectState(transition target state)。

在这段逻辑中
reach状态尝试过渡到下一个状态失败后会跳回Idling状态，但是Idling又不是叶子节点，所以从Idling节点开始一路判断后面的状态的Enter Conditon找可激活的叶子节点。然后进行下一帧的逻辑。
![[Game AI相关/状态树 StateTree Media/14.png]]

# Lyra行为树转状态树
首先进来selector会执行左边的结点的装饰器的判断，是否有子弹，通过的话下面又是套了一个selector的聚合节点，然后本身跑一个服务。
然后这个服务上还跑一个eqs来持续检测敌人。
![[Game AI相关/状态树 StateTree Media/15.png]]

接下来是如果找到了敌人，并且系统设置是可以开火，然后下面是接着跑两个服务，射击和设置瞄准
然后下面是两个task eqs找到位置，然后move to这个位置。
![[Game AI相关/状态树 StateTree Media/16.png]]

如果有子弹但是敌人没被找到的情况下走这条
逻辑就是寻找任务的装饰器判断敌人目标 应该是要没被设置的。
然后会一直跑清除瞄准和重载武器的服务，然后是跑搜索武器和玩家的两个分支任务2选1
![[Game AI相关/状态树 StateTree Media/17.png]]

然后现在是一直在AI巡逻的节点下面跑，所以巡逻的服务的逻辑会一直在跑
![[Game AI相关/状态树 StateTree Media/18.png]]

当这个服务发现敌人的时候
会直接跳到前面的is enemy find的这条链路
如果一直没有发现敌人就会进入WaitNoAction这个任务进行等待。
![[Game AI相关/状态树 StateTree Media/19.png]]

看一下另一边的逻辑
如果是out of Ammo就清除目标。然后就一直检查ammo，执行搜索武器的eqs然后move to
![[Game AI相关/状态树 StateTree Media/20.png]]
玩过lyra 的demo都知道AI很强，原因是在task中开了天眼。
这个search for weapon是一定能找到离AI最近的武器，search for enmy是不管自己的视线是否被挡住的。
![[Game AI相关/状态树 StateTree Media/21.png]]

我们现在要把行为树转成一个状态树
![[Game AI相关/状态树 StateTree Media/22.png]]

![[Game AI相关/状态树 StateTree Media/23.png]]

开启插件
![[Game AI相关/状态树 StateTree Media/24.png]]

创建状态树资产
![[Game AI相关/状态树 StateTree Media/25.png]]

选择schema
StateTreeComponent其实是继承BrainComponent的和行为树组件一样
![[Game AI相关/状态树 StateTree Media/26.png]]

给状态树资产新增一个状态
![[Game AI相关/状态树 StateTree Media/27.png]]

在state tree component 的schema下有三个默认的task可以让我们选择
![[Game AI相关/状态树 StateTree Media/28.png]]

debug text task这个就是在你的component上面显示一个字告诉你现在的状态
![[Game AI相关/状态树 StateTree Media/29.png]]

delay task就是模拟一个task的完成还是不完成
run forever的话这个task就一直在running，它不会在task的tick中返回它是success还是failure
把run forever去掉可以加一个duration表示几秒后success这个task
![[Game AI相关/状态树 StateTree Media/30.png]]

添加一个transition当任意一个task完成时（succeed和failed的结合）
就过渡到root中。gate delay就是满足过渡条件时你是否需要一个人为的delay操作。
![[Game AI相关/状态树 StateTree Media/31.png]]

看到这里我们不确定这个State完成的trigger是在什么时候触发（我觉得有可能是等所有task都完成后），上面说了是在当前状态的一个task成功或者失败。所以我进入代码中查看一些。发现确实如上所述。
![[Game AI相关/状态树 StateTree Media/32.png]]

我们打开bot的AIController在上面加一个StateTreeComponent
如果还是想通过blackboard的方式进行数据存储当然也是可以的。
![[Game AI相关/状态树 StateTree Media/33.png]]

在组件中把asset赋值进去就行了
![[Game AI相关/状态树 StateTree Media/34.png]]

状态树组件中也是赋值进asset然后勾上自动开始逻辑
![[Game AI相关/状态树 StateTree Media/35.png]]

游戏中
![[Game AI相关/状态树 StateTree Media/36.png]]

在我们之前的行为树中
我们其实有一个变量IsOutofAmmo 是在下面一直再被服务check的
像这样的变量就比较适合作为一个global的evaluator，来让它每帧不停的去检查当前有没有子弹。
![[Game AI相关/状态树 StateTree Media/37.png]]

这时候我们就会去写一个global的一个evaluator的蓝图
![[Game AI相关/状态树 StateTree Media/38.png]]

这个蓝图中就是
不停的去update这个ammo的值
这个update has ammo的方法其实就是从行为树中相应的节点中copy过来就行了。
![[Game AI相关/状态树 StateTree Media/39.png]]

global evaluator中我们主要重载这3个函数就行了：TreeStart，TreeStop，Tick
在这evaluator中的（context）是一个actor变量，output是一个bool变量和Pawn。
在状态树中指定一个evaluator
![[Game AI相关/状态树 StateTree Media/40.png]]

假设我们创建一个变量
![[Game AI相关/状态树 StateTree Media/41.png]]

它的categories是input，在状态树下会有一个in的后缀。
![[Game AI相关/状态树 StateTree Media/42.png]]

output变量后面没有绑定的选项，因为output是由evaluator计算给出的，然后可以作为后续的task的input，而input和context是有绑定的选项的。如果不指定变量是output还是input还是context，它会有一个复选框让我们直接赋值这个变量或者从外部绑定传进去。但是不会作为一个output
![[Game AI相关/状态树 StateTree Media/43.png]]

context就是一个特殊的input
如果后面你没有去绑定一个东西，他会从默认的从这个上面的context中绑定过来
对于StateTreeComponent来说，如果你的context actor class 是actor的话它会把AIController传给你
然后下面的evaluator中的context就是自动绑定成了ai controller
如果上面的context actor class被改成一个pawn的话，它就是传进来这个ai controller绑定的那个pawn
![[Game AI相关/状态树 StateTree Media/44.png]]

在上面的情况就是默认context传进来一个AI Controller，然后再evaluator中拿到pawn然后赋值output的那个pawn
![[Game AI相关/状态树 StateTree Media/45.png]]

下面讲一下这个condition
默认其实提供了非常多的condition的种类
![[Game AI相关/状态树 StateTree Media/46.png]]

比如我们使用一个bool比较的conditon
他可以绑定global Evaluator上的变量，或者是context。
![[Game AI相关/状态树 StateTree Media/49.png]]

当满足条件就可以进入状态。
![[Game AI相关/状态树 StateTree Media/47.png]]

enter condition 也可以通过蓝图自定义
继承基类然后实现重载函数即可，这个函数中的逻辑其实就是把原来装饰器中的判断逻辑copy过来实现的。
![[Game AI相关/状态树 StateTree Media/48.png]]

因为我们没有设置任何参数，所以不需要绑定任何东西。
如果添加多个条件我们可以选择条件之间是or还是and的关系。
![[Game AI相关/状态树 StateTree Media/50.png]]

接着是task有些task只需要执行一次如搜寻物品，有一些task需要像service一样一直重复的如ai的巡逻。
![[65.png]]
一次性的task举个例子比如eqs find weapon。需要重载的函数有enter state和tick
![[Game AI相关/状态树 StateTree Media/51.png]]

另外两个重载函数是
退出和完成，之前已经介绍过了complete是当前状态中只要一个task成功或者失败就调用每一个task的complete（并且能知道到底是哪个task的成功或失败触发的），exit则是在transiton之前会去调用每个激活状态task的exitstate函数。
![[52.png]]
同样exitState函数中也会告诉我们一些Task的信息
类似下面这个结构体，比如我们这个transiton的目标是什么，或者是next active state。
![[Game AI相关/状态树 StateTree Media/53.png]]

我们来看一下最下面的changeType 的含义是什么
在Search Addition Weapon有一个transition是失败的到search enemy的
这里两个状态有一个共同点就是他们都有相同的父状态。
![[Game AI相关/状态树 StateTree Media/54.png]]

在这样的transition的情况下，他们的父节点收到的enter state和exit state的change type就是
sustained（持续的）而不是change
![[Game AI相关/状态树 StateTree Media/55.png]]

而Search Addition Weapon里面的task收到exit state的change type是chaged的。
因为它在下一个active的状态下是不存在的。这样做我们可以在change type是change的时候我们可以清理掉很多cache，sustained的时候保持不变。然后就是在task的tick中我们调用eqs的逻辑。
![[Game AI相关/状态树 StateTree Media/56.png]]

跑eqs后把最终的结果填入movegoal，并且会更改querystatus，状态正确的时候说明eqs跑完了。
![[Game AI相关/状态树 StateTree Media/57.png]]

最后把输出的参数作为下面的task的输入
![[Game AI相关/状态树 StateTree Media/58.png]]

这就是一次性的task的情况
接下来是重复性的task
在state tree中传入public 的参数。
![[Game AI相关/状态树 StateTree Media/59.png]]

这里面的逻辑其实就很简单
在tick的时候一直设置参数。然后每隔一段时间去trigger一个service
![[Game AI相关/状态树 StateTree Media/60.png]]

这个run service其实是留给它的子类去触发的。子类需要重载
这个service其实干的事情也是run 一个query
![[Game AI相关/状态树 StateTree Media/61.png]]

内部还是跑一个eqs
![[Game AI相关/状态树 StateTree Media/62.png]]

同样的类似的它会跟上面的find position一样会有一个query finished 的状态，他有一个isenmyfound的bool变量来代替这个query finished状态。
我们放在这个状态不停的运行根据这个参数设置的间隔
![[63.png]]

这个其实也就是对应了behavior tree中的这个节点
![[64.png]]

这个就是一个可重复性的task

接下来我们用C++去拓展state tree
![[66.png]]

比如这个move to 的task不是用BP来实现的而是C++实现的
![[67.png]]

做法就是在到原来行为树中的moveto 的task
BT—task中也有per instance的data，这个东西是每个bot都会有的一份data。
![[68.png]]

在state tree下面设定的这些property
其实就不是per instance的property，是所有的bot都会受同样的这一组参数的影响。
![[69.png]]

像这样一些
task的finish和abort的事件的东西可以被putting到state tree task 的tick中enter state中的参数中去
![[70.png]]

如果需要参考案例我们可以在city sample那个案例中到到一些不错的或者源码的mass ai 模块中也有。
我们把所有的需要的东西准备好后

![[71.png]]

在做这个state上的task时其实他的父节点的也是一直处于激活状态
上面的几点注意都会tick的
![[72.png]]

在这个节点的task中当成功找到位置时我们会一直让他处于一个很running状态，而不会告诉他我们已经结束了
所以才能保证下面的engage enemy可以继续的move to
![[73.png]]

transition一般都会放在叶子节点中去做。
下面这个触发时机时在on tick的时候。
![[74.png]]

简化
![[75.png]]

一些优化的东西
每个state中有一个type，可以指定成subtree，group的意思时说这个节点时不能有任何的task的。
![[76.png]]

link和subtree实际可以联合在一起用的。
![[77.png]]

subtree主要来做state的复用
![[78.png]]

如果使用black board的话 可以在evaluator中拿到
![[79.png]]

因为能拿到aicontroller
![[80.png]]