# Socratic Learn — Go1 Step 2 补充视角

<p align="center">
  <img src="https://img.shields.io/badge/Topic-Go1%20Walk%20These%20Ways-blue" alt="Topic">
  <img src="https://img.shields.io/badge/Method-Socratic%20Learn-orange" alt="Method">
  <img src="https://img.shields.io/badge/Step-2%2F3%20补充视角-red" alt="Step 2/3">
</p>

---

## 争议五：<span style="color:#ffa726">Dense Reward</span> vs <span style="color:#42a5f5">课程 + 稀疏奖励</span>

<p align="center">
  <img src="https://img.shields.io/badge/△-底层分歧：复杂度放在奖励里，还是放在探索和课程里？-critical" alt="Core">
</p>

### 我的立场：偏 B

<span style="color:#9e9e9e">不是哲学偏好，是基于预训练模型的实际数据。</span>

预训练模型 4982 个快照里：

| 指标 | 早期 (0-6k) | 末期 (35k-49k) | <span style="color:#ef5350">变化</span> |
|------|:----------:|:----------:|:----:|
| `orient / total` | -2.4 | -9.1 | <span style="color:#ef5350">×4</span> |
| `lin_vel / total` | 3.9 | 4.5 | <span style="color:#26a69a">+15%</span> |
| 命令速度幅值 | 1.46 m/s | 2.60 m/s | ×1.78 |

命令只难了不到两倍，姿态惩罚涨了四倍。ji22 的 `exp(neg / 0.02)` 把小姿态误差指数放大——<span style="color:#ef5350">后期策略更多是在防御 orient 项爆炸，而非追求更快更稳。</span>

### 怎么区分谁对谁错

| | 实验 | 预期结果 |
|:--|:--|:--|
| 🟢 我错 | 去掉 stance_width、raibert_heuristic、feet_clearance_cmd_linear | 策略出现拖脚或身体侧倾 |
| 🔴 对方错 | `orientation_control` scale: -5.0 → -2.0 或 `sigma_rew_neg`: 0.02 → 0.1 | 速度跟踪和姿态控制同时改善 |

---

## 课程 · 密集奖励 · 稀疏奖励

<p align="center">
  <img src="https://img.shields.io/badge/三个概念解决的是不同层面的问题-informational" alt="Core">
</p>

### <span style="color:#26a69a">课程 (Curriculum)</span>

> 不预设难度曲线，难度随策略表现自动调整。

| 课程 | 调控对象 | 触发条件 |
|:--|:--|:--|
| **地形课程** | 地面粗糙度、台阶高度 | 跟踪达标 → 更难地形 |
| **命令课程** | 速度/步态参数范围 | 当前分布下跟踪达标 → 扩范围 |

训练初期 0.5 m/s 平地慢走 → 后期 ±2.6 m/s 复杂地形 + 四种步态全覆盖。

课程只管<span style="color:#26a69a">"题目出多难"</span>，不管"怎么做是对的"——后者是奖励函数的职责。

### <span style="color:#ffa726">密集奖励 (Dense Reward)</span>

> 每一步都给反馈：你这一小步走得好不好。

20+ 项奖励，每 0.02 秒计算一次：

```text
速度跟踪 + 姿态 + 关节力矩 + 触地力时序 + 足端间隙
+ 碰撞 + 动作平滑度 + slippage + Raibert 先验
    ↓
ji22 变换: rew = pos × exp(neg / 0.02)
```

<span style="color:#ef5350">代价：</span>权重互相作用。后期 orient 惩罚一炸，其他项再高分也被乘法压扁。

### <span style="color:#42a5f5">稀疏奖励 (Sparse Reward)</span>

> 只在关键事件给信号，中间全靠策略自己探索。

假设只用两个信号：`摔倒 = -1`，`维持速度 10s = +1`。中间 500 步没有任何指导。

<span style="color:#26a69a">好处：</span>不会被某一项权重绑架全局。
<span style="color:#ef5350">坏处：</span>可能永远碰不出稳定 gait。

### 分歧本质

<table>
<tr>
<td align="center" width="50%" style="border-left:4px solid #ff9800;padding:12px">
<b>路线 A</b><br/>
课程 → 密集奖励 → 策略<br/>
<sub>人类既定难度，又教每一步</sub>
</td>
<td align="center" width="50%" style="border-left:4px solid #2196f3;padding:12px">
<b>路线 B</b><br/>
课程 → 稀疏奖励 → 策略<br/>
<sub>人类只定义成功，策略自己发现</sub>
</td>
</tr>
</table>

分歧不在"奖励多还是少"，而在 <span style="color:#7e57c2">**谁定义什么是好的运动**</span>。路线 A 假设工程师知道四足怎么走最好；路线 B 承认工程师不一定知道。

---

## 三个亮点

### <span style="color:#26a69a">1. gait 作为连续 latent</span>

不是四个模型然后切换。15 维命令 + 2 维 clock → 同一个网络在 trot/pace/bound/pronk 之间连续平移。

```text
command = [vx, vy, yaw, height, freq, phase, offset,
           bound, duration, swing_height, pitch, roll,
           stance_width, stance_length]
clock   = [sin(2πt/T), cos(2πt/T)]
```

### <span style="color:#7e57c2">2. adaptation module 只有 2 维</span>

RMA 浓缩到 `num_privileged_obs = 2`：摩擦 + 弹性恢复系数。训练时 critic 有真值做 teacher forcing，部署时从观测历史推测。维度来自 `priv_observe` 开关的精确控制——2 维够用本身就是结论。

### <span style="color:#ffa726">3. 全部整合到能跑，且上了真机</span>

```text
actuator net + history wrapper(30帧) + DR(7项)
+ 地形课程 + 命令课程 + ji22 奖励
+ PPO(adaptation loss) + JIT 导出 + LCM 部署
```

单个模块不新鲜。全串起来跑通真机 Go1，需要大量工程判断。数据里的 reward 下沉说明这套组合不是自动就对的——调出来就是贡献。
