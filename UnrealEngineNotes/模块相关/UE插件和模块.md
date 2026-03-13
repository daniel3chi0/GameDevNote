一个插件目录大概如下
![[模块相关/UE插件和模块Media/1.png]]

可以看到上图中插件目录下和项目目录一样代码都是放在source目录中的同样有次级目录public和private，同样public存放.h文件private存放.cpp文件。

ue4项目是由多个模块构成的，插件也是由一个或多个模块构成的
ue4工程和插件分别至少需要一个主模块。区分各个模块的文件就是我们经常看见的xx.build.cs文件它的后缀是cs没错这是一个C#文件。

要在工程中使用插件就必须在项目的主模块中加入插件的主模块
![[模块相关/UE插件和模块Media/2.png]]

同时插件中的代码类需要有个导出宏不然会报错（和插件主模块名字类似，此时插件中只有一个模块就是主模块stonetoolclass因为只有一个stonetoolclass.build.cs文件）
![[模块相关/UE插件和模块Media/3.png]]

到这里就能正常使用插件了
我在之后上看到有人在这步骤之后还有一步操作（后续发现其实这个操作相当于在编辑器插件窗口手动勾选启用插件，勾选启用插件后会在这个文件中模块中加入插件模块以及插件的{}）
![[模块相关/UE插件和模块Media/4.png]]

有的时候手动勾选会出现AdditionalDependencies的字段，用于声明模块在加载前必须满足的**额外依赖关系**。它确保了模块间的正确加载顺序和功能完整性。理解AdditionalDependencies的关键在于区分它与模块的.Build.cs文件中定义的依赖关系(PublicDependencyModuleNames和PrivateDependencyModuleNames)的不同角色。

- **.Build.cs中的依赖（Public/PrivateDependencyModuleNames）**：这些是**编译期和链接期**的依赖。它们告诉UnrealBuildTool（UBT）当前模块在编译源代码时需要哪些其他模块的头文件，以及在链接成动态库（DLL）时需要链接哪些其他模块的库文件。这确保了代码能正确编译并通过链接器检查。
- **.uproject/.uplugin中的AdditionalDependencies**：这是**运行期加载**的依赖。它告诉虚幻引擎的运行时模块管理器，在准备加载某个模块的DLL到内存中并调用其初始化函数时，必须确保哪些模块已经完成了它们自身的初始化。

通常，一个模块在.Build.cs中声明的PublicDependencyModuleNames，也应该被添加到其在.uproject或.uplugin中的AdditionalDependencies里。这是因为，如果你在编译时依赖了某个模块的接口（头文件），那么在运行时你的模块初始化代码很可能需要调用那个模块的功能，因此必须确保它在你的模块之前被加载并初始化。

![[模块相关/UE插件和模块Media/5.png]]

**uproject文件中的模块声明主要解决的是运行时加载顺序问题**

游戏主模块和插件都有这种类型的配置文件，分别是.uproject文件和.uplugin文件
**自定义插件和模块的的相同之处**：
1. 都需要在主模块的uproject，添加模块或插件的对应字段。
2. 都需要在Build.cs中添加插件和模块字段。

**自定义插件和模块不同之处**：
1. 如果只是添加项目模块还需要改动的是target文件 而插件不需要
 ![[模块相关/UE插件和模块Media/6.png]]

模块创建方式网上很多：[https://blog.csdn.net/m0_45029082/article/details/104653550](https://blog.csdn.net/m0_45029082/article/details/104653550)
这个文章中遗漏了创建模块时要对Target.cs中ExtraModuleNames添加模块名这一步是告诉UBT该模块要最终编译链接到可执行文件中。最后记得要重新生成项目Generate Visual Studio project files
