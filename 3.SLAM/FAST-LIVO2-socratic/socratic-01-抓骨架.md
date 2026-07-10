# 第一步：抓骨架

## 一句话总纲

**FAST-LIVO2 = 一个不断循环的系统：**

`预测 -> 多源观测 -> 状态更新 -> 地图更新 -> 下一轮继续用地图`

先记角色分工：

| 模块 | 它负责什么 |
| --- | --- |
| IMU | 预测 |
| LiDAR | 提供几何观测 |
| Vision | 提供图像观测 |
| Map | 提供参考，并持续更新 |

---

## 一轮到底在干什么

先只记这一句：

`收数据 -> IMU先预测 -> LiDAR更新 -> Vision更新 -> 地图更新`

如果你觉得抽象，就换成更口语的版本：

`先把数据凑齐，再让IMU先猜，然后拿激光纠正，再拿图像继续纠正，最后把结果写回地图`

---

## 主循环

先看入口：

- `main.cpp`
- `LIVMapper::run`

主循环只有 3 件事：

1. `sync_packages()`
   把这一轮要处理的 LiDAR、IMU、图像凑成一包。

2. `processImu()`
   用 IMU 做预测，并把点云去畸变。

3. `stateEstimationAndMapping()`
   进入这一轮的观测更新。

你现在先建立一个感觉：

**FAST-LIVO2 不是“全局一起算”，而是“一轮一轮地算”。**

---

## 五个核心块

### 1. 组包

**它在做什么**

- 按时间把 LiDAR、IMU、图像整理成这一轮的数据包

**你可以怎么想**

- 像做饭前先配菜

**它为什么重要**

- 后面所有预测和更新都建立在“时间对齐”上

**先看哪里**

- `LIVMapper::sync_packages`

**先记住**

> FAST-LIVO2 不是拿到什么就算什么，而是每一轮先组一包正确的数据。

---

### 2. IMU 先预测

**它在做什么**

- 先估计现在大概转到哪、移到哪、速度多少
- 顺手把 LiDAR 点云去畸变

**你可以怎么想**

- IMU 先给一个起点

**它为什么重要**

- 如果没有预测，LiDAR 和 Vision 都不知道从哪里开始对齐

**先看哪里**

- `LIVMapper::processImu`
- `ImuProcess::UndistortPcl`

**先记住**

> IMU 不负责单独给出最终答案，它负责给后面的观测更新提供预测值。

---

### 3. LiDAR 提供观测

**它在做什么**

- 把当前点云放到地图里比对
- 看这些点和已有平面贴不贴
- 根据误差更新状态

**真正拆开，其实是三步**

1. 找对应
2. 建残差
3. 用残差更新状态

**你可以怎么想**

- IMU 说：“我猜你在这”
- LiDAR 说：“我看到你和地图还有这些误差”

**先看哪里**

- `LIVMapper::handleLIO`
- `VoxelMapManager::StateEstimation`

**先记住**

> LiDAR 不是自己单独求位姿，而是提供观测，让系统修正 IMU 预测。

---

### 4. Vision 也提供观测

**它在做什么**

- 看地图里的视觉点投到当前图像上对不对
- 看 patch 像不像
- 再继续更新状态

**你可以怎么想**

- LiDAR 看几何
- Vision 看纹理

**注意**

- Vision 不是替代 LiDAR
- Vision 也不是可有可无的装饰
- 它是第二轮 measurement update

**先看哪里**

- `LIVMapper::handleVIO`
- `VIOManager::processFrame`

**先记住**

> 在执行顺序上，Vision 排在 LiDAR 后面；在系统角色上，它和 LiDAR 都属于观测更新。

---

### 5. 地图不是最后结果，而是工作内存

**它在做什么**

- 提供 LiDAR 找平面的参考
- 提供 Vision 找视觉点的参考
- 在每轮更新后继续被写回

**你可以怎么想**

- 地图不是“最后导出的成果”
- 地图是系统一直在用的内存

**先看哪里**

- `VoxelMapManager::BuildVoxelMap`
- `VoxelMapManager::UpdateVoxelMap`

**先记住**

> 地图不是最后一步，而是一个持续被查询、持续被更新、再持续参与下一轮的闭环部件。

---

## 线性图 vs 闭环图

### 如果先按直线记

`组包 -> IMU预测 -> LiDAR观测更新 -> Vision观测更新 -> 地图更新`

### 但真实结构更像闭环

`地图 -> LiDAR观测`

`地图 -> Vision观测`

`IMU预测 + 多源观测 -> 状态更新`

`状态更新 -> 地图更新`

`地图更新 -> 下一轮继续提供参考`

真正该记住的是这句：

**FAST-LIVO2 不是“算完再存地图”，而是“边用地图，边更新地图”。**

---

## 最小阅读顺序

如果你现在要读源码，建议顺序固定成这样：

1. `main.cpp`
   先确认入口很薄。

2. `LIVMapper::run`
   先看主循环每轮做什么。

3. `LIVMapper::sync_packages`
   先看“这一轮的数据包”怎么来。

4. `LIVMapper::processImu`
   再看 IMU 在主流程里的位置。

5. `LIVMapper::handleLIO`
   再看 LiDAR 怎么提供观测并触发更新。

6. `LIVMapper::handleVIO`
   最后看 Vision 怎么接在 LiDAR 后面继续更新。

这个顺序比直接冲进 `vio.cpp` 或 `voxel_map.cpp` 更稳。

---

## 速记版

如果只留 4 句：

1. FAST-LIVO2 是按轮次运行的，不是一次性全局算完。
2. IMU 负责预测，LiDAR 和 Vision 负责观测更新。
3. LiDAR 先更新，Vision 后更新，但两者都属于 measurement update。
4. 地图不是结果展示，而是闭环里的工作内存。

---

## 反问

先别急着进第二步。你先用自己的话回答两个问题：

1. FAST-LIVO2 每一轮最先做的事是什么？
2. LiDAR 和 Vision 在这个系统里，谁负责预测，谁负责观测？
