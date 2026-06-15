# 🔥 Socratic Learn — Go1 Step 3 被拷问

<p align="center">
  <img src="https://img.shields.io/badge/Topic-Go1%20Walk%20These%20Ways-blue" alt="Topic">
  <img src="https://img.shields.io/badge/Method-Socratic%20Learn-orange" alt="Method">
  <img src="https://img.shields.io/badge/Step-3%2F3%20被拷问-red" alt="Step 3/3">
</p>

---

## 📌 第三步：被拷问（Be Cross-Examined）

> 🔑 基于前两步的骨架和争议，出 10 道题。  
> 每题不是考“记没记住”，而是考“会不会跨模块推理”。

---

### 题 1 — Cross-Connect

> `commands` 在这个项目里不只是“遥控器速度指令”。它同时进入 observation、curriculum 和 reward。请解释这三条路径分别在哪里体现；如果 command 语义在真机端和仿真端不一致，会发生什么？

<details>
<summary>点击展开参考思路</summary>

- observation：`compute_observations()` / `LCMAgent.get_obs()` 都会把 commands 乘 scale 后拼进 obs
- curriculum：`_init_command_distribution()` 和 `_resample_commands()` 根据 command bins / gait categories 采样和扩展难度
- reward：速度跟踪、接触时序、足端高度、姿态控制等都依赖 commands
- 真机端语义不一致时，策略会“以为自己在执行另一个任务”：速度、步态相位、摆腿高度、身体姿态都可能错

</details>

---

### 题 2 — Counterfactual

> 如果把 `Cfg.env.num_observation_history` 从 30 改成 1，但其他网络结构不改，最先会伤害哪个能力：速度跟踪、步态节律、还是 Sim2Real 适应？为什么？

<details>
<summary>点击展开参考思路</summary>

最先伤害的是 **Sim2Real 适应**。

`adaptation_module` 的输入是 `obs_history`，它要从一段时间的响应中推断摩擦、延迟、电机强度、restitution 等隐变量。单帧 observation 只能告诉你当前姿态和关节状态，很难区分“低摩擦”“电机弱”“状态估计滞后”这些不同原因。

步态节律也会受影响，但 clock inputs 仍提供相位信息；速度跟踪有 command 和当前状态支撑。真正被砍掉根基的是 history → latent。

</details>

---

### 题 3 — Diagnosis

> 训练时 `act_teacher(obs_history, privileged_obs)` 表现很好，但 `act_student(obs_history)` 很差。你会优先怀疑 reward、PPO、actor_body，还是 adaptation_module？为什么？怎么验证？

<details>
<summary>点击展开参考思路</summary>

优先怀疑 **adaptation_module / history 可观测性**。

因为 teacher 和 student 共享 `actor_body`，差别在 latent 来源：

- teacher：直接用 `privileged_obs`
- student：用 `adaptation_module(obs_history)` 预测 latent

如果 teacher 好，说明 reward、PPO 主体和 actor_body 至少能产生好策略。student 差，说明从 history 到 privileged latent 的映射没学好，或 history 本身不够区分隐藏参数。

验证：看 `mean_adaptation_module_loss` 和 teacher/student 分别 rollout；再做 ablation：给 student 更长 history、更多 privileged obs target，或缩小 domain randomization 范围。

</details>

---

### 题 4 — Counterfactual

> 如果把 `control_type` 从 `actuator_net` 改成普通 `P` 控制，同时不重新训练策略，只拿旧 checkpoint 去 play，会发生什么？为什么？

<details>
<summary>点击展开参考思路</summary>

大概率性能明显变差，甚至动作不稳定。

原因是策略训练时面对的是 `actuator_net` 产生的 torque 响应。`actuator_net` 输入 joint error / velocity 的历史，用 TorchScript 模型拟合 Unitree Go1 电机动态。普通 P 控制的力矩响应不同，相当于换了 action-to-torque dynamics。

这不是一个“底层实现细节”改动，而是改变了环境动力学。策略输出同样的 joint target，身体得到的加速度和接触结果会变。

</details>

---

### 题 5 — Diagnosis

> 真机部署时，observation 维度完全正确，但机器人一启动 gait 就乱。列出至少 5 个“维度正确但语义错误”的可能原因。

<details>
<summary>点击展开参考思路</summary>

常见原因：

1. 关节顺序和训练时 DOF 顺序不一致
2. `default_dof_pos` 不一致
3. `commands_scale` 或 `obs_scales` 不一致
4. `clock_inputs` 的相位更新和训练不一致
5. `action_scale` / `hip_scale_reduction` 不一致
6. `control_dt` 不一致，导致 gait phase 积分速度错
7. IMU gravity vector 坐标系和仿真不同
8. last action / last last action 历史没有正确初始化

维度只检查 shape，不检查每一维代表什么。

</details>

---

### 题 6 — Cross-Connect

> 为什么部署只导出 `adaptation_module_latest.jit` 和 `body_latest.jit`，不导出 `critic_body`？这和 PPO 的训练/推理分工有什么关系？

<details>
<summary>点击展开参考思路</summary>

critic 只在训练时估计 value，用于计算 advantage / returns，帮助 PPO 更新策略。部署时不需要估计“这个状态值多少钱”，只需要根据当前 obs_history 选择 action。

部署路径是：

```text
obs_history -> adaptation_module -> latent
[obs_history, latent] -> actor_body -> action
```

`critic_body` 的输入还包含 privileged observation，这在真机上本来就没有。强行部署 critic 说明混淆了训练辅助网络和推理策略网络。

</details>

---

### 题 7 — Counterfactual

> 如果去掉 `tracking_contacts_shaped_force` 和 `tracking_contacts_shaped_vel`，只保留速度跟踪和平滑奖励，策略还能跑得快。那为什么它可能更难稳定部署到真机？

<details>
<summary>点击展开参考思路</summary>

速度对了，不代表接触模式对了。

接触奖励把 gait command 和足端接触/摆动速度绑起来：该支撑时脚要慢并承力，该摆动时脚要离开地面。不加这类约束，策略可能通过滑步、拖脚、异常弹跳等方式拿到速度奖励。

仿真里的接触容错高，真机上这种“速度正确但接触脏”的 gait 会带来打滑、冲击、大扭矩尖峰和跌倒风险。

</details>

---

### 题 8 — Diagnosis

> 训练曲线里总 reward 上升，但分步态评估发现 trot 变好、pace 和 bound 变差。你如何判断这是共享网络干扰、课程采样偏置，还是某个 reward 对 gait 不公平？

<details>
<summary>点击展开参考思路</summary>

排查顺序：

1. 看 `curriculum/distribution`：pace/bound 是否采样少，或长期停在低难度 bin
2. 分 gait 看 `tracking_contacts_shaped_force/vel`、速度误差和 fall rate
3. 固定 command 做 rollout，排除课程分布影响
4. 单独训练某个 gait，对比是否能学好
5. 看 reward 是否隐含偏向 trot，例如 foot phase / offset 设置是否让 trot 更容易拿分

平均 reward 上升可能只是 trot 样本多或更容易拿高分。

</details>

---

### 题 9 — Cross-Connect + Diagnosis

> 真机上出现大扭矩尖峰。如何区分是策略 action 跳变、action→joint target 映射错误，还是状态估计/LCM 延迟导致的 PD 误差突然变大？

<details>
<summary>点击展开参考思路</summary>

看日志指纹：

- action 跳变：`action` 自身有尖峰，`joint_pos_target` 同步跳
- 映射错误：action 平滑，但 `joint_pos_target = action*scale + default` 后异常，常见于 `action_scale`、`hip_scale_reduction`、关节顺序错
- 延迟/状态估计：action 和 target 平滑，但 `dof_pos/dof_vel` 突然滞后或跳变，PD error 放大；LCM timestamp 或 control loop frequency 会异常

只看 action 不够，因为 torque 由 target、当前 joint state、PD/actuator dynamics 一起决定。

</details>

---

### 题 10 — Counterfactual

> 如果要把这个项目从“Go1 多步态行走”迁移到“Go1 跳跃越障”，五个骨架模型里哪些能复用，哪些必须重做？

<details>
<summary>点击展开参考思路</summary>

| 模型 | 可复用？ | 原因 |
|:---|:---:|:---|
| RL 环境契约 | ⚠️ 部分复用 | step/reset/obs 框架可复用，但 termination 和 command 要改 |
| 命令条件化 | ⚠️ 部分复用 | 速度/gait command 不够，需要障碍高度、起跳时机、落地目标 |
| 奖励塑形 | ❌ 大量重做 | 需要起跳、空中姿态、落地冲击、越障成功奖励 |
| PPO_CSE/RMA | ✅ 大体复用 | history adaptation 和 PPO 框架仍有用 |
| Sim2Real 部署 | ⚠️ 部分复用 | JIT/LCM 可复用，但落地冲击和安全层要重新设计 |

最大工作量通常在 reward/command，而不是 PPO 本身。

</details>

---

## 📊 诊断报告

全部 10 题做完后，按以下维度评估：

| 题型 | 题号 | 考查能力 |
|:---|:---|:---|
| **Cross-Connect** | 1, 6 | 能否把源码模块连成系统 |
| **Counterfactual** | 2, 4, 7, 10 | 能否推理“改掉 X 会怎样” |
| **Diagnosis** | 3, 5, 8, 9 | 能否从现象定位根因 |

### 评分标准

- ✅ **答对且能指出源码证据** → 该概念已内化
- ⚠️ **结论对但理由模糊** → 追问“为什么你认为是这样？”
- ❌ **答错或只会说 sim2real gap** → 回到对应骨架模型重读

---

## ❓ 开始

从第 1 题开始，一题一题来。  

> `commands` 在这个项目里不只是“遥控器速度指令”。它同时进入 observation、curriculum 和 reward。请解释这三条路径分别在哪里体现；如果 command 语义在真机端和仿真端不一致，会发生什么？

