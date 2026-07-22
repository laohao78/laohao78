# FAST-LIO 系列 SLAM 算法关系与演进总览

## 概览：同一实验室的渐进式演进

FAST-LIO、FAST-LIO2、FAST-LIVO 和 FAST-LIVO2 均由**香港大学 MaRS 实验室**开发，是一套从 LiDAR-惯性里程计（LIO）逐步扩展到 LiDAR-惯性-视觉里程计（LIVO）的紧耦合 SLAM 框架。它们的核心关系：

```
FAST-LIO (2020)                → LiDAR + IMU，基于特征提取
    ↓
FAST-LIO2 (2021)               → LiDAR + IMU，直接法，ikd-Tree，全LiDAR类型支持
    ↓
FAST-LIVO (2022)               → LiDAR + IMU + Camera，三传感器融合
    ↓
FAST-LIVO2 (2024, T-RO)       → 统一体素地图，像素级精度，ARM平台适配
    ↓
GS-LIVO (2025)                 → 加入 Gaussian Splatting 实时照片级重建
```

核心关联项目还包括 **R2LIVE**（以 FAST-LIO 为前端的 LiDAR-惯性-视觉融合）、**Point-LIO**（高带宽高速机动）、**FAST-Calib**（快速标定工具）等。

---

## 1. FAST-LIO（2020）— 高效 LiDAR-惯性里程计的开端

| 维度 | 内容 |
|------|------|
| **全称** | Fast LiDAR-Inertial Odometry |
| **传感器** | LiDAR + IMU |
| **核心方法** | 紧耦合**误差状态迭代卡尔曼滤波（ESIKF）** |
| **特征处理** | **需要提取**线/面特征（基于 LOAM-Livox 思路） |
| **地图结构** | 并行 KD-Tree 搜索 |
| **计算效率** | 比当时 LOAM 类方案快约 2 倍 |
| **关键贡献** | ① 提出新的卡尔曼增益计算公式，将复杂度从 \(O(N^2)\) 降至 \(O(1)\)（相对状态维度）；② 基于流形（\(SO(3)\)、\(\mathbb{S}^2\)）的滤波理论；③ 支持快速/激烈运动场景 |

**局限：** 仍需特征提取模块，建图效率有提升空间，小视场角 LiDAR 易退化。

---

## 2. FAST-LIO2（2021）— 直接法 + ikd-Tree 革新

| 维度 | 内容 |
|------|------|
| **全称** | Fast Direct LiDAR-Inertial Odometry |
| **传感器** | LiDAR + IMU |
| **核心方法** | ESIKF + **直接法**（Direct Method） |
| **特征处理** | **无需特征提取**，直接配准原始点云 |
| **地图结构** | **ikd-Tree**（增量 KD-Tree），支持动态增删和部分重建 |
| **LiDAR支持** | 旋转式（Velodyne/Ouster）和固态（Livox Avia/Horizon/MID-70）**全类型** |
| **计算效率** | 比同类方法快 **~10 倍**，支持 >100Hz LiDAR 速率 |
| **平台支持** | 新增 ARM 平台（Khadas VIM3、Nvidia TX2、树莓派 4B） |

**核心创新——ikd-Tree：**
- 增量操作（增/删）时间复杂度 \(O(\log N)\)
- 盒式插入 \(M\) 个点花费 \(O(M \log N)\)
- 动态监控平衡准则，必要时部分重建
- 解决了传统 KD-Tree 不能增量更新的痛点

**直接法原理：** 将当前扫描点投影至地图坐标系 → 在 ikd-Tree 中搜索最近邻并拟合局部平面 → 最小化**点到平面距离**作为观测残差 → ESIKF 迭代优化。

---

## 3. FAST-LIVO（2022）— 三传感器紧耦合融合

FAST-LIVO 在 FAST-LIO2 的基础上**引入相机（视觉）信息**，形成 LiDAR-Inertial-Visual Odometry（LIVO）：

- **传感器组合：** LiDAR + IMU + Camera
- **融合策略：** 紧耦合，时空对齐后通过直接法融合
- **视觉模态：** 使用直接法（光度误差最小化）而非特征点法
- **关键优势：** 在 LiDAR 退化场景下（如长走廊、空旷广场），视觉纹理可以提供额外约束来抑制漂移
- **定位：** 与 R2LIVE（也是 HKU Mars 的工作，以 FAST-LIO 为 LiDAR 前端）共同推动了 LiDAR-惯性-视觉融合方向

---

## 4. FAST-LIVO2（2024，T-RO）— 统一体素地图与像素级重建

这是该系列在 2024-2025 年的**集大成之作**，发表于 IEEE T-RO。

| 维度 | 内容 |
|------|------|
| **传感器** | LiDAR + IMU + Camera |
| **估计器** | **序列化 ESIKF**（先 LiDAR 更新，再视觉更新） |
| **地图结构** | **统一概率体素地图（Unified Voxel Map）** |
| **LiDAR 模块** | 直接法点-面配准 + 双重不确定性建模 |
| **视觉模块** | 直接法稀疏图像对齐（光度误差最小化） |
| **状态维度** | 19 维：姿态 R(3) + 位置 p(3) + 曝光时间 τ(1) + 速度 v(3) + gyro bias(3) + accel bias(3) + 重力 g(3) |
| **精度** | **像素级**重建精度 |
| **鲁棒性** | 在线曝光估计 + 按需射线投射（LiDAR盲区补充） |
| **平台** | 支持 RK3588、Jetson Orin NX、RB5 等 ARM 平台，单帧处理 <78ms |

### 核心技术亮点

1. **统一体素地图：** 每个体素同时存储几何信息（概率平面：法向量 + 协方差）和视觉信息（图像补丁 patches），实现几何与纹理的深度协同。

2. **几何先验引导视觉：** 利用 LiDAR 平面参数作为视觉深度估计的先验，避免了传统直接法中耗时的极线搜索深度估计。

3. **跨模态不确定性传播：** LiDAR 平面质量越好 → 对应视觉约束权重越高；平面越差（如树丛）→ 自动降权，实现自适应融合。

4. **退化检测与补偿：**
   - 对信息矩阵 \(H^T H\) 做特征值分解检测退化方向
   - 退化方向上回退至 IMU 预测值，有效方向上继续使用 LiDAR/视觉约束

5. **序列化 ESIKF：** LiDAR 和视觉观测依次更新状态，解决了多源观测维度不一致的问题。

---

## 五种系统对比总结

| 特性 | FAST-LIO | FAST-LIO2 | FAST-LIVO | FAST-LIVO2 |
|------|----------|-----------|-----------|------------|
| **年份** | 2020 | 2021 | 2022 | 2024 |
| **传感器** | LiDAR+IMU | LiDAR+IMU | LiDAR+IMU+Camera | LiDAR+IMU+Camera |
| **估计器** | IESKF | IESKF | IESKF | 序列化 IESKF |
| **特征提取** | 需要（线/面） | 不需要（直接法） | 不需要（直接法） | 不需要（直接法） |
| **地图结构** | 并行 KD-Tree | ikd-Tree | Voxel Map | 统一概率体素地图 |
| **视觉约束** | 无 | 无 | 有（直接法） | 有（几何先验引导） |
| **抗退化** | 被动 | 被动 | 几何+纹理互补 | 主动（双重互补） |
| **LiDAR 兼容** | Livox 为主 | 全类型 | 全类型 | 全类型 |
| **计算量** | 低 | 极低 | 中 | 中高 |
| **ARM 支持** | 否 | 是 | 部分 | 是（深度优化） |

---

## 最新进展（2025）

根据 HKU MaRS 在 2025 年发布的博士论文及开源代码：

- **FAST-Calib：** 亚厘米级 LiDAR-Camera 标定框架，亚秒级求解，支持机械和固态 LiDAR
- **GS-LIVO：** 首个实时 Gaussian-based LIVO 系统，利用 α-blending 从稀疏点云生成照片级高保真重建，已在 Jetson Orin NX 上实现实时运行
- **轻量级 FAST-LIVO2：** 针对嵌入式平台的退化感知自适应视觉帧选择器 + 内存高效的局部-长期混合地图结构

---

## 技术脉络关系图

```
                    IKFoM 数学基础
                    (Kalman Filters on Differentiable Manifolds, 2021)
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                 ▼
   ikd-Tree        FAST-LIO (2020)    R2LIVE (2021)
   (2021)               │             LiDAR-Inertial-Visual
        │               ▼                 │
        └──────►  FAST-LIO2 (2021)  ◄─────┘
                  (直接法 + ikd-Tree)
                        │
                        ▼
                  FAST-LIVO (2022)
                  (+ Camera 紧耦合)
                        │
                        ▼
                  FAST-LIVO2 (2024)
                  (统一体素地图, T-RO)
                        │
            ┌───────────┼───────────┐
            ▼           ▼           ▼
      GS-LIVO    轻量FAST-LIVO2   相关系统
     (Gaussian)   (ARM优化)     (Point-LIO,
                                  Swarm-LIO2等)
```

## 开源源码

| 系统 | 代码地址 |
|------|---------|
| FAST-LIO/FAST-LIO2 | `github.com/hku-mars/FAST_LIO` |
| FAST-LIVO2 | `github.com/hku-mars/FAST-LIVO2` |
| ikd-Tree | `github.com/hku-mars/ikd-Tree` |
| FAST-Calib | `github.com/hku-mars/FAST-Calib` |

这套算法的核心脉络可以总结为：**从滤波理论创新（IKFoM）→ 数据结构革新（ikd-Tree）→ 传感器模态扩展（LIO↗LIVO）→ 地图表示统一化（Voxel Map）→ 渲染质量提升（Gaussian Splatting）**，始终以极高计算效率为核心竞争力，逐步从单一里程计走向完整的空间智能系统。

---

## 本地运行步骤

```sh
# 步骤 1：启动 FAST-LIVO2 节点（终端 1）
cd ~/gpufree-data/livo2
source ~/gpufree-data/livo2/devel/setup.bash
roslaunch fast_livo mapping_avia.launch

# 步骤 2：播放 bag 文件（终端 2）
cd ~/gpufree-data
source ~/gpufree-data/livo2/devel/setup.bash
rosbag play Red_Sculpture.bag --clock
rosbag play Retail_Street.bag --clock
```
