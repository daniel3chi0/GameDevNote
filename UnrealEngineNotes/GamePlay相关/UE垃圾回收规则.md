引用这个篇文章部分内容，该文章中还有详细的解释底层流程。[https://zhuanlan.zhihu.com/p/448412111](https://zhuanlan.zhihu.com/p/448412111)
[https://www.cnblogs.com/kekec/p/13045042.html](https://www.cnblogs.com/kekec/p/13045042.html)
[http://www.v5xy.com/?p=808](http://www.v5xy.com/?p=808)

垃圾回收的核心
![[GamePlay相关/UE垃圾回收规则(Media)/1.png]]

# 纳入被自动GC机制管理的条件：
1. UObject对象中的UObject* 成员变量必须被UPROPERTY宏定义修饰，自动被加入到引用。
2. UObject对象实现了派生的AddReferencedObjects接口，手动将UObject* 变量添加到引用。
3. 对于Struct结构体（以及普通C++类）中引用的UObject，必须继承FGCObject对象，并实现派生的AddReferencedObjects接口，手动将UObject* 变量加入引用。
![[GamePlay相关/UE垃圾回收规则(Media)/2.png]]

在Unreal Engine中使非UObject对象加入垃圾回收系统，是一项通过**继承`FGCObject`并实现`AddReferencedObjects`接口**来完成的标准化操作。该机制的本质是让非UObject类能够向GC系统“申报”其所依赖的`UObject`引用，从而在标记-清除过程中构建起正确的可达性链条，防止资源被过早释放。这是UE混合内存管理模型（`UObject`的自动GC与C++对象的手动/智能指针管理）中至关重要的桥梁，确保了复杂项目中不同类型对象间引用关系的安全性与一致性。

如果这个非Uobject类上没有Uobject的成员变量，可以只用智能指针来管理。
需要注意的是，`TSharedPtr`、`TWeakPtr`等UE自定义智能指针主要用于管理**非UObject对象**的共享所有权和生命周期，它们与`UObject`的GC系统是两套独立的机制。一个非UObject类如果使用`TSharedPtr`管理其内部状态，这与它继承`FGCObject`来保护引用的`UObject`是并行不悖的。

# 一个UObject不会被GC的条件：

1. UObject对象被加入到Root根上(调用AddRoot函数)（无论包裹类是UObject类还是普通C++类）。
2. 直接或者间接被根节点Root对象引用（UPROPERTY宏修饰的UObject* 成员变量 注：UObject* 放在UPROPERTY宏修饰的TArray、TMap中也可以）
3. 直接或间接被存活的FGCObject对象引用

结点上注定是UObject对象，~~普通C++对象用智能指针管理，不需要AddRoot或是AddReferencedObjects~~。

而FGCObject对象不在GC链的结点上。
![[GamePlay相关/UE垃圾回收规则(Media)/3.png]]

# 清扫：

1. 一个普通C++类不继承FGCObject，身上引用一个Uobject对象，一段时间后Uobject对象被清理。（没有AddReferencedObjects不在GC链管理中，系统不知道对象被引用了，所以会被清理） ![[GamePlay相关/UE垃圾回收规则(Media)/4.png]]
2. 一个Uobject类，身上引用非反射Uobject对象，一段时间后Uobject对象被清理。（没有Uproperty或AddReferencedObjects不在GC链管理中，系统不知道对象被引用了，所以会被清理）

下面这里的自动清空为nullptr是只虽然对象被清理了，但指针还是指向了不知名内存中，所造成的野指针。
![[GamePlay相关/UE垃圾回收规则(Media)/5.png]]