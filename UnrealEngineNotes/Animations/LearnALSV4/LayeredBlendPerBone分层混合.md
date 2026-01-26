这个节点主要用于分层混合动画，使用的方式在官方文档对应的部分有介绍[虚幻引擎中的动画蓝图混合节点 | 虚幻引擎 5.7 文档 | Epic Developer Community](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/animation-blueprint-blend-nodes-in-unreal-engine)

# `Blend Depth` 参数详解

`Blend Depth` 的值定义了混合权重从 `Branch Filter` 中指定的骨骼（`Bone Name`）开始，向其子骨骼链传递时的**渐变方式和影响深度**。其工作原理和效果根据取值的不同，可以分为以下几种情况：

1. **`Blend Depth` 为 0 或 1**：当 `Blend Depth` 设置为 0 或 1 时，其效果是相同的
    
    这意味着从你指定的 `Bone Name` 骨骼开始，**该骨骼及其所有的子骨骼都将以 100% 的权重（即 `Blend Weights` 参数设定的值）完全参与混合**，立即达到设定的混合强度，没有渐变过程
    
    例如，如果你指定骨骼为 `spine_03`（胸部），并设置 `Blend Depth` 为 0，那么从 `spine_03` 开始，包括双臂在内的所有子骨骼都会完全被 `Blend Pose` 的动画所覆盖。
    
2. **`Blend Depth` 为大于 1 的正整数 (N)**：当 `Blend Depth` 设置为一个大于 1 的正整数（例如 2 或 3）时，混合效果会呈现一个**从指定骨骼开始，向子骨骼方向逐渐增强的渐变过程
    
    - **具体规则**：从指定的 `Bone Name` 开始算作第 1 层，其子骨骼为第 2 层，以此类推。`Blend Depth` 的值 `N` 表示**从第 `N` 层子骨骼开始，混合权重才达到 100%**。而在到达第 `N` 层之前，每一层的混合权重会线性递增。
    - **举例说明**：假设骨骼链为 `pelvis -> Bone2 -> Bone3`，指定 `Bone Name` 为 `pelvis`。
        - 如果 `Blend Depth = 2`，则表示从 `pelvis`（第1层）开始的第2根骨骼（即 `Bone2`）达到100%混合。那么，`pelvis` 本身的混合权重约为 50%，`Bone2` 为 100%，`Bone3` 也为 100%。
          
        - 如果 `Blend Depth = 3`，则表示从 `pelvis`（第1层）开始的第3根骨骼（即 `Bone3`）达到100%混合。那么，`pelvis` 的混合权重约为 33%，`Bone2` 约为 66%，`Bone3` 为 100%。这种设置可以实现非常平滑的动画过渡，避免动作在关节连接处出现生硬的切换。
    
3. **`Blend Depth` 为 -1**：当 `Blend Depth` 设置为 -1 时，其效果类似于一个“阻断”开关。它意味着**仅指定的 `Bone Name` 骨骼本身参与混合，而其所有的子骨骼都将不参与混合，完全保留 `Base Pose` 的动画**
    
    。这在你只想让混合效果精确控制在某一块骨骼，而不希望影响到其肢体时非常有用。


# `Curve Blend Option` 参数详解
在 Unreal Engine 的动画蓝图中，`Layered blend per bone` 节点的 **`Curve Blend Option`（曲线混合选项）** 是一个关键但常被忽视的高级功能。它专门用于控制动画曲线（Animation Curves）在分层混合时的计算方式，与处理骨骼变换（Transform）的 `Branch Filter` 和 `Blend Depth` 参数相辅相成，共同决定了最终动画的细节表现。

根据搜索结果，`Curve Blend Option` 提供了多种模式，每种模式对应着不同的曲线值计算逻辑，适用于不同的动画需求。

## 一、 `Curve Blend Option` 的核心模式详解

`Curve Blend Option` 决定了从 `Base Pose`（基础姿势）和 `Blend Pose`（混合姿势）传入的动画曲线值如何被计算，以产生最终输出给 `Final Pose` 的曲线值。其主要模式如下：

1. **Override（覆盖模式）**
    - **工作原理**：这是最“霸道”的模式。只要 `Blend Pose` 参与了混合（即其混合权重 `Blend Weight > 0`），并且 `Blend Pose` 自身携带了有效的动画曲线，那么 `Blend Pose` 的曲线值将**完全覆盖**掉 `Base Pose` 的曲线值，无论 `Blend Weight` 的具体数值是多少。
      
    - **应用场景**：适用于需要 `Blend Pose` 动画**完全掌控**某些状态或特效参数的情况。例如，一个“受伤”的蒙太奇动画可能带有控制角色面部痛苦表情或身上流血特效的曲线。当播放这个蒙太奇时，你希望这些曲线值完全生效，不受基础移动动画中可能存在的同类曲线值影响。
      
2. **Do Not Override（不覆盖模式）**
    - **工作原理**：与 `Override` 模式相反。只要 `Base Pose` 携带了有效的动画曲线，最终的曲线值就**始终取自 `Base Pose`**，`Blend Pose` 的曲线值将被完全忽略。
      
    - **应用场景**：当 `Blend Pose` 是用于叠加局部骨骼动画（如上半身射击），但你希望角色的整体状态（如移动速度、 stamina 值等由曲线控制的参数）仍然由 `Base Pose`（如下半身移动动画）来主导时，使用此模式。
      
3. **Blend by Weight（按权重混合模式）**
    - **工作原理**：在此模式下，最终的曲线值是 `Base Pose` 的曲线值与 `Blend Pose` 的曲线值，按照 `Blend Weight` 进行线性插值的结果。计算公式可以理解为：`最终值 = Base值 + Blend Weight * Blend值`。但需要注意的是，该模式**必须为节点设置有效的 `Branch Filter` 骨骼**，否则混合可能不生效。如果不想影响实际骨骼，可以创建一个虚拟骨骼（如 `VB_Curves`）作为过滤骨骼。
      
    - **应用场景**：这是最符合直觉的“混合”模式。适用于希望曲线值能平滑过渡的场景。例如，用一个 `Blend Pose` 控制角色“疲惫度”曲线，并希望通过调整 `Blend Weight`（从0到1）来让角色从精力充沛逐渐过渡到精疲力尽的状态。
      
4. **Normalized by Weight（按权重归一化混合模式）**
    - **工作原理**：此模式同样**必须设置 `Branch Filter` 骨骼**。其计算方式与 `Blend by Weight` 类似，但会对结果进行归一化处理。计算公式约为：`最终值 = (Base值 + Min(Blend Weight, 1) * Blend值) / (Min(Blend Weight, 1) + 1)`。这确保了无论 `Blend Weight` 多大，最终值都会被“拉回”到一个合理的范围内，防止曲线值因权重过大而失控。
      
    - **应用场景**：适用于需要更稳定、受控的曲线混合，尤其是当 `Blend Weight` 可能被动态设置为较大值，但又希望曲线输出保持在一定区间内时。
      
5. **其他模式**
    - **Use Base Pose**：曲线值仅取自 `Base Pose`，效果类似 `Do Not Override`。
    - **Use Min Value**：取 `Base Pose` 和 `Blend Pose` 曲线值中的**最小值**。
    - **Use Max Value**：取 `Base Pose` 和 `Blend Pose` 曲线值中的**最大值**。

## 二、 设计意图与工作流中的重要性

`Curve Blend Option` 的设计，赋予了动画师和程序员对**动画驱动参数**进行精细化分层控制的能力。动画曲线不仅可以驱动材质参数、粒子特效的显示与强度，还能在动画蓝图中通过 `Get Curve Value` 节点获取，进而驱动逻辑状态、混合空间参数或物理效果。

在复杂的分层动画工作流中，`Layered blend per bone` 节点可能同时处理多个 `Blend Pose` 的输入。`Curve Blend Option` 确保了在骨骼姿势被分层混合的同时，与之关联的曲线数据也能按照预期的逻辑（覆盖、混合、取极值等）进行整合，从而保证动画表现的完整性。

## 三、 使用技巧与注意事项

1. **与 `Branch Filter` 的协同**：`Blend by Weight` 和 `Normalized by Weight` 模式**强制要求设置 `Branch Filter`**。这意味着曲线混合的范围可以通过骨骼遮罩来精确控制。如果你只想混合曲线而不想影响任何骨骼姿势，一个常见的技巧是**创建一个不影响实际变形的虚拟骨骼（Dummy Bone），并将其设置为 `Branch Filter`**。
   
2. **性能与选择**：`Override` 和 `Do Not Override` 模式计算开销最小，因为它们不涉及插值运算。而 `Blend by Weight` 和 `Normalized by Weight` 需要进行实时计算。在性能敏感的场景下，应优先考虑使用计算简单的模式。
   
3. **调试与验证**：在动画蓝图中使用 `Get Curve Value` 节点获取曲线值时，获取到的是经过所有混合节点计算后、最终汇入 `Output Pose` 的曲线值。因此，在调试分层混合效果时，务必在最终输出前检查曲线值是否符合 `Curve Blend Option` 设定的预期。

# BlendMasks
[虚幻引擎中的混合遮罩和混合描述 | 虚幻引擎 5.7 文档 | Epic Developer Community](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/blend-masks-and-blend-profiles-in-unreal-engine)