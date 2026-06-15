# 🔥 Socratic Learn — Go2W Step 3 被拷问

<p align="center">
  <img src="https://img.shields.io/badge/Topic-Go2W%20Locomotion-blue" alt="Topic">
  <img src="https://img.shields.io/badge/Method-Socratic%20Learn-orange" alt="Method">
  <img src="https://img.shields.io/badge/Step-3%2F3%20被拷问-red" alt="Step 3/3">
</p>

---

## 📌 第三步：被拷问

> 🔑 基于前两步的骨架和争议，出 10 道题。每题不是考"记没记住"，而是考"会不会跨模块推理"。

---

### 题 1 — Cross-Connect

> `commands` 在 Go2W 中同时进入 observation、reward 和 curriculum 三条路径。请分别指出在哪个 cfg 的哪个字段配置的。如果部署时 `commands_scale` 和训练不一致（比如训练时 lin_vel_x 在 obs 中除以 1.0，部署时除以 2.0），策略的行为会怎么变？

<details>
<summary>点击展开参考思路</summary>

三条路径：
- Observation: `velocity_env_cfg.py:156-161` — `ObsTerm(func=mdp.generated_commands, params={"command_name": "base_velocity"})` — 4 维拼入 policy obs
- Reward: `rough_env_cfg.py:191-192` — `track_lin_vel_xy_exp(weight=3.0)` 和 `track_ang_vel_z_exp(weight=1.5)` 比较当前速度和 command
- Curriculum: `velocity_env_cfg.py:671` — `terrain_levels_vel` 用 tracking reward 判断升降级

部署时 commands_scale 不一致：策略收到的命令被缩放。比如部署端除以 2.0 → 策略以为目标速度是你给的一半。结果为"你让它走 1m/s，它以为你让它走 0.5m/s，实际走 0.5m/s"。表现是"机器人走太慢"，但动作本身可能看起来正常——因为 obs 维度对、网络算对了，只是输入语义被缩放。
</details>

---

### 题 2 — Counterfactual

> Go2W 子类把父类的 `illegal_contact` termination 关闭了（`rough_env_cfg.py:223`）。如果重新打开，body_names 设为 `[base_link_name, ".*_hip"]`，你的训练（1775 轮，time_out=1.0）会发生什么？这个设计选择是偷懒还是合理？

<details>
<summary>点击展开参考思路</summary>

time_out=1.0 意味着当前没有 episode 因接触终止。重新打开后：
- 正常行走：base 和 hip 可能偶尔擦地（尤其粗糙地形 + 轮足高度低）→ time_out 降到 0.95-0.99
- 训练初期（terrain 低级时）：可能频繁触发，episode 变短

这是合理的设计选择，不是偷懒。Go2W 的轮子直径增加了机身离地高度，但同时轮足结构在颠簸时 hip/thigh 擦地是物理必然——和四足的"身体触地=摔倒"逻辑不同。把这种接触判为终止会制造大量假阳性，打断学习。
</details>

---

### 题 3 — Diagnosis

> 你的训练中 `terrain_levels = 5.18` 已停滞。`terrain_levels_vel` 函数的升级条件是 tracking reward 超过阈值。如果 Go2W 的轮足结构导致它的 tracking 天然比纯四足差（轮子打滑、非平整地形轮子悬空），threshold 是按四足调的——你怎么办？

<details>
<summary>点击展开参考思路</summary>

三步：
1. 确认是阈值问题还是能力问题：在固定 level 6 地形上测 tracking error，对比 level 5。如果 level 6 显著更差 → 策略确实没准备好
2. 如果 level 5 和 6 的 tracking 差异小（<5%），但都不达标 → 可能是阈值对 Go2W 太高，降低 upgrade threshold
3. 如果 tracking error 的方差大（某些地形类型特别差），问题在 terrain 采样而非阈值——需要按地形类型分桶评估

核心原则：先确认策略真的"不能"还是阈值"不准"，再动手。不要假设 level 9 应该是所有机器人的目标——轮足可能 level 5-6 就是实用上限。
</details>

---

### 题 4 — Cross-Connect

> `joint_pos_rel_without_wheel`（`rough_env_cfg.py:82`）和普通 `joint_pos_rel` 的区别是什么？如果部署端误用了普通 `joint_pos_rel`（包含轮子绝对角度），轮角跑到 500 rad 时，网络会发生什么？

<details>
<summary>点击展开参考思路</summary>

`joint_pos_rel` = 当前角度 − 默认角度。轮子持续滚动 → 轮角累积到几百弧度 → relative pos 变成无界增长的值 → 输入神经网络的数值从训练时分布（[-π, π] 或更小）偏离到 500+ → 激活值爆炸。

神经网络训练时没见过 500 这种输入，ELU/线性层的输出会脱离正常范围，最终 14 维 action 全部乱掉。不止轮子——腿也会因为共享隐藏层而受影响。

这就是"维度对但语义错"最危险的场景：部署端完全可能拼出"正确维度"的 obs（16 个 float），但其中 4 个 float 的语义不是网络训练时以为的语义。
</details>

---

### 题 5 — Counterfactual

> 如果把 `num_envs` 从 4096 减到 512，其他参数不变。PPO 每轮只有 512×24=12,288 transitions（vs 98,304）。对训练速度和 policy 质量各有什么影响？Go2W 的混合 action space 会让这个变化更严重还是更不严重？

<details>
<summary>点击展开参考思路</summary>

速度：每轮时间缩短（~2.15s → ~0.3-0.5s），但到收敛需要的轮数可能翻倍。总 wall-clock 时间不一定更短。

质量：小 batch → advantage 估计方差更大，PPO 更新更噪。14 维混合 action（位置+速度尺度不同）让方差问题更严重——小 batch 中轮子速度 action 的高 variance 可能主导梯度方向。

Go2W 让这个变化**更严重**：混合 action space 的 heteroscedastic 特性（不同维度天然有不同的方差）需要大 batch 来稳定梯度估计。纯四足（12 维同一类型）对小 batch 的容忍度更高。
</details>

---

### 题 6 — Diagnosis

> 你的 `Metrics/base_velocity/error_vel_xy = 0.9 m/s`。命令范围 [-1, 1] m/s。这意味着什么？如果 flatten 到平地后 error 降到 0.3 m/s，问题出在哪？

<details>
<summary>点击展开参考思路</summary>

0.9 m/s 误差在 ±1.0 命令范围上：如果命令 1.0 m/s，实际速度在 0.1-1.9 m/s 之间波动。策略大致知道方向，但精确度差。

如果平地 error=0.3，粗糙 terrain error=0.9 → 问题在地形，不在策略。可能：
- 轮子在非平整地形上打滑或悬空
- 策略在粗糙地形上把更多"注意力"放在保持平衡而非精确跟踪
- terrain_levels=5 的地形扰动已经显著影响速度

进一步验证：分 terrain_level 统计 error。如果 level 1-3 的 error < 0.3，level 4-5 的 error > 0.8，说明跟踪精度和地形难度强相关。这不是 bug，是 tradeoff。
</details>

---

### 题 7 — Counterfactual

> PPO critic 训练时 `enable_corruption=False`（无观测噪声），actor 训练时 `enable_corruption=True`。如果给 actor 也关闭噪声，仿真中 tracking 会变好还是变差？部署到真机会怎样？

<details>
<summary>点击展开参考思路</summary>

仿真中 tracking 可能短期变好（确定性更高，学习更快），但部署时：
- 真机 IMU 有 ~5ms 延迟 + 高频振动 → `projected_gravity` 和 `ang_vel` 含噪声
- 关节编码器有量化误差 → `joint_pos` / `joint_vel` 含噪声
- 速度估计（如果有）更差

策略在训练时没见过 noisy obs → 碰到噪声可能输出异常动作。当前设计（actor noisy, critic clean）是正确的：critic 只参与训练，需要准确评估价值；actor 需要鲁棒。

这和"privileged obs 可以给 critic 但不能给 actor"是同一逻辑——训练端和部署端的信息不对称。
</details>

---

### 题 8 — Cross-Connect

> Go2W 的 16 个关节中，4 个是轮子。`joint_mirror` 的配对联是 `["FR_(hip|thigh|calf).*" ↔ "RL_(hip|thigh|calf).*"]` 和 `["FL_(hip|thigh|calf).*" ↔ "RR_(hip|thigh|calf).*"]`。为什么轮子不参与 mirror？如果强制把 FR_foot 和 RL_foot 加入配对联，会制造什么错误学习信号？

<details>
<summary>点击展开参考思路</summary>

轮子不参与 mirror 是正确的。轮子的任务是产生前进/转向速度，不是维持姿态对称。转弯时左右轮天然需要不同速度（差速转向）——如果 mirror 惩罚这种差异，等于告诉策略"不要差速转弯"。

强制加 mirror → 策略在转弯时被惩罚 → 可能学出"为了对称牺牲转向"：用腿转向（不对称步态）来补偿轮子不能差速的限制 → 动作复杂化、tracking 变差。

mirror 的正确使用域是"硬件对称且任务对称的关节"。腿满足这个条件（左右各一套，对称步态是合理的），轮子不满足（任务需要非对称运动）。
</details>

---

### 题 9 — Diagnosis

> 你的 `Episode_Reward/upward = 3.88`（权重 1.0），`track_lin_vel_xy_exp = 1.62`（权重 3.0）。upward 占正奖励的 ~62%，且已接近天花板。后续训练中 upward 无法再提供有区分度的信号。这时如果不调整 reward 权重，策略还能继续提升速度跟踪吗？为什么？

<details>
<summary>点击展开参考思路</summary>

能，但动力不足。原因：

1. upward 已经饱和——每个 episode 都是满分 → 梯度信号中来自 upward 的分量接近零 → 策略更新主要由 tracking 项驱动
2. 但 tracking 的提升空间被 `action_rate_l2(-0.68)` 和 `joint_pos_penalty(-0.69)` 夹在中间：策略想做更大动作来跟踪速度，但被惩罚项拉住
3. 这种情况下，PPO 的梯度是 tracking 正信号和 penalty 负信号的拉扯——可能陷入局部最优：动作不够大 → tracking 不够好 → 但动作再大惩罚就更重

如果不调 reward，策略可能仍然缓慢提升但效率很低。更有效的做法是：在确认姿态稳定后，逐步降 `upward` 权重或改为稀疏（只在翻车时惩罚），释放优化空间给 tracking。
</details>

---

### 题 10 — Counterfactual

> 如果要把 Go2W 从"崎岖地形速度跟踪"迁移到"仓库平地物流搬运"（车身加 5kg 负载，要求平稳不洒），五个骨架模型哪些能复用，哪些必须改？

<details>
<summary>点击展开参考思路</summary>

| 模型 | 可复用？ | 原因 |
|:---|:---:|:---|
| 混合运动 MDP | ✅ 复用 | 腿+轮控制接口不变 |
| 命令条件化 | ⚠️ 调参 | 速度范围可能缩小（物流不需要 1m/s 全速），转向更平缓 |
| 奖励塑形 | ⚠️ 加项 | 需启用 `body_lin_acc_l2`（当前权重=0）抑制急加速；可能降 `action_rate_l2` 权重以允许更保守的动作 |
| PPO 训练 | ✅ 复用 | 网络和超参可复用 |
| Sim-to-Real | ⚠️ 部分 | Events 里质量随机化要包含 +5kg；平地场景不需要 terrain curriculum |

最大改动在模型三（奖励塑形）——从"崎岖地形上的速度跟踪"变为"平地 + 负载 + 防洒"，reward 需要新增平稳度约束。模型一、四直接复用。
</details>

---

## 📊 诊断报告

| 题型 | 题号 | 考查能力 |
|:---|:---|:---|
| **Cross-Connect** | 1, 4, 8 | 能否把 obs/action/reward/curriculum/deploy 串成系统 |
| **Counterfactual** | 2, 5, 7, 10 | 能否推理"改掉 X 会怎样" |
| **Diagnosis** | 3, 6, 9 | 能否从数字定位根因并设计实验 |

### 评分标准

- ✅ **答对且能指出源码位置和参数值** → 该概念已内化
- ⚠️ **结论对但理由模糊** → 追问"为什么你认为是这样？"
- ❌ **答错或给模糊方向** → 回到对应骨架模型重读

---

## 复盘报告

| 模块 | 你的结论 |
|------|----------|
| 我真正懂的诊断场景 | |
| 我还会漏查的断点 | |
| 我能立刻定位的代码文件 | |
| 我会设计的第一个对照实验 | |
| 下一步要在 Go2W 上跑的验证 | |

---

## ❓ 开始

从第 1 题开始，一题一题来。

> `commands` 在 Go2W 中同时进入 observation、reward 和 curriculum 三条路径。请分别指出在哪个 cfg 的哪个字段配置的。如果部署时 `commands_scale` 和训练不一致，策略的行为会怎么变？
