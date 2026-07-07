# ECCV 2026 | RoboStream：打通 VLM 机器人长程操作的记忆断层

> **Paper**: [RoboStream: Weaving Spatio-Temporal Reasoning with Memory in Vision-Language Models for Robotics](https://arxiv.org/abs/2603.12939)  
> **Project**: [robostream123.github.io](https://robostream123.github.io/)  
> **Venue**: ECCV 2026  
> **Code**: Coming soon

![RoboStream teaser](./robostream_assets/teaser.png)

VLM 机器人已经能看懂当前画面，却仍很难完成真正稳定的长程操作。

问题不在于模型完全不懂物体，也不在于它不会生成下一步动作。更核心的瓶颈是：当机器人连续抓取、堆叠、遮挡和恢复物体时，当前许多 VLM-based planner 仍然像在做一连串彼此割裂的单帧问答。每执行一步，系统重新看图、重新推理，却缺少一个持续更新的世界状态。

RoboStream 关注的正是这个断层：**机器人不是只需要看见当前世界，还需要记住世界如何被动作一步步改写。**

为此，RoboStream 提出一个 training-free 的空间-时间记忆框架。它不微调 VLM，而是在 VLM 推理循环外构建结构化记忆：用 **Spatio-Temporal Fusion Tokens (STF-Tokens)** 将视觉证据锚定到 3D 几何实体上，再用 **Causal Spatio-Temporal Graph (CSTG)** 记录动作触发的状态变化。这样，机器人可以在多步操作中持续追踪物体身份、空间关系和因果历史。

实验也把这个问题推到了更接近真实机器人的设置里。RoboStream 在 long-horizon RLBench 上达到 **90.5%** 平均成功率；在真实 Franka Research 3 机械臂的 Hard 级 block building 中达到 **44.4%**，Hard 级 block disassembly 中达到 **66.7%**；在 Hide-and-Restore 全遮挡恢复任务中达到 **88.9%**，而 SoFar 和 VoxPoser 在该任务上均为 **0%**。

## 现有 VLM 机器人卡在哪里

当前主流 VLM-based robot planner 的流程大体相似：输入当前视觉观测，让 VLM 理解场景并生成下一步动作，执行后再基于新图像重新推理。

这个范式在短任务里很自然。一次抓取、一次放置，当前帧往往已经包含足够信息。但长程操作会很快暴露三个问题。

- **物体恒常性不足**：目标被移动、遮挡或短暂离开视野后，模型可能无法稳定追踪它；
- **空间关系无法继承**：上一轮形成的支撑、堆叠、容器关系，不能可靠传递到后续规划；
- **错误链式放大**：早期一次小的空间误判，会在后续抓取、放置、遮盖和恢复中持续累积。

这类失败不是简单的“看图能力不够”。它更像是机器人缺少一条任务执行过程中的状态流：物体在哪里，关系如何变化，刚才的动作造成了什么后果，下一步应该基于哪个世界状态继续。

## RoboStream：给 VLM 外接一层可更新的时空记忆

RoboStream 的设计可以分成两条主线。

第一条是 **Spatio-Temporal Grounding**。STF-Tokens 将每个任务相关物体的视觉外观与 3D 几何属性绑定，包括中心位置、形状和空间结构。这样，VLM 不再只面对一片图像区域，而是可以引用跨步骤保持身份连续的 3D 对象实例。

第二条是 **Causal Memory Tracking**。CSTG 将对象、空间关系和动作后果组织成因果图，显式记录动作引发的状态转移。一个物体被移动后，原位置会失效；一个目标被杯子盖住后，它不可见但仍存在；一个结构被拆开后，支撑关系也必须更新。CSTG 的作用，就是让这些变化进入后续推理。

图 1：RoboStream 整体框架。系统从 RGB-D 观测中构建 STF-Tokens，再更新 CSTG，并将结构化记忆送入 VLM 进行长程规划。

![RoboStream overview](./robostream_assets/Overview.png)

这使 RoboStream 与许多反应式 planner 形成了清晰区别：它不是每一步都从像素重新开始，而是在任务执行过程中维护一个持续演化的世界模型。

## STF-Tokens：把物体从图像区域变成 3D 实体

普通视觉 token 更擅长描述当前图像里的局部区域。但机器人操作中的“物体”不是静态像素块，而是会被抓取、移动、遮挡、堆叠、放入容器，并在之后重新出现的三维实体。

STF-Tokens 的关键，是把视觉证据和显式几何属性绑定到同一个对象上。对于长程任务，这个绑定非常重要：模型不仅要知道“红色积木在图里哪里”，还要知道它在 3D 空间中的位置、它是否仍然是前一步移动过的那个对象，以及它和其他物体形成了什么关系。

这种表示让空间推理从“当前帧猜测”变成“对象级追踪”。当后续动作依赖此前状态时，VLM 可以基于稳定的对象引用继续推理，而不是反复从图像中重建几何关系。

## CSTG：让机器人记住动作改变了什么

长程操作真正困难的地方在于，动作会不断改变世界。

如果机器人把蓝色积木放到黄色积木上，后续放置动作就必须把这个新结构作为支撑关系的一部分。如果机器人用杯子盖住目标，目标虽然消失在图像里，但它的身份和最后位置仍然需要被保留。如果机器人先拆掉上层积木再取出底层目标，那么每一步都必须服从前面的因果依赖。

CSTG 记录的正是这些动作造成的状态转移。它让规划器知道“刚刚发生了什么”，并在物体被移动、遮挡或恢复时保留可追溯的因果历史。

这也是 RoboStream 在遮挡和恢复任务中表现更强的关键。直接视觉证据消失后，系统仍可以从因果记忆中恢复隐藏物体的身份和位置。

## 真机实验：把记忆问题放进真实物理世界

RoboStream 的真实世界实验并不是简单展示，而是围绕长程记忆设计的压力测试。

作者在 Franka Research 3 机械臂上进行实验，使用前视 Intel RealSense D435i RGB-D 相机进行视觉和深度感知。任务覆盖 block building、block disassembly 和 block hide-and-restore 三类场景，共 **21 个真实任务配置**，每个配置执行 3 次。

图 2：真实机器人实验设置。系统在 FR3 机械臂和 RealSense RGB-D 相机上进行长程积木操作。

![Real-world setup](./robostream_assets/appendix_robotenv.png)

其中 Hard 级任务最高扩展到 **7 个物体、6-7 个连续步骤**。这类任务不仅要求机器人抓取准确，还要求它持续维护支撑关系、操作顺序和物体身份。一旦提前取错支撑块，或忘记被遮挡目标，后续动作会直接崩掉。

图 3：Hard 级真实长程积木操作。任务要求机器人在多物体、多步骤场景中维持结构稳定和正确操作顺序。

![Real-world Hard tasks](./robostream_assets/appendix_realworld3.png)

在 Hard 级 block building 中，RoboStream-235B 达到 **44.4%** 成功率；在 Hard 级 block disassembly 中达到 **66.7%**。相比之下，SoFar 和 VoxPoser 在这些真实长程任务中难以超过 **11.1%**。

更能体现记忆价值的是 Hide-and-Restore。机器人需要先隐藏目标，执行中间干扰步骤，再取回被遮挡物体并恢复原始结构。此时当前图像已经不能提供完整答案，系统必须依赖此前的对象身份、最后位置和状态转移记录。

图 4：Hide-and-Restore 真实任务。目标被完全遮挡后，RoboStream 仍能依靠因果记忆恢复隐藏对象并还原场景。

![Real-world Hide and Restore](./robostream_assets/appendix_realworld4.png)

在这类全遮挡恢复任务中，RoboStream-235B 达到 **88.9%**，RoboStream-8B 也保持 **33.3%**；SoFar 和 VoxPoser 则均为 **0%**。这说明差距并不只是来自更强的单步视觉理解，而是来自持续状态追踪和因果记忆。

![Real-world manipulation results](./robostream_assets/real_world_res.png)

## 模拟长程任务：不是只靠更大的 VLM

RoboStream 还在 RLBench 中构建了 8 个复杂长程任务，包括 goal-image guided 的多物体搭建任务，以及 text-guided 的遮挡、恢复和重排任务。每个方法在每个任务上测试 25 个 episode。

结果显示，RoboStream-235B 在 long-horizon RLBench 上达到 **90.5%** 平均成功率，明显高于 SoFar 的 **28.0%** 和 VoxPoser 的 **26.5%**。即使是 RoboStream-8B，也达到 **58.0%**。

![RLBench long-horizon results](./robostream_assets/RLBench_Long_Tab.png)

这个结果说明，模型规模有帮助，但不是唯一答案。RoboStream-8B 已经大幅超过更大的反应式基线，说明结构化时空记忆本身就是长程稳定性的关键来源。

消融实验进一步验证了这一点。去掉 CSTG 后，平均成功率从 **90.5%** 降到 **14.5%**；去掉 STF-Tokens 后，降到 **79.5%**；两者同时去掉时，仅为 **12.0%**。

![Ablation results](./robostream_assets/Ablation_Tab.png)

这组结果很清楚：CSTG 提供长程逻辑和因果连续性，STF-Tokens 提供物理执行中的几何精度。前者让机器人知道顺序和历史，后者让每一步落到正确的空间位置。

## 泛化与空间推理

除了长程操作，RoboStream 还在 SIMPLER、6-DoF SpatialBench 和 Open6DOR V2 上进行评估，覆盖 zero-shot cross-embodiment generalization、空间 VQA 和 6-DoF object rearrangement。

在这些任务中，STF-Tokens 的作用更加直接：它为 VLM 提供稳定的几何参照，使模型在位置、方向和物体关系上不只依赖图像-文本对齐。

![SIMPLER results](./robostream_assets/SimplerEnv_Tab.png)

![Spatial reasoning and Open6DOR results](./robostream_assets/VQA_Tab.png)

## 定性结果：失败往往发生在忘记之前

定性对比显示，SoFar 和 VoxPoser 的失败常常不是完全看不见目标，而是没有把此前状态传递到后续步骤。物体被遮挡后，历史位置丢失；结构被改变后，支撑关系没有更新；目标恢复时，模型无法回到正确的因果链。

RoboStream 通过 STF-Tokens 和 CSTG 同时保留对象 grounding 与因果记忆，使 VLM 能在长程任务中持续对齐“现在看到什么”和“之前发生过什么”。

![Qualitative comparison](./robostream_assets/Qualitative_results.png)

## 为什么这件事重要

VLM 进入机器人之后，能力瓶颈不会只来自视觉识别，也不会只来自语言理解。真实长程操作要求系统在连续交互中维护一个稳定、可更新、可追溯的世界状态。

RoboStream 的价值在于，它把这个状态显式建出来：视觉证据提供对象外观，3D 几何提供空间锚点，CSTG 提供动作历史和因果链。

如果说很多 VLM robot planner 仍像是在每一步重新看图做题，那么 RoboStream 更像是在让机器人带着记忆行动。它不仅知道当前画面里有什么，也知道这些物体是怎样一步步变成现在这样的。

这正是 VLM 机器人走向真实长程操作时必须补上的一环：**persistent spatio-temporal memory for an evolving physical world**。

## Citation

```bibtex
@misc{huang2026robostreamweavingspatiotemporalreasoning,
  title={RoboStream: Weaving Spatio-Temporal Reasoning with Memory in Vision-Language Models for Robotics},
  author={Yuzhi Huang and Jie Wu and Weijue Bu and Ziyi Xiong and Gaoyang Jiang and Ye Li and Kangye Ji and Shuzhao Xie and Yue Huang and Chenglei Wu and Jingyan Jiang and Zhi Wang},
  year={2026},
  eprint={2603.12939},
  archivePrefix={arXiv},
  primaryClass={cs.RO},
  url={https://arxiv.org/abs/2603.12939}
}
```
