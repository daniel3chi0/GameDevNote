首先需要自定义UE4 AssetType资产用插件的方式创建，一篇不错的文章[https://blog.csdn.net/qq_29667889/article/details/109307203](https://blog.csdn.net/qq_29667889/article/details/109307203)
如文章所述我创建自己的生成器插件
结构如下分成runtime模块和editor模块
对于新创建的空插件，只有一个Runtime类型的模块，复制这个模块文件，把对应的文件夹名和源文件等都改成带有“Editor”，作为Editor模块。
![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/1.png]]

按照文章的内容先添加好依赖
在Editor模块的配置cs文件中要添加对Runtime模块和UnrealEd的依赖. 在插件的uplugin中添加Editor模块。
CustomJoystickCreatorEditor.Build.cs
![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/2.png]]

CustomJoystickCreator.uplugin
![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/3.png]]

接着我们需要自定义资产类
比如我们在编辑器中看到的这种
![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/4.png]]

这个assetype 是继承自Uobject的
我们这里就不重新自定义一个了，只需要在touchinterface上拓展就行了。
![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/5.png]]

![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/6.png]]

其中要把数据结构重写一下
也就是上图中的ControlEXs
![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/7.png]]

![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/8.png]]

新加两个旋转的相关的变量
构造函数如下
![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/9.png]]

注意资源类型要有这个宏标签否则不能在蓝图中使用
![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/10.png]]

然后是要使用自定义的activate函数
![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/11.png]]

因为参数和Utouchinterface不同所以不能重写（override）只能采用这种隐藏的方式
隐藏的方式不能用多态 所以需要注意使用。方法的实现很简单基本就是抄touchinterface里的实现
再多加两行我们上边定义的变量相关内容
![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/12.png]]

接着在插件editor模块中创建资源的工厂类
![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/13.png]]

实现下
![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/14.png]]

做完这一步，如果是4.25版本以前的工程，重新编译之后打开引擎，在ContentBrowser中创建自定义的资产类实例了。 但是4.25版本需要再多做一步：在Editor模块中利用AssetTypeAction注册 自定义资产。
![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/15.png]]

![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/16.png]]

在编辑器模块中注册资产实例
![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/17.png]]

接着是复制一份引擎中默认的touchinterface 的Slate UI类改成我么自己需要的
也就是上文中提到的activate里的只能指针参数
大体上和svirtualjoystick中一样
多加里几个特殊的
![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/18.png]]

![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/19.png]]

![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/20.png]]

![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/21.png]]

为了实现我们的功能只要在ui中的onpaint中添加逻辑即可
![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/24.png]]

角度的计算
![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/22.png]]

最终插件结构如下
![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/23.png]]

最终debug后还有在插件的runtime中引入依赖
![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/25.png]]

除此之外还需要在相应的controller中
重写和隐藏 创建和激活方法
![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/26.png]]

![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/27.png]]

注意在这里的变量类型
![[Plugin&Editor/UE4自定义可旋转Joystick (Media)/28.png]]
变量也支持 override
基本上就这样了