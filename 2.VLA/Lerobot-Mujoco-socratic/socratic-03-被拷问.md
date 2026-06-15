# Socratic Learn — Step 3 被拷问

<p align="center">
  <img src="https://img.shields.io/badge/Topic-LeRobot%20MuJoCo-blue" alt="Topic">
  <img src="https://img.shields.io/badge/Method-Socratic%20Learn-orange" alt="Method">
  <img src="https://img.shields.io/badge/Step-3%2F3%20被拷问-red" alt="Step 3/3">
  <img src="https://img.shields.io/badge/Date-2026--06--11-lightgrey" alt="Date">
</p>

---

## 第三步：被拷问（Be Cross-Examined）

> 基于前两步的骨架和争议，出 10 道题。每题不是考“记没记住”，而是考“会不会用”。

### 使用说明

这个文件保留“题目 + 折叠答案”的格式，适合自测。更接近苏格拉底用法的方式是：先只看题目，自己写出推理，再展开答案对照。答案里的结论分两类：

- **代码事实：** 直接由仓库文件支持。
- **工程推断：** 基于代码事实和常见模仿学习问题推导，可能需要实验验证。

---

### 题 1 — Cross-Connect

> `1.collect_data.ipynb` 里，人按键盘得到的是末端位姿增量，但数据集里的 `action` 保存的是 `joint_q`。那么 ACT 训练时到底在学什么映射？为什么部署时用 `action_type='joint_angle'` 是合理的？

<details>
<summary>点击展开</summary>

ACT 学的不是“键盘按法”，而是：

```text
图像 + 状态  ->  6 个关节目标 + 夹爪
```

采集时链路是：

```text
teleop_robot() 产生末端增量
step(action) 通过 IK 解出关节角
dataset 保存 IK 后的 joint_q
```

所以部署时如果策略输出的是训练集中同一种 `joint_q`，环境就应该用 `joint_angle` 来解释它。这里的关键不是变量名叫不叫 `action`，而是这个字段的物理语义必须和部署环境一致。

**依据：** `teleop_robot()` 生成末端增量；`SimpleEnv.step()` 根据 `action_type` 做 IK 或直接解释关节角；`1.collect_data.ipynb` 中写帧逻辑把 `joint_q` 保存为 `"action"`；`4.deploy.ipynb` 中部署环境使用 `action_type='joint_angle'`。

**不确定点：** notebook 中 cell 可能被用户改过；判断以当前仓库文件为准。

</details>

---

### 题 2 — Counterfactual

> 如果训练数据里的 `action` 是关节角，但部署时误把环境设成 `action_type='eef_pose'`，会发生什么？为什么这类错误不一定马上报 shape 错？

<details>
<summary>点击展开</summary>

最可能结果是机器人动作完全异常。

原因是两种 action 都可能是 7 维：

- `eef_pose`: `dx dy dz droll dpitch dyaw gripper`
- `joint_angle`: `j1 j2 j3 j4 j5 j6 gripper`

shape 一样，但语义完全不同。一个关节角数值例如 `1.2` rad，如果被当成末端 `dx=1.2m`，就是离谱的大位移；一个末端小增量 `0.007m`，如果被当成关节角，又会让关节目标几乎不动。

这类 bug 的危险点在于：tensor 维度都对，模型也能 forward，只有闭环行为会暴露问题。

**依据：** `SimpleEnv.step()` 里 `eef_pose` 分支把 `action[:3]` 加到末端位置，把 `action[3:6]` 当 RPY 增量；`joint_angle` 分支直接把 `action[:-1]` 当关节角。

**不确定点：** 具体失败形态取决于动作数值范围、IK 是否收敛、MuJoCo 关节限位和夹爪接触状态。

</details>

---

### 题 3 — Diagnosis

> 训练 loss 很低，离线 mean action error 也很低，但 rollout 时机械臂一碰到杯子就失败。你按优先级诊断三个可能原因。

<details>
<summary>点击展开</summary>

优先排查：

1. **闭环分布偏移**：训练时模型只见过专家状态；部署时自己的小误差导致夹爪接触点偏了，下一帧状态离开专家轨迹，误差继续扩大。
2. **动作/状态契约错位**：训练里的 `observation.state` 可能是末端位姿，部署时却给了关节状态，或者 action 的 `joint_angle/eef_pose` 语义不一致。
3. **接触阶段数据不足**：抓取前的接近动作容易拟合，真正难的是接触、夹紧、提起。少量 demo 可能没有覆盖足够多的接触扰动。

离线误差低只说明“在专家轨迹附近预测得像专家”，不说明“偏离专家轨迹后能自己恢复”。

**依据：** 仓库中有离线训练和离线 action error 检查，也有部署 rollout；二者评估的是不同分布。

**不确定点：** 如果实际 demo 覆盖很密、初始状态随机化足够、任务短且接触稳定，闭环分布偏移可能不是主要原因。需要看失败视频和每步 action/state 才能排序根因。

</details>

---

### 题 4 — Cross-Connect

> `obj_init` 在 README 里写着不用于训练。既然不训练，为什么还要保存？它和数据回放、评估、debug 有什么关系？

<details>
<summary>点击展开</summary>

`obj_init` 是可复现实验的锚点。

它至少有三个作用：

1. **回放重建**：同一条 episode 要在 MuJoCo 里复现，必须知道 mug 和 plate 初始位置。
2. **失败归因**：如果某些初始位置总失败，可以按 `obj_init` 分析是不是数据覆盖不够。
3. **评估对齐**：离线数据中的动作序列只有放回相同初始物体布局，才有合理的物理含义。

它不一定直接喂给 policy，但它支撑了 dataset replay 和诊断。

**依据：** README 和数据 schema 都保存 `obj_init`；环境提供 `set_obj_pose()` 用于设置物体位置。普通任务里 `obj_init` 是 mug + plate，语言任务里是 red mug + blue mug + plate。

**不确定点：** 具体 replay notebook 是否完整使用 `obj_init`，要以对应 cell 为准；这里强调的是它在实验复现中的作用。

</details>

---

### 题 5 — Counterfactual

> 语言环境只支持 red 和 blue 两个目标。如果你在推理时给任务文本 `"Place the green mug on the plate."`，可能会发生什么？这暴露了语言条件策略的哪个边界？

<details>
<summary>点击展开</summary>

有两层结果：

1. 环境层：`SimpleEnv2.set_instruction()` 只检查文本里有没有 `red` 或 `blue`。如果直接用它设置 `green`，会抛出 `ValueError`，因为无法绑定 `obj_target`。
2. 策略层：pi0/SmolVLA 也许从预训练里知道 green 是颜色，但本项目的数据和环境没有 green mug 的目标绑定，也没有对应成功条件。

这说明“语言模型懂词”不等于“机器人环境有这个可执行目标”。语言必须落到环境对象、视觉实例和成功条件上，才是机器人任务。

**依据：** `SimpleEnv2.set_instruction()` 只把包含 `red` 的指令绑定到 `body_obj_mug_5`，把包含 `blue` 的指令绑定到 `body_obj_mug_6`，否则抛错；`check_success()` 检查的是 `self.obj_target`。

**不确定点：** 如果绕过 `set_instruction()` 直接给 policy 传 green 文本，策略可能输出某种动作，但环境没有 green 目标的成功定义，不能算这个任务被支持。

</details>

---

### 题 6 — Diagnosis

> 在离线环境运行 pi0 或 SmolVLA 时，模型加载阶段卡在 Hugging Face 下载。结合 `train_model.py` 和 README，应该查哪些地方？

<details>
<summary>点击展开</summary>

优先查：

1. **本地预训练路径是否存在**：`train_model.py` 里把 pi0 的 `pretrained_path` 设成 `lerobot/pi0`，SmolVLA 设成 `lerobot/smolvla_base`。这些目录必须真的有模型文件。
2. **notebook 中 `from_pretrained()` 是否仍指向远程名**：`7.pi0.ipynb` 和 `8.smolvla.ipynb` 里有本地路径示例，例如 `./omy_pnp_pi0`、`./omy_pnp_smolvla`。
3. **tokenizer / processor 是否也走本地**：VLA 不只是权重，还可能加载 tokenizer、processor、config。只下载 checkpoint 不够。

本质是：VLA 的“模型”不是一个 `.pt` 文件，而是一组权重、配置、tokenizer 和视觉语言组件。

**依据：** `train_model.py` 在 pi0 和 SmolVLA 分支中设置本地 `pretrained_path`；README 明确提醒离线使用时要确保模型文件在本地；pi0/SmolVLA notebooks 也展示了本地 `from_pretrained()` 路径。

**不确定点：** 卡下载也可能来自 dataset、processor、tokenizer 或其他依赖，不一定只来自 policy 权重。需要看具体日志 URL。

</details>

---

### 题 7 — Cross-Connect

> ACT 训练常用 `chunk_size=10`，pi0/SmolVLA 配置里是 `chunk_size=5, n_action_steps=5`。action chunking 对闭环控制有什么好处？又会引入什么风险？

<details>
<summary>点击展开</summary>

好处：

- 一次预测多个未来动作，轨迹更平滑。
- 降低逐帧视觉噪声导致的动作抖动。
- 对短时遮挡或视觉波动更鲁棒。

风险：

- chunk 太长会引入延迟：环境已经变了，策略还在执行旧计划。
- 接触任务里，杯子被碰动后需要马上修正；长 chunk 可能继续执行错误动作。
- `n_action_steps` 和控制频率绑定，如果 20Hz 下执行 10 步，就是 0.5 秒的开环片段。

所以 chunking 是平滑性和反应速度的交换。

**依据：** ACT 训练 notebook 使用 `chunk_size=10, n_action_steps=10`，ACT 部署中设置 `chunk_size=10, n_action_steps=1, temporal_ensemble_coeff=0.9`；pi0/SmolVLA 配置使用 `chunk_size=5, n_action_steps=5`。

**不确定点：** 最佳 chunk 长度依赖控制频率、任务时长、接触敏感度和模型延迟，不能从仓库配置直接推出最优值。

</details>

---

### 题 8 — Counterfactual

> 如果训练时 `observation.state` 是末端位姿 `[x,y,z,r,p,y]`，部署时却喂入 6 个关节角，shape 仍然是 `(6,)`。模型会怎样？为什么这比维度错误更难发现？

<details>
<summary>点击展开</summary>

模型会把关节角当成末端位姿来理解，行为大概率错乱。

难发现是因为：

- shape 一样，DataLoader 和 policy forward 都不会报错。
- normalization 也可能照常执行，只是统计分布不匹配。
- 错误只体现在控制行为上：比如模型以为末端在桌面上方，实际输入却是关节 2 的角度。

这就是数据契约最核心的警告：字段名和 shape 不够，语义也必须一致。

**依据：** 普通采集中 `observation.state` 来自 `ee_pose`；语言采集中 `observation.state` 来自 `joint_q[:6]`。两者 shape 都是 6，但语义不同。

**不确定点：** 如果分别训练、分别部署且 stats 与输入一致，这不是问题；问题只在混用数据集、checkpoint 或部署构造观测时出现。

</details>

---

### 题 9 — Diagnosis

> 数据回放看起来完全正确，说明 episode 可以复现；但策略 rollout 不稳定。为什么“回放正确”不能证明“策略正确”？

<details>
<summary>点击展开</summary>

数据回放通常执行的是记录好的动作序列：

```text
初始场景 + 专家动作 -> 专家轨迹
```

策略 rollout 执行的是模型预测动作：

```text
当前观测 -> 模型动作 -> 新观测 -> 模型动作 -> ...
```

回放正确只能证明：

- 初始物体位置能恢复。
- 动作日志能驱动环境。
- 图像和 metadata 没有明显损坏。

它不能证明模型在每一帧都能选对动作，也不能证明模型偏离轨迹后能恢复。回放是数据完整性测试，rollout 才是控制能力测试。

**依据：** dataset replay 使用记录动作和初始条件；policy rollout 使用 `policy.select_action(data)` 的预测动作闭环推进环境。两者的因果链不同。

**不确定点：** 如果 replay notebook 同时评估 policy，那就不再是纯数据回放；需要区分“回放专家动作”和“回放模型动作”。

</details>

---

### 题 10 — Cross-Connect + Counterfactual

> 如果要把普通单 mug 任务升级成“按语言指令把红/蓝 mug 放到 plate”，五个骨架模型里哪些必须改？哪些可以复用？

<details>
<summary>点击展开</summary>

| 模型 | 是否要改 | 原因 |
|:--|:--|:--|
| 模型一：MuJoCo 任务世界 | 必须改 | 场景要有两个 mug，成功条件要根据 `obj_target` 判断 |
| 模型二：控制接口与 IK | 基本复用 | 机械臂控制方式不变，仍然是末端/关节桥接 |
| 模型三：LeRobot 数据契约 | 必须改 | `task` 文本变重要，`obj_init` 从 6 变 9，state/action 语义要重新确认 |
| 模型四：策略家族 | 可能要换 | ACT 可以训练，但 pi0/SmolVLA 更适合用语言区分目标 |
| 模型五：闭环部署与评估 | 必须改 | success 不再只看固定 mug，而要看指令对应的 red 或 blue mug |

核心变化不是“多一个杯子”这么简单，而是从固定目标任务变成了语言到对象的绑定问题。

**依据：** `SimpleEnv2` 增加了 `set_instruction()`、`obj_target` 和基于目标 mug 的 `check_success()`；语言采集把 `task = PnPEnv.instruction` 写入数据集；pi0/SmolVLA 推理时向 policy 输入 `task`。

**不确定点：** ACT 也能在固定文本或 one-hot 目标上训练，但当前仓库里语言条件主线更偏向 pi0/SmolVLA。

</details>

---

## 诊断报告模板

全部 10 题做完后，按以下维度评估：

| 题型 | 题号 | 考查能力 |
|:--|:--|:--|
| **Cross-Connect** | 1, 4, 7, 10 | 能否把控制接口、数据契约、策略和部署连起来 |
| **Counterfactual** | 2, 5, 8 | 能否推理“如果语义错位，会怎样” |
| **Diagnosis** | 3, 6, 9 | 能否从现象定位根因 |

### 评分标准

- **答对且逻辑清晰**：该概念已经内化。
- **答对但理由模糊**：停留在表面记忆，继续追问“为什么你认为是这样？”
- **答错或漏洞明显**：回到相关骨架模型重读，尤其是数据契约和部署闭环。

---

## 开始

从第 1 题开始，一题一题来。**你准备好了就回答第 1 题。**

> `1.collect_data.ipynb` 里，人按键盘得到的是末端位姿增量，但数据集里的 `action` 保存的是 `joint_q`。那么 ACT 训练时到底在学什么映射？为什么部署时用 `action_type='joint_angle'` 是合理的？
