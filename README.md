# ECCV 2026 | RoboStream：给机器人 VLM 装上“持续记忆”，让长程操作不再每一步从零开始

> Project page: [https://robostream123.github.io/](https://robostream123.github.io/)  
> Paper: [https://arxiv.org/abs/2603.12939](https://arxiv.org/abs/2603.12939)  
> Code: Coming soon

![RoboStream teaser](./robostream_assets/teaser.png)

机器人长程操作的难点，很多时候并不在“看不懂当前画面”，而在“记不住刚刚发生了什么”。

现有基于视觉语言模型（VLM）的机器人规划方法，通常把每一步都当成一次独立的 observation-to-action 映射：看到当前图像，重新理解场景，再生成下一步动作。短任务里这套流程往往够用，但任务一旦变长，问题就会被逐步放大：物体被遮挡后容易被遗忘，动作造成的状态变化没有被持续追踪，前一步的微小感知错误也可能在后续步骤里连锁放大，最终导致任务前提被破坏。

RoboStream 关注的正是这个断点：机器人不应该每一步都从像素重新推理世界，而应该像人一样，在交互过程中维护一个持续更新的空间-时间记忆。

论文 **RoboStream: Weaving Spatio-Temporal Reasoning with Memory in Vision-Language Models for Robotics** 提出一个 training-free 框架，通过 **Spatio-Temporal Fusion Tokens（STF-Tokens）** 和 **Causal Spatio-Temporal Graph（CSTG）**，把视觉证据、三维几何属性和动作造成的状态变化串起来，让 VLM 在长程操作中具备更稳定的物体恒常性、因果追踪和空间推理能力。

在长程 RLBench 任务上，RoboStream 达到 **90.5%** 成功率；在更具挑战的真实世界积木搭建任务上达到 **44.4%**，显著超过 SoFar 和 VoxPoser 的 **11.1%**。

## 现有 VLM 机器人规划的瓶颈在哪

很多 VLM-based planner 的流程可以概括成三步：

1. 输入当前 RGB-D 或多视角观测；
2. 让 VLM 理解当前场景并规划动作；
3. 执行动作后进入下一轮，再从新的图像重新推理。

这个范式的问题在于，它缺少一个稳定的“世界状态”。每一步看起来都很聪明，但跨步骤的信息没有被结构化保存。

长程任务中，这会带来三个典型失败模式：

- **物体恒常性不足**：目标物被遮挡、移动或短暂离开视野后，模型可能无法持续追踪它。
- **因果链断裂**：模型知道当前画面是什么，却不知道这个状态是由哪一步动作造成的。
- **错误逐步累积**：前面某一步的空间误解，会影响后续抓取、放置、堆叠和遮盖动作。

RoboStream 的核心判断很直接：长程机器人操作需要的不只是强 VLM，还需要一个能随动作不断更新的空间-时间记忆系统。

## RoboStream 做了什么

RoboStream 没有重新训练一个大模型，而是在预训练 VLM 外围构建了一套结构化记忆机制。整体上，它做了两件关键的事：

- 用 **STF-Tokens** 把视觉观测和三维几何属性绑定起来，让物体在时间上保持可追踪；
- 用 **CSTG** 记录动作触发的状态变化，让规划器能沿着因果链推理下一步。

![RoboStream overview](./robostream_assets/Overview.png)

这使得 RoboStream 不再把每一步当成孤立问题，而是把任务执行看成一条连续的状态流。

## STF-Tokens：把物体从图像里“固定”到三维记忆中

STF-Tokens 可以理解为 RoboStream 的持久物体锚点。

传统 VLM 看到的是当前图像里的视觉 token，但这些 token 本身并不天然保存“这个物体在三维空间里是谁、在哪里、之前经历了什么”。一旦视角变化、遮挡发生，模型很容易把同一个物体重新识别成新对象，或者干脆遗忘它。

STF-Tokens 的作用，是把视觉证据和 3D 几何属性绑定起来，让物体不只是当前画面中的一块区域，而是任务执行过程中持续存在的实体。

这对机器人操作特别关键。比如“先把红色积木放到蓝色积木上，再用盒子盖住它”，第二步能否成功，取决于系统是否还记得红色积木的位置和它与蓝色积木的关系，而不是只看当前画面里露出来的像素。

## CSTG：不只是记物体，还要记动作造成了什么

如果 STF-Tokens 解决的是“物体是谁、在哪里”，那么 CSTG 解决的是“动作改变了什么”。

CSTG 全称是 **Causal Spatio-Temporal Graph**。它把任务执行过程中的对象、空间关系、动作结果组织成图结构，让模型可以追踪状态变化的因果链。

这类记忆对长程任务非常重要。因为很多机器人错误不是发生在最后一步，而是前面某个状态没有被正确继承：一个物体其实已经被移动了，但 planner 仍按旧位置规划；一个支撑关系已经被破坏，但后续动作还假设它存在；一个目标被遮挡，但系统没有保留它的空间锚点。

RoboStream 通过 CSTG 把这些状态变化显式保留下来，让后续规划不再只依赖当前图像。

## Training-Free：不微调 VLM，也能提升长程可靠性

RoboStream 的另一个重点是 **training-free**。

它并不依赖大规模机器人数据重新训练 VLM，而是利用已有预训练模型的视觉语言能力，再通过外部结构化记忆增强长程推理。这一点对机器人很现实：真实机器人数据昂贵、任务分布复杂、场景变化大，单靠继续微调很难覆盖所有长程交互情况。

RoboStream 的路线更像是在 VLM 外面补上一个“工作记忆层”：让模型不用变大，也不用重新学习全部知识，就能在多步任务里更稳定地记住物体、关系和动作后果。

## 实验：长程任务里，记忆比单步聪明更重要

在 RLBench 长程任务上，RoboStream 取得 **90.5%** 成功率，说明结构化空间-时间记忆能显著改善多步操作的稳定性。

![RLBench results](./robostream_assets/RLBench_Long_Tab.png)

更有说服力的是真实世界实验。真实积木搭建任务里，遮挡、接触误差、视觉噪声和几何关系变化都会让任务难度明显上升。RoboStream 在该设置下达到 **44.4%** 成功率，而 SoFar 和 VoxPoser 均为 **11.1%**。

![Real-world results](./robostream_assets/real_world_res.png)

这说明 RoboStream 的提升并不只是 benchmark 上的数字变化，而是在真实长程操作中减少了“前面忘了，后面崩了”的连锁失败。

在 SimplerEnv、空间推理和 Open6DOR 相关评估中，RoboStream 也保持了更稳定的表现。这类任务虽然不一定都像 RLBench 长程任务那样强调连续执行，但同样会考察模型对物体位置、空间关系和目标约束的理解能力。

![SimplerEnv results](./robostream_assets/SimplerEnv_Tab.png)

![Spatial reasoning and Open6DOR results](./robostream_assets/VQA_Tab.png)

## 定性结果：遮挡和状态变化下更稳

从定性对比来看，RoboStream 在需要持续追踪目标、理解空间关系、处理遮挡和恢复任务状态的场景中更稳定。

![Qualitative comparison](./robostream_assets/Qualitative_results.png)

这也符合它的设计逻辑：当任务中出现暂时不可见目标、堆叠关系、容器关系、支撑关系等变化时，单帧 VLM 判断容易丢上下文；而 RoboStream 可以从 STF-Tokens 和 CSTG 中取回历史状态，继续沿着任务链推理。

## Ablation：两个记忆模块缺一不可

消融实验进一步说明，RoboStream 的收益来自空间锚定和因果记忆的协同。

只记住视觉信息还不够，因为机器人需要知道物体在三维空间中的位置和关系；只记录图结构也不够，因为图中的节点必须和真实视觉证据持续对齐。

STF-Tokens 负责把对象落到可追踪的几何实体上，CSTG 负责把动作导致的状态变化组织起来。两个模块结合后，RoboStream 才能在长程任务中保持稳定的对象记忆和因果连续性。

![Ablation results](./robostream_assets/Ablation_Tab.png)

## 为什么这件事重要

机器人走向开放世界，不能只会“看一眼，做一步”。它需要在连续交互中维护一个可更新的世界模型：知道自己刚刚动了什么，知道哪些物体被遮挡但仍然存在，知道前一步动作如何改变了下一步的可行性。

RoboStream 的价值在于，它把这个问题从“继续扩大或微调 VLM”转向了“给 VLM 接入可追踪的空间-时间记忆”。这条路线对真实机器人尤其有意义，因为真实环境里错误往往不是单次识别失败，而是状态没有被连续维护。

从这个角度看，RoboStream 提供了一个很清晰的信号：长程机器人智能的下一步，可能不只是更强的视觉语言理解，而是更可靠的记忆、几何锚定和因果状态追踪。

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
