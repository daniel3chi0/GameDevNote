AnimationLayer （动画层）也是一种 AnimGraph，一种“Sub-AnimGraph”，动画蓝图中AnimGraph中的功能它都能实现。其本质上是一种函数，只不过函数处理或输出的数据是动画数据。

# 动画层的类型：
1. 在当前的动画蓝图类中直接创建。这种AnimationLayer具体叫 **AnimLayer**。AnimLayer的函数实现方式直接在当前的动画蓝图类中实现。
2. 借由继承Anim Layer Interface[[动画层接口]]后获得。这种AnimationLayer具体叫 **LinkedAnimLayer**。LinkedAnimLayer既可以在当前的动画蓝图类中实现，也可以通过指定另一个动画蓝图类，让这个动画蓝图类中的同名函数来实现。这种指定是可以Runtime执行的，非常灵活，也是Lyra中实现多套动画解耦的方法。 ^8b5785

# 动画层接口中的属性

![[动画层接口#关于Group：]]

![[动画层接口#关于InstanceClass：]]