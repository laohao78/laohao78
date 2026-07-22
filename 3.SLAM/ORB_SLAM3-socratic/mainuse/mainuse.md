# ORB_SLAM3 使用指南

## 传感器模式 (6种)

| 模式 | 枚举值 | 输入 | 尺度 |
|------|--------|------|------|
| **MONOCULAR** | 0 | 单目图像 | 无绝对尺度 |
| **STEREO** | 1 | 双目图像对 | 有绝对尺度 |
| **RGBD** | 2 | RGB + 深度图 | 有绝对尺度 |
| **IMU_MONOCULAR** | 3 | 单目图像 + IMU | 有绝对尺度 |
| **IMU_STEREO** | 4 | 双目图像对 + IMU | 有绝对尺度 |
| **IMU_RGBD** | 5 | RGB + 深度图 + IMU | 有绝对尺度 |

---

## 数据准备

EuRoC 数据集以 ROS bag 形式存在，需先解包为文件夹格式：

```
<sequence>/
└── mav0/
    ├── cam0/data/
    │   ├── <timestamp_ns>.png
    │   └── ...
    └── imu0/
        └── data.csv
```

### 解包脚本

```sh
cd ~/gpufree-data && source /opt/ros/noetic/setup.bash && python3 extract_bag_to_euroc.py <bag_file> <output_dir>
```

示例：

```sh
# MH 系列 (单目+IMU, 20Hz cam, 200Hz IMU)
python3 extract_bag_to_euroc.py MH_01_easy.bag MH_01_easy
python3 extract_bag_to_euroc.py MH_02_easy.bag MH_02_easy
python3 extract_bag_to_euroc.py MH_03_medium.bag MH_03_medium
python3 extract_bag_to_euroc.py MH_04_difficult.bag MH_04_difficult
python3 extract_bag_to_euroc.py MH_05_difficult.bag MH_05_difficult

# Vicon 系列
python3 extract_bag_to_euroc.py V1_01_easy.bag V1_01_easy
python3 extract_bag_to_euroc.py V1_02_medium.bag V1_02_medium
python3 extract_bag_to_euroc.py V1_03_difficult.bag V1_03_difficult
python3 extract_bag_to_euroc.py V2_01_easy.bag V2_01_easy
python3 extract_bag_to_euroc.py V2_02_medium.bag V2_02_medium
python3 extract_bag_to_euroc.py V2_03_difficult.bag V2_03_difficult
```

---

## 运行示例

### 1. IMU_MONOCULAR — 单目惯性 (VIO)

```sh
cd ~/gpufree-data/ORB_SLAM3/src/ORB_SLAM3

./Examples/Monocular-Inertial/mono_inertial_euroc \
    ./Vocabulary/ORBvoc.txt \
    ./Examples/Monocular-Inertial/EuRoC.yaml \
    ~/gpufree-data/MH_01_easy \
    ~/gpufree-data/MH_01_easy/timestamps.txt
```

参数：`vocabulary settings sequence_folder timestamps_file`

### 2. MONOCULAR — 纯单目

```sh
./Examples/Monocular/mono_euroc \
    ./Vocabulary/ORBvoc.txt \
    ./Examples/Monocular/EuRoC.yaml \
    ~/gpufree-data/MH_01_easy \
    ~/gpufree-data/MH_01_easy/timestamps.txt
```

### 3. STEREO — 纯双目

```sh
./Examples/Stereo/stereo_euroc \
    ./Vocabulary/ORBvoc.txt \
    ./Examples/Stereo/EuRoC.yaml \
    ~/gpufree-data/MH_01_easy \
    ~/gpufree-data/MH_01_easy/timestamps.txt
```

### 4. IMU_STEREO — 双目惯性

```sh
./Examples/Stereo-Inertial/stereo_inertial_euroc \
    ./Vocabulary/ORBvoc.txt \
    ./Examples/Stereo-Inertial/EuRoC.yaml \
    ~/gpufree-data/MH_01_easy \
    ~/gpufree-data/MH_01_easy/timestamps.txt
```

---

## 时间戳文件

位于 `Examples/<Mode>/EuRoC_TimeStamps/` 下：

| 序列 | 时间戳文件 |
|------|-----------|
| MH_01_easy | MH01.txt |
| MH_02_easy | MH02.txt |
| MH_03_medium | MH03.txt |
| MH_04_difficult | MH04.txt |
| MH_05_difficult | MH05.txt |
| V1_01_easy | V101.txt |
| V1_02_medium | V102.txt |
| V1_03_difficult | V103.txt |
| V2_01_easy | V201.txt |
| V2_02_medium | V202.txt |
| V2_03_difficult | V203.txt |

> **注意**：bag 解包后生成的时间戳可能与官方 EuRoC 时间戳文件有差异，建议优先用官方提供的 `EuRoC_TimeStamps/` 下的文件，保证轨迹评估的一致性。

---

### 5. RGBD — TUM RGB-D 数据集

```sh
cd ~/gpufree-data/ORB_SLAM3/src/ORB_SLAM3

./Examples/RGB-D/rgbd_tum \
    ./Vocabulary/ORBvoc.txt \
    ./Examples/RGB-D/TUM3.yaml \
    ~/gpufree-data/00/TUM/rgbd_dataset_freiburg3_walking_xyz \
    ~/gpufree-data/00/TUM/rgbd_dataset_freiburg3_walking_xyz/associations.txt
```

参数：`vocabulary settings sequence_folder association_file`

三个 TUM 数据集：

```sh
# walking_xyz (827 frames) — 在桌面上做圆周运动
./Examples/RGB-D/rgbd_tum ./Vocabulary/ORBvoc.txt \
    ./Examples/RGB-D/TUM3.yaml \
    ~/gpufree-data/00/TUM/rgbd_dataset_freiburg3_walking_xyz \
    ~/gpufree-data/00/TUM/rgbd_dataset_freiburg3_walking_xyz/associations.txt

# walking_halfsphere — 在半个球面上的轨迹
./Examples/RGB-D/rgbd_tum ./Vocabulary/ORBvoc.txt \
    ./Examples/RGB-D/TUM3.yaml \
    ~/gpufree-data/00/TUM/rgbd_dataset_freiburg3_walking_halfsphere \
    ~/gpufree-data/00/TUM/rgbd_dataset_freiburg3_walking_halfsphere/associations.txt

# walking_static — 很小的运动，主要用于测试回环
./Examples/RGB-D/rgbd_tum ./Vocabulary/ORBvoc.txt \
    ./Examples/RGB-D/TUM3.yaml \
    ~/gpufree-data/00/TUM/rgbd_dataset_freiburg3_walking_static \
    ~/gpufree-data/00/TUM/rgbd_dataset_freiburg3_walking_static/associations.txt
```

TUM 数据集目录结构：

```
rgbd_dataset_freiburg3_walking_xyz/
├── rgb/                      # RGB 图片 (640x480)
│   ├── 1341846313.592026.png
│   └── ...
├── depth/                    # 深度图 (5000 = 1m)
│   └── ...
├── rgb.txt                   # RGB 时间戳列表
├── depth.txt                 # 深度时间戳列表
├── associations.txt          # RGB-深度 对齐后的时间戳 (程序用这个)
├── accelerometer.txt         # 加速度计数据 (无实际用途)
└── groundtruth.txt           # 真值轨迹 (用于评估)
```

> `RGBD.DepthMapFactor: 5000.0` — 深度图值 ÷ 5000 = 米

---

## 输出

运行结束后会在当前目录生成：

- `CameraTrajectory.txt` — 相机轨迹（TUM 格式）
- `KeyFrameTrajectory.txt` — 关键帧轨迹
- 若带额外文件名参数，则输出为 `f_<name>.txt` / `kf_<name>.txt`
