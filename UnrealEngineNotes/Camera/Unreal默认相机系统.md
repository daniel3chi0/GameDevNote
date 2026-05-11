引用文章：[虚幻引擎相机系统原理机制源码剖析-你丫才建模-SurfCG](https://www.surfcg.com/article/835644)
[【UEGamePlay】- 3C篇（三） : Camera - 丨桐 - 博客园](https://www.cnblogs.com/Tong0115/p/19201040)

相机的本质是封装和维护一套具有视口目标的位置和旋转等信息的数据结构，本体无法直接进入渲染流程，在Runtime下主要由PlayerCameraManager提供当前激活的相机，利用其中的数据提交至渲染器(RHI)。编辑器中看到的相机实体则代表了玩家的视角。

# Camera
相机（摄像机，Camera） 代表了玩家的视角，比如玩家如何查看世界。因此， 相机只和玩家控制的人物有关。PlayerController 会指定一个PlayerCameraManager类以此计算玩家从哪个位置和角度 观察世界。

# CameraActor
本质上就是一个默认带有CameraComponent的Actor，有关相机的所有属性和行为均在 CameraComponent 中设置。CameraActor 类主要用作 CameraComponent 的包装器，以使相机可以被直接放置在该关卡内，而非另一个类中。

# CameraComponent
CameraComponent 代表相机视角和设置，比如投射类型（Projection Type）、视野（Field Of View） 和 后期处理覆盖（Post-Process Overrides）。如果 ViewTarget 是一个 CameraActor 或者包含 CameraComponent 并且 bFindCameraComponentWhenViewTarget 设为 true 的Actor，bTakeCameraControlWhenPossessed 是一个可以为任何 Pawn 设置的相关属性，Pawn 会在 PlayerController 占有时自动变为 ViewTarget。

# Actors 和 PlayerControllers
PlayerControllers 和 Actors 都含有 CalcCamera 函数。如果 bFindCameraComponentWhenViewTarget 为 true，而且 CameraComponent 存在，Actor 的 CalcCamera 函数返回 Actor 中的首个 CameraComponent 的相机视图，否则，它获取 Actor 的位置和旋转方向。在 PlayerController 类中，CalcCamera 函数的行为方式与第二种情况类似，如果存在占有 Pawn ，则返回其位置以及 PlayerController 的控制旋转。