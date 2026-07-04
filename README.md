# ECCV 2026 | RoboStream：让机器人 VLM 拥有“连续记忆”，长程操作不再每一步重开

> **Paper**: [RoboStream: Weaving Spatio-Temporal Reasoning with Memory in Vision-Language Models for Robotics](https://arxiv.org/abs/2603.12939)  
> **Project**: [robostream123.github.io](https://robostream123.github.io/)  
> **Venue**: ECCV 2026  
> **Code**: Coming soon

![RoboStream teaser](./robostream_assets/teaser.png)

大模型进入机器人之后，一个看似朴素的问题被重新放大了：机器人到底是在“理解世界”，还是只是在“理解当前这一帧图像”？

对短任务来说，这个差别并不明显。模型看到桌面、识别目标、生成动作，很多时候就能完成一次抓取或放置。但当任务变成长程操作，问题很快出现：物体被遮挡后还在不在？刚刚移动过的积木现在应该在哪里？前一步搭好的结构，后一步还能不能继续作为支撑？

现有很多 VLM 机器人方法的问题，并不是单步看不懂，而是**跨步骤记不住**。

RoboStream 关注的正是这个断层。它提出一种 training-free 的空间-时间记忆框架，用 **Spatio-Temporal Fusion Tokens（STF-Tokens）** 绑定视觉证据与 3D 几何属性，再用 **Causal Spatio-Temporal Graph（CSTG）** 记录动作引发的状态变化，让机器人在长程任务中持续追踪对象、关系与因果链。

结果也很直接：RoboStream 在长程 RLBench 任务上达到 **90.5%** 成功率；在真实世界积木操作中达到 **44.4%**，显著超过 SoFar 和 VoxPoser 的 **11.1%**。

## 问题不在“看见”，而在“记住”

当前主流 VLM-based robot planner 的流程，大多可以概括为：

1. 输入当前视觉观测；
2. 让 VLM 理解场景并生成下一步动作；
3. 执行动作后，再基于新的图像重新推理。

这种范式在单步任务里很自然，但在长程任务中会暴露一个根本问题：每一步都像是一次新的问答，系统缺少一个持续更新的世界状态。

于是，机器人会遇到三类典型失败：

- **物体恒常性不足**：目标被遮挡、移动或短暂离开视野后，模型无法稳定追踪它；
- **空间关系断裂**：上一步建立的支撑、堆叠、容器关系，没有可靠传递到后续规划；
- **错误链式放大**：早期一次小的空间误判，会在后续抓取、放置、遮盖和恢复动作中持续累积。

RoboStream 的核心判断是：长程机器人操作需要的不只是更强的视觉语言模型，还需要一个能随动作不断更新的空间-时间记忆。

## RoboStream：在 VLM 外接一个可更新的记忆层

RoboStream 没有重新训练一个大模型，而是在预训练 VLM 外围构建了一套结构化记忆机制。

它的设计可以拆成两条线：

- **STF-Tokens**：把视觉 token 和 3D 几何属性绑定起来，让物体成为可持续追踪的实体；
- **CSTG**：把对象、动作和状态变化组织成因果图，让规划器知道“刚刚发生了什么”。

![RoboStream overview](./robostream_assets/Overview.png)

这意味着，RoboStream 不是每一步都从像素重新开始，而是在任务执行过程中不断维护一条状态流：物体在哪里、关系如何变化、哪些动作导致了当前局面。

## STF-Tokens：把物体从图像区域变成三维实体

普通视觉 token 往往只描述当前图像里的局部区域。问题是，机器人操作中的“物体”并不是一块静态像素，而是一个会被移动、遮挡、堆叠、放入容器、再重新出现的三维实体。

STF-Tokens 的作用，就是把视觉证据和显式 3D 几何属性编织在一起，让模型在不同时间、不同视角和不同遮挡条件下，仍能追踪同一个对象。

举个例子：如果任务要求“先把红色积木放到蓝色积木上，再用盒子盖住它”，第二步能否成功，取决于系统是否还记得红色积木已经被放到了哪里，以及它和蓝色积木之间形成了什么空间关系。

这不是简单的图像识别问题，而是对象记忆问题。

## CSTG：让机器人知道动作改变了什么

只记住对象还不够。长程任务真正困难的地方在于，动作会不断改变世界。

CSTG，即 **Causal Spatio-Temporal Graph**，记录的正是这些由动作触发的状态转移。它把对象、空间关系和动作后果组织成图结构，使后续规划可以沿着因果链继续推理。

这在真实操作里尤其重要。一个积木被移动后，原来的位置就不再可靠；一个盒子盖住目标后，目标虽然不可见，但并没有消失；一个结构被拆开后，后续动作不能再假设它仍然稳定存在。

RoboStream 用 CSTG 把这些变化显式保存下来，从而减少“当前图像看起来没问题，但历史状态已经错了”的失败。

## Training-Free：不微调，也能让长程推理更稳

RoboStream 的另一个关键点是 training-free。

在机器人场景中，真实数据昂贵、任务分布复杂、长程交互又很容易产生组合爆炸。单靠继续微调 VLM，很难覆盖所有可能的物体状态、遮挡模式和动作后果。

RoboStream 选择了另一条路：不改变 VLM 本身，而是给它补上一套结构化外部记忆。

这使得模型既能保留预训练 VLM 的视觉语言能力，又能在多步任务中拥有更稳定的对象追踪、空间关系维护和因果状态更新。

## 实验：长程任务里，记忆开始成为核心能力

在长程 RLBench 任务上，RoboStream 达到 **90.5%** 成功率。这个结果说明，当任务需要跨多个步骤维持目标和状态时，结构化空间-时间记忆带来的提升非常明显。

![RLBench long-horizon results](./robostream_assets/RLBench_Long_Tab.png)

真实世界实验更能体现这类方法的价值。积木操作中会出现遮挡、接触误差、视角变化和几何关系变化，模型不仅要看懂当前画面，还要记住前面动作已经把场景改成了什么样。

在真实世界 block manipulation 任务中，RoboStream 达到 **44.4%** 成功率，而 SoFar 和 VoxPoser 为 **11.1%**。

![Real-world manipulation results](./robostream_assets/real_world_res.png)

换句话说，RoboStream 的优势不只是 benchmark 数字更高，而是在更接近真实机器人执行的设置中，减少了长程任务最常见的“前面忘了，后面错了”。

## 空间推理与泛化：不只是会做任务，还要理解关系

除了长程执行，RoboStream 也在 SimplerEnv、空间推理和 Open6DOR 相关评估中展示了更稳定的表现。

这些测试并不只考察模型能否给出动作，还考察它是否理解物体位置、目标约束、空间关系和任务状态。对机器人来说，这些能力往往比单次识别更接近真实可用性。

![SimplerEnv results](./robostream_assets/SimplerEnv_Tab.png)

![Spatial reasoning and Open6DOR results](./robostream_assets/VQA_Tab.png)

## 定性结果：遮挡、移动、恢复，都是记忆的考场

从定性结果看，RoboStream 在需要持续追踪目标、处理遮挡、恢复空间关系的场景中更稳。

![Qualitative comparison](./robostream_assets/Qualitative_results.png)

这类任务对 VLM 的挑战并不只是“图里有什么”，而是“之前发生过什么，以及现在应该如何接着做”。当对象短暂不可见、关系被改变、任务目标需要回溯时，结构化记忆的价值就会被放大。

## 消融实验：STF-Tokens 和 CSTG 是一组配合

消融实验显示，RoboStream 的收益并不是来自单个技巧，而是来自对象级记忆和因果图记忆的协同。

STF-Tokens 让对象在三维空间中保持可追踪；CSTG 让动作造成的变化被记录和继承。前者解决“物体是谁、在哪里”，后者解决“动作改变了什么、下一步应当基于哪个状态继续”。

![Ablation results](./robostream_assets/Ablation_Tab.png)

这也是 RoboStream 与许多单步 VLM planner 的关键区别：它不只是在每一步问模型“现在该做什么”，而是在不断维护一个随交互演化的任务状态。

## 为什么这件事重要

机器人走向开放环境后，能力瓶颈不会只来自视觉识别，也不会只来自语言理解。更大的挑战是：它能否在连续交互中维护一个稳定、可更新、可推理的世界模型。

RoboStream 给出的信号很清楚：如果希望 VLM 真正进入机器人长程任务，就不能只把它当作单帧图像问答器。它需要记忆，需要几何锚定，需要知道动作如何改变世界。

从这个角度看，RoboStream 不是简单地把 VLM 接到机器人上，而是在补上 VLM 做机器人时最缺的一环：**持续的空间-时间记忆**。

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
