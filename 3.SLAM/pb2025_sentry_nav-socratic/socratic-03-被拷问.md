# SMBU2025 苏格拉底学习笔记 03：被拷问

> 学习目标：用问题暴露“我以为我懂了”的地方。  
> <span style="color:#EF6C00"><strong>做题规则：</strong></span> 先自己答，再展开 `<details>` 看参考答案。

## 颜色图例

| 颜色 | 含义 |
|---|---|
| <span style="color:#1565C0"><strong>蓝色</strong></span> | 概念链路 |
| <span style="color:#2E7D32"><strong>绿色</strong></span> | 参考结论 |
| <span style="color:#EF6C00"><strong>橙色</strong></span> | 排查顺序 |
| <span style="color:#C62828"><strong>红色</strong></span> | 风险 / 错误判断 |

## 题目类型

| 类型 | 要求 |
|---|---|
| **Cross-connect** | 跨模型连接：把 TF、定位、costmap、控制串起来 |
| **Counterfactual** | 反事实推理：如果某个条件变了，系统会怎样 |
| **Diagnosis** | 故障诊断：给出现象，判断可能原因和排查顺序 |

---

## 1. `slam:=True` 真的意味着“没有定位”吗？

**类型：** Cross-connect

你需要解释：

- `slam:=True` 时哪些定位相关节点仍然存在？
- 为什么 `point_lio` 仍然需要启动？
- `map -> odom` 在这个模式下来自哪里？
- 这个选择会如何影响 Nav2 的全局规划？

**教练追问：**  
如果你说“SLAM 模式没有重定位”，你说的是没有哪个节点？为什么不是没有 `point_lio`？

<details>
<summary><strong>参考答案</strong></summary>

`slam:=True` 不等于“没有定位”。它仍然启动 `point_lio`，因为机器人仍然需要 LiDAR/IMU 里程计来提供局部连续运动估计和注册点云。

真正没有的是 **`small_gicp_relocalization`**。建图模式下不使用先验 PCD 做重定位，而是启动 `slam_toolbox`、`pointcloud_to_laserscan` 和 `map_saver_server`，并通过静态 `map -> odom` 维持地图和里程计之间的关系。

<span style="color:#2E7D32"><strong>关键句：</strong></span> `point_lio` 两种模式都在；<span style="color:#C62828"><strong>`small_gicp_relocalization` 只在 `slam:=False` 的定位/导航模式中启用。</strong></span>

</details>

---

## 2. 如果 `small_gicp_relocalization` 被错误地放进建图模式，会发生什么？

**类型：** Counterfactual

你需要连接：

- `slam_toolbox`
- `point_lio`
- `small_gicp_relocalization`
- `map -> odom`
- Nav2 costmap

**教练追问：**  
两个模块同时想影响全局位姿时，谁在维护地图一致性？谁在校正漂移？它们的职责会不会打架？

<details>
<summary><strong>参考答案</strong></summary>

会出现职责冲突。建图模式中，`slam_toolbox` 的目标是根据传感器数据维护地图；`small_gicp_relocalization` 的目标是把当前点云和先验 PCD 对齐，并校正 `map -> odom`。

如果两者同时工作，就可能出现两个问题：

| 冲突点 | 后果 |
|---|---|
| `slam_toolbox` 想维护当前地图一致性 | 地图会随在线观测变化 |
| `small_gicp` 想把机器人拉回先验地图 | `map -> odom` 可能被外部强行修正 |
| 两者同时影响全局位姿语义 | Nav2 costmap 和路径会出现跳变或不一致 |

<span style="color:#2E7D32"><strong>核心判断：</strong></span> 建图模式追求“生成/维护地图”，定位模式追求“贴合先验地图”。<span style="color:#C62828"><strong>这两个目标不能随便混在一起。</strong></span>

</details>

---

## 3. 机器人在 RViz 里路径正常，但底盘实际运动方向不对，你先查哪里？

**类型：** Diagnosis

候选方向：

- `gimbal_yaw` / `gimbal_yaw_fake`
- `fake_vel_transform`
- `cmd_vel_nav2_result` 到 `cmd_vel`
- `odometry` 的 child frame
- 控制器输出

**教练追问：**  
为什么这不一定是 `pb_omni_pid_pursuit_controller` 的问题？

<details>
<summary><strong>参考答案</strong></summary>

先查 **速度参考系和速度转换链路**，不要一上来怪控制器。

推荐排查顺序：

| 顺序 | 查什么 | 为什么 |
|---|---|---|
| 1 | Nav2 使用的 `robot_base_frame` 是否是 `gimbal_yaw_fake` | Nav2 可能在 fake frame 下输出速度 |
| 2 | `cmd_vel_nav2_result` 和最终 `cmd_vel` 是否方向一致 | 判断问题在控制器输出前还是 fake 转换后 |
| 3 | `fake_vel_transform` 是否正确读取 `odometry` 和 yaw 差 | 它负责把 fake frame 速度转回真实 frame |
| 4 | `odometry.child_frame_id` 是否符合预期 | frame 语义错会让速度方向错 |
| 5 | 最后再看控制器参数 | 控制器可能没错，只是速度被解释错了 |

<span style="color:#2E7D32"><strong>核心判断：</strong></span> “路径正常但运动方向错”高度怀疑 <span style="color:#EF6C00"><strong>TF / fake frame / cmd_vel remap</strong></span>，不一定是 controller 算法错。

</details>

---

## 4. 如果 `terrain_map` 的 frame 错了，Nav2 会表现出什么症状？

**类型：** Diagnosis

你需要说明：

- 点云 frame 错如何进入 `IntensityVoxelLayer`
- local costmap 和 global costmap 可能分别怎样错
- 机器人会表现为“不走”“绕远”还是“撞障碍”

**教练追问：**  
为什么 costmap 错不一定来自 costmap 参数，也可能来自 TF？

<details>
<summary><strong>参考答案</strong></summary>

`terrain_map` 是点云形式的障碍证据。如果它的 frame 错了，`IntensityVoxelLayer` 会把障碍物标到错误位置。

可能症状：

| 症状 | 可能原因 |
|---|---|
| 明明前方可走却绕远 | 障碍物被投到路径上 |
| 明明有障碍却规划穿过去 | 障碍物被投到别处 |
| local costmap 跳动 | `odom`、雷达 frame 或时间戳不一致 |
| global costmap 长期偏移 | `map -> odom` 或点云 frame 语义错误 |

<span style="color:#2E7D32"><strong>核心判断：</strong></span> costmap 是结果，不是源头。costmap 错可能来自 **terrain 参数**，也可能来自 <span style="color:#C62828"><strong>TF 把点云放错地方</strong></span>。

</details>

---

## 5. 为什么项目要同时有 `terrain_analysis` 和 `terrain_analysis_ext`？

**类型：** Cross-connect

不要只说“一个近一个远”。你需要解释：

- 近距离和远距离地形对控制的意义不同在哪里？
- local costmap 和 global costmap 为什么需要不同尺度的信息？
- 点云衰减、范围、分辨率会如何影响规划？

**教练追问：**  
如果只保留一个 terrain 节点，你会删哪个？你愿意承担什么后果？

<details>
<summary><strong>参考答案</strong></summary>

两者对应不同尺度的导航需求。

| 节点 | 主要服务对象 | 特点 |
|---|---|---|
| `terrain_analysis` | local costmap / 近距离避障 | 分辨率更细，更贴近控制安全 |
| `terrain_analysis_ext` | global costmap / 远距离规划 | 范围更大，帮助全局路径提前绕开不可通行区域 |

如果只有近距离地形，机器人可能全局规划过于乐观，临近障碍才反应。  
如果只有远距离地形，局部控制可能缺少足够细的近场障碍信息。

<span style="color:#2E7D32"><strong>核心判断：</strong></span> local costmap 关心 <span style="color:#EF6C00"><strong>“马上怎么躲”</strong></span>，global costmap 关心 <span style="color:#1565C0"><strong>“远处怎么绕”</strong></span>。

</details>

---

## 6. 如果先验 PCD 和真实场地有偏差，系统最可能在哪几个地方暴露问题？

**类型：** Diagnosis

你需要至少覆盖：

- `small_gicp_relocalization`
- `map -> odom`
- `terrain_map`
- Nav2 global costmap
- 控制器跟踪

**教练追问：**  
你如何区分“定位错了”和“地图本身错了”？

<details>
<summary><strong>参考答案</strong></summary>

先验 PCD 偏差会首先影响 `small_gicp_relocalization`，因为它用当前点云对齐先验点云。如果匹配错误，`map -> odom` 会被校正到错误位置。

连锁反应：

```text
prior PCD 偏差
  -> small_gicp 匹配错误
  -> map -> odom 错
  -> robot pose 在 map 中错
  -> global costmap / path 错
  -> controller 看起来努力跟踪，但目标本身错
```

区分方法：

| 判断点 | 更像定位错 | 更像地图错 |
|---|---|---|
| 当前点云和实物一致，但和地图对不上 | 是 | 可能 |
| robot pose 在 RViz 中整体偏移 | 是 | 不一定 |
| 不同位置都出现固定地图结构偏差 | 不一定 | 是 |
| `small_gicp` 匹配结果跳动 | 是 | 可能 |

<span style="color:#2E7D32"><strong>核心判断：</strong></span> 先看当前点云是否贴合真实环境，再看它是否贴合先验地图。

</details>

---

## 7. 为什么 `point_lio` 输出不能直接无脑喂给 Nav2？

**类型：** Cross-connect

你需要说明：

- point_lio 的输出语义和项目中的 `odom` 语义有什么差异？
- `loam_interface` 做了什么转换？
- `sensor_scan_generation` 接下来又补了什么？

**教练追问：**  
如果跳过 `loam_interface`，你预测 TF tree 或 costmap 会怎么坏？

<details>
<summary><strong>参考答案</strong></summary>

`point_lio` 输出的里程计和注册点云带有 `lidar_odom` 语义，而项目需要把数据整理进真实 `odom` 链路，让 Nav2、terrain、controller 都能用同一套 frame 解释。

链路是：

```text
point_lio
  -> loam_interface: lidar_odom 语义整理到 odom
  -> sensor_scan_generation: 发布 odometry / TF / sensor_scan
  -> terrain + Nav2
```

如果跳过 `loam_interface`，点云和 odometry 可能在 frame 语义上不一致。结果是 costmap 看似有数据，但障碍物、机器人位姿、速度参考方向彼此对不上。

<span style="color:#2E7D32"><strong>核心判断：</strong></span> Nav2 需要的不是“有 odometry”，而是 <span style="color:#C62828"><strong>frame 语义正确的 odometry</strong></span>。

</details>

---

## 8. 如果把 `robot_base_frame` 从 `gimbal_yaw_fake` 改回 `gimbal_yaw`，会发生什么？

**类型：** Counterfactual

你需要分析：

- 云台自旋时 Nav2 会如何理解机器人朝向？
- 局部路径跟踪会出现什么异常？
- `fake_vel_transform` 的存在意义会不会消失？

**教练追问：**  
这个改动是在简化系统，还是在破坏 Nav2 的隐含假设？

<details>
<summary><strong>参考答案</strong></summary>

如果 Nav2 直接使用 `gimbal_yaw`，云台自旋会让 Nav2 认为机器人本体朝向也在剧烈变化。局部路径跟踪器可能把速度方向解释错，导致底盘运动方向异常。

`gimbal_yaw_fake` 的作用是给 Nav2 一个更稳定的速度参考系；`fake_vel_transform` 再把这个稳定 frame 下的速度转换回真实 `gimbal_yaw`。

<span style="color:#2E7D32"><strong>核心判断：</strong></span> 改回 `gimbal_yaw` 看起来简化了 TF，实际可能 <span style="color:#C62828"><strong>破坏 Nav2 对稳定 `robot_base_frame` 的隐含假设</strong></span>。

</details>

---

## 9. 如果机器人总是绕很远，你会优先怀疑 planner、costmap 还是 terrain？

**类型：** Diagnosis

你必须给出一个具体排查路径：

```text
现象
  -> 先看什么可视化
  -> 再查哪个 topic
  -> 再查哪个参数
  -> 最后才改哪个模块
```

**教练追问：**  
为什么不能一上来就调 planner？

<details>
<summary><strong>参考答案</strong></summary>

不要先调 planner。planner 只是基于 costmap 结果做路径搜索，绕远通常先说明 costmap 或 terrain 给了过于保守的障碍证据。

推荐排查：

```text
绕远
  -> RViz 看 local/global costmap 是否障碍过多
  -> 看 terrain_map / terrain_map_ext 是否把可通行区标成障碍
  -> 查 IntensityVoxelLayer 的 intensity / height 阈值
  -> 查 inflation_radius 是否过大
  -> 最后才看 Theta Star planner 权重
```

<span style="color:#2E7D32"><strong>核心判断：</strong></span> planner 看到的是 costmap。<span style="color:#C62828"><strong>如果 costmap 已经错了，调 planner 只是让错误路径变成另一种错误路径。</strong></span>

</details>

---

## 10. 你如何向新人解释“SMBU2025 不是 Nav2 demo”？

**类型：** Cross-connect

你需要用自己的话串起：

- 仿真/实车输入
- `point_lio`
- `small_gicp` 或 `slam_toolbox`
- TF frame
- terrain costmap
- 自定义 controller
- `fake_vel_transform`

**教练追问：**  
如果只能用一张图解释，你会把哪三个节点画得最大？为什么？

<details>
<summary><strong>参考答案</strong></summary>

SMBU2025 不是简单把 Nav2 跑起来，而是围绕 RoboMaster 哨兵场景做了一整条导航闭环。

一条合格解释应该包括：

```text
仿真/实车传感器
  -> point_lio 提供局部里程计
  -> slam_toolbox 或 small_gicp 处理 map -> odom
  -> loam_interface / sensor_scan_generation 整理 TF 和 odometry
  -> terrain_analysis 把点云变成 costmap 证据
  -> planner_server 生成 Path，不是 cmd_vel
  -> controller_server 调用 pb_omni_pid_pursuit_controller
  -> 控制器把 Path 跟踪问题转换成速度命令
  -> fake_vel_transform 修正速度参考系
  -> cmd_vel
```

如果只能画三个最大节点，我会画：

| 节点/模块 | 原因 |
|---|---|
| **TF / frame 链路** | 它决定所有数据如何被解释 |
| **point_lio + small_gicp/slam_toolbox** | 它决定机器人相信自己在哪里 |
| **terrain costmap + fake_vel_transform** | 它决定 Nav2 的路径和底盘速度是否可执行 |

<span style="color:#2E7D32"><strong>核心判断：</strong></span> Nav2 是框架，不是全部；SMBU2025 的工程价值在于把 <span style="color:#1565C0"><strong>定位、TF、地形和控制</strong></span> 接成比赛可用闭环。

</details>

---

## 诊断表

| 如果你卡在 | 说明你可能没吃透 | 回去重读 |
|---|---|---|
| `slam:=True` / `False` 的节点差异 | 启动编排模型 | `bringup_launch.py`、`slam_launch.py`、`localization_launch.py` |
| `map -> odom -> base` 关系 | TF 坐标系治理模型 | `loam_interface`、`sensor_scan_generation`、Nav2 params |
| `gimbal_yaw_fake` 的意义 | 速度参考系和控制模型 | `fake_vel_transform` README 和源码 |
| `terrain_map` 如何进 costmap | 点云到可通行性模型 | `terrain_analysis`、`terrain_analysis_ext`、`IntensityVoxelLayer` |
| 路径正常但运动异常 | 规划控制和 TF 的连接 | `navigation_launch.py`、controller params、`cmd_vel` remap |

## 最后一问

请用不超过 5 句话回答：

> <span style="color:#6A1B9A"><strong>SMBU2025 的导航闭环中，哪一环最容易“看起来正常但实际错了”？为什么？</strong></span>

如果你的回答没有同时提到 **TF、定位、costmap 或速度参考系** 中至少两个概念，说明你还在背模块名，没有真正建立系统模型。
