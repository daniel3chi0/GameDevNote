# Character父子结构
打卡Demo关卡首先看到的就是我们主角，这个米其林男性小蓝（ALS_AnimMan_CharacterBP）派生（ALS_Base_CharacterBP）

![[Animations/LearnALSV4/从Character解析ALSV4结构Media/1.png]]


ALS_AnimMan_CharacterBP蓝图结构如下

![[Animations/LearnALSV4/从Character解析ALSV4结构Media/2.png]]


其基类ALS_Base_CharacterBP则是如下

![[Animations/LearnALSV4/从Character解析ALSV4结构Media/3.png]]


基类ALS_Base_CharacterBP在视口中如下，只有一个默认的Mesh像个火柴人

![[Animations/LearnALSV4/从Character解析ALSV4结构Media/4.png]]


在子类ALS_AnimMan_CharacterBP中把Mesh从火柴人换成了米其林小蓝。并且在Mesh(SkeletalMeshComponent)下加了如下结构

```md
VisualMeshes (SceneComponent)
	BodyMesh (SkeletalMeshComponent)
HeldObjectRoot (SceneComponent)
	SkeletalMesh (SkeletalMeshComponent)
	StaticMesh (StaticMeshComponent)
```

^5b3890

# 同骨架的骨骼网格体的动画蓝图的动态复用

![[同骨架的骨骼网格体的动画蓝图的动态复用]]

# 手持道具到OverlayLayer
![[手持道具到OverlayLayer]]

# LayerBlending

![[LayerBlending]]

