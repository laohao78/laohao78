# VINS-Mono 与 VINS-Fusion 总结

> 香港科技大学空中机器人组 (HKUST Aerial Robotics Group) 开源的两个视觉-惯性 SLAM 系统。

---

## 1. VINS-Mono：单目视觉-惯性状态估计器

### 1.1 概述

VINS-Mono 是 **单目相机 + IMU** 的实时 SLAM 框架，基于优化驱动的滑动窗口方法，提供高精度视觉-惯性里程计 (VIO)。核心论文发表于 **IEEE TRO 2018**。

### 1.2 架构（三个 ROS 节点）

| 模块 | ROS 包 | 功能 |
|---|---|---|
| **前端** | `feature_tracker` | KLT 光流追踪角点，提取新 feature，发布 feature topic |
| **后端** | `vins_estimator` | 滑动窗口紧耦合 VIO：IMU 预积分、视觉-惯性 BA、在线标定 |
| **闭环** | `pose_graph` | DBoW2 回环检测 + 全局位姿图优化 + 地图合并/复用 |

```
相机图像 ──► feature_tracker ──► vins_estimator ──► pose_graph
  IMU ═══════════════════════════════►                (回环/全局优化)
```

### 1.3 核心功能

- **IMU 预积分**：含 bias 校正的高效预积分
- **自动初始化**：视觉-惯性松耦合初始化，无需先验
- **在线外参标定**：相机-IMU 外参在线估计 (`estimate_extrinsic`)
- **在线时间标定**：相机-IMU 时间偏置在线估计 (`estimate_td`)
- **回环检测**：基于 DBoW2 词袋模型
- **4-DoF 全局位姿图优化**：回环 + 地图合并 + 位姿图复用
- **Rolling Shutter 支持**：可设置 `rolling_shutter` 参数
- **失败检测与恢复**：跟踪丢失后的重定位

### 1.4 传感器支持

| 传感器 | 支持 |
|---|---|
| 单目 + IMU | ✅ 主要模式 |
| 双目 | ❌ |
| GPS | ❌ |

### 1.5 依赖

- Ubuntu 16.04 / ROS Kinetic
- Ceres Solver **1.14.0**（不兼容 2.0+）
- OpenCV 3.x

---

## 2. VINS-Fusion：多传感器状态估计器

### 2.1 概述

VINS-Fusion 是 VINS-Mono 的扩展版本，支持 **多种传感器组合**。KITTI Odometry Benchmark 双目开源算法排名第一（2019.01）。

### 2.2 架构（四个 ROS 节点）

| 模块 | ROS 包 | 功能 |
|---|---|---|
| **前端+后端** | `vins` | FeatureTracker + VIO 滑动窗口估计器（单目/双目/双目+IMU） |
| **闭环** | `loop_fusion` | DBoW2 回环检测 + 全局位姿图优化 |
| **全局融合** | `global_fusion` | GPS + VIO 全局融合（toy example） |
| **相机标定** | `camera_models` | 支持 pinhole/mei/equidistant 等多种相机模型标定 |

```
相机图像(单目/双目) ──► vins_estimator ──► loop_fusion ──► global_fusion
       IMU ═══════════════════►               (回环)         (GPS融合)
```

### 2.3 核心功能（VINS-Mono 全部 + 新增）

| 功能 | VINS-Mono | VINS-Fusion |
|---|---|---|
| 滑动窗口 VIO | ✅ | ✅ |
| IMU 预积分 | ✅ | ✅ |
| 自动初始化 | ✅ | ✅ |
| 在线外参标定 | ✅ | ✅ |
| 在线时间标定 | ✅ | ✅ |
| 回环检测 | ✅ | ✅ |
| 4-DoF 位姿图优化 | ✅ | ✅ |
| **多传感器 (单目/双目/IMU)** | ❌ | ✅ 新增 |
| **纯双目模式** | ❌ | ✅ 新增 |
| **GPS 全局融合** | ❌ | ✅ 新增 |
| **多相机模型** | pinhole, MEI | pinhole, MEI, equidistant, Scaramuzza |
| **标定工具** | 无独立工具 | camera_models 包 |

### 2.4 五种运行模式

```bash
# 1. EuRoC 单目+IMU（同 VINS-Mono）
rosrun vins vins_node euroc_mono_imu_config.yaml

# 2. EuRoC 双目+IMU
rosrun vins vins_node euroc_stereo_imu_config.yaml

# 3. EuRoC 纯双目（无IMU）
rosrun vins vins_node euroc_stereo_config.yaml

# 4. KITTI 双目里程计
rosrun vins kitti_odom_test kitti_config00-02.yaml DATASET/sequences/00/

# 5. KITTI 双目+GPS 融合
rosrun vins kitti_gps_test kitti_10_03_config.yaml DATASET/2011_10_03_drive_0027_sync/
rosrun global_fusion global_fusion_node
```

### 2.5 依赖

- Ubuntu 16.04/18.04 / ROS Kinetic/Melodic（本环境 Ubuntu 20.04 / ROS Noetic + OpenCV 4.x 已适配）
- Ceres Solver

---

## 3. 架构差异对比

| 维度 | VINS-Mono | VINS-Fusion |
|---|---|---|
| **前端位置** | 独立节点 `feature_tracker` | 集成在 `vins_estimator` 内部 `featureTracker/` |
| **后端** | `vins_estimator`（单目） | `vins_estimator`（单目+双目） |
| **闭环** | `pose_graph` | `loop_fusion`（同源，结构几乎一致） |
| **GPS** | ❌ | `global_fusion` |
| **标定** | 无独立工具 | `camera_models` |
| **DBoW + DVision** | `pose_graph/src/ThirdParty/` | `loop_fusion/src/ThirdParty/` |

---

## 4. 代码目录结构对比

```
VINS-Mono/src/                       VINS-Fusion/src/VINS-Fusion/
├── feature_tracker/   前端(KLT)     ├── vins_estimator/     前端+后端
│   ├── src/feature_tracker.cpp      │   ├── src/
│   └── src/feature_tracker_node.cpp │   │   ├── featureTracker/   ← 原 feature_tracker
│                                    │   │   ├── estimator/        ← 原 vins_estimator
├── vins_estimator/    后端(VIO)     │   │   ├── factor/           视觉+IMU+双目因子
│   ├── src/estimator.cpp            │   │   ├── initial/
│   ├── src/estimator_node.cpp       │   │   ├── KITTIGPSTest.cpp
│   ├── src/factor/                  │   │   └── KITTIOdomTest.cpp
│   ├── src/initial/                 │   └── config/
│   └── src/feature_manager.cpp      │
│                                    ├── loop_fusion/        闭环
├── pose_graph/        闭环          │   ├── src/pose_graph.cpp
│   ├── src/pose_graph.cpp           │   ├── src/keyframe.cpp
│   ├── src/keyframe.cpp             │   └── src/ThirdParty/DBoW+DVision
│   └── src/ThirdParty/DBoW/         │
│                                    ├── global_fusion/      GPS融合
└── camera_model/      相机模型      │   ├── src/globalOpt.cpp
    └── src/                         │   └── src/globalOptNode.cpp
                                     │
                                     └── camera_models/      相机标定工具
                                         └── src/calib/
                                         └── src/chessboard/
```

---

## 5. 核心算法流程

```
┌──────────────────────────────────────────────────────────────┐
│  1. 数据预处理                                               │
│     图像 ──► KLT 光流 / 双目匹配 ──► 特征点跟踪               │
│     IMU  ──► 预积分 (Δp, Δv, Δq + 协方差)                    │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│  2. 初始化 (视觉-惯性松耦合)                                  │
│     纯视觉 SFM (滑窗内) ──► 对齐 IMU 预积分                   │
│     ──► 估计陀螺仪偏置、速度、重力方向、尺度                   │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│  3. 滑动窗口紧耦合 VIO                                       │
│                                                              │
│     ┌──────────┐    ┌──────────┐    ┌───────────┐           │
│     │ 视觉约束  │    │ IMU约束  │    │ 边缘化先验 │           │
│     │ 重投影误差│ +  │ 预积分误差│ +  │          │           │
│     │ 双目约束  │    │          │    │          │           │
│     └──────────┘    └──────────┘    └───────────┘           │
│              │              │              │                  │
│              └──────────────┼──────────────┘                  │
│                             │                                 │
│                     Ceres 联合优化                             │
│                  (在线外参 + 时间偏置)                         │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│  4. 回环检测与全局优化                                       │
│     关键帧 ──► DBoW2 词袋查询 ──► 回环候选                    │
│     ──► 几何验证 (2D-2D + PnP) ──► 4-DoF 位姿图优化          │
│     ──► 地图合并 / 位姿图复用                                │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼ (仅 VINS-Fusion)
┌──────────────────────────────────────────────────────────────┐
│  5. GPS 全局融合                                             │
│     VIO 局部轨迹 + GPS 全局测量 ──► 全局优化                  │
│     (将 VIO 位姿对齐到 GPS 坐标系)                            │
└──────────────────────────────────────────────────────────────┘
```

---

## 6. 实际运行命令

### 6.1 VINS-Mono (EuRoC 数据集)

```sh
# 终端 1 - 启动 VINS 估计器核心
source ~/gpufree-data/VINS-Mono/devel/setup.bash
roslaunch vins_estimator euroc.launch

# 终端 2 - 启动 RViz 可视化界面
source ~/gpufree-data/VINS-Mono/devel/setup.bash
rviz -d ~/gpufree-data/VINS-Mono/src/config/vins_rviz_config.rviz

# 终端 3 - 播放 bag 文件
cd ~/gpufree-data
rosbag play MH_01_easy.bag --clock
rosbag play MH_02_easy.bag --clock
rosbag play MH_03_medium.bag --clock
rosbag play MH_04_difficult.bag --clock
rosbag play MH_05_difficult.bag --clock

rosbag play V1_01_easy.bag --clock
rosbag play V1_02_medium.bag --clock
rosbag play V1_03_difficult.bag --clock

rosbag play V2_01_easy.bag --clock
rosbag play V2_02_medium.bag --clock
rosbag play V2_03_difficult.bag --clock
```

### 6.2 VINS-Fusion (EuRoC)

```sh
source ~/gpufree-data/VINS-Fusion/devel/setup.bash

# 终端 1 - RViz
roslaunch vins vins_rviz.launch

# 终端 2 - VIO (三选一)
rosrun vins vins_node ~/gpufree-data/VINS-Fusion/src/VINS-Fusion/config/euroc/euroc_mono_imu_config.yaml    # 单目+IMU
rosrun vins vins_node ~/gpufree-data/VINS-Fusion/src/VINS-Fusion/config/euroc/euroc_stereo_imu_config.yaml  # 双目+IMU
rosrun vins vins_node ~/gpufree-data/VINS-Fusion/src/VINS-Fusion/config/euroc/euroc_stereo_config.yaml      # 纯双目

# 终端 3 (可选) - 回环
rosrun loop_fusion loop_fusion_node ~/gpufree-data/VINS-Fusion/src/VINS-Fusion/config/euroc/euroc_mono_imu_config.yaml

# 终端 4 - 播放数据
rosbag play MH_01_easy.bag --clock
```

### 6.3 VINS-Fusion (KITTI)

```sh
source ~/gpufree-data/VINS-Fusion/devel/setup.bash

# 终端 1 - RViz
roslaunch vins vins_rviz.launch

# 终端 2 (可选) - 回环
rosrun loop_fusion loop_fusion_node ~/gpufree-data/VINS-Fusion/src/VINS-Fusion/config/kitti_odom/kitti_config00-02.yaml

# 终端 3 - KITTI 双目里程计
rosrun vins kitti_odom_test ~/gpufree-data/VINS-Fusion/src/VINS-Fusion/config/kitti_odom/kitti_config00-02.yaml YOUR_DATASET/sequences/00/

# KITTI 双目+GPS 融合
rosrun vins kitti_gps_test ~/gpufree-data/VINS-Fusion/src/VINS-Fusion/config/kitti_raw/kitti_10_03_config.yaml YOUR_DATASET/2011_10_03_drive_0027_sync/
rosrun global_fusion global_fusion_node
```

---

## 7. 参考文献

- **VINS-Mono**: T. Qin, P. Li, S. Shen, *"VINS-Mono: A Robust and Versatile Monocular Visual-Inertial State Estimator"*, IEEE TRO, 2018.
- **时间标定 (IROS 最佳学生论文)**: T. Qin, S. Shen, *"Online Temporal Calibration for Monocular Visual-Inertial Systems"*, IROS 2018.
- 项目地址：[VINS-Mono](https://github.com/HKUST-Aerial-Robotics/VINS-Mono) | [VINS-Fusion](https://github.com/HKUST-Aerial-Robotics/VINS-Fusion)
