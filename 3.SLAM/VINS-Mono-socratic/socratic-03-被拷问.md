# VINS-Mono 苏格拉底学习 03：被拷问

使用方式：一题一题答。不要先看源码搜索定义，先用自己的模型推理。每道题答完后，都追问自己一句：

> 为什么你认为是这样？你的解释连接了哪些模型？

如果答不出来，不要急着查答案，先定位盲区属于：可观测性、前端测量、IMU 预积分、滑动窗口优化、边缘化/pose graph。

## 题 1：纯旋转初始化

类型：counterfactual + diagnosis

你把设备拿在手里，启动时只做原地转动，几乎没有平移。前端能跟踪到很多特征，RANSAC 也能过，但系统反复提示初始化困难或尺度不稳定。

问题：为什么“很多特征 + 很好跟踪”仍然不足以完成可靠初始化？请同时解释视觉 SFM、尺度、重力和 IMU 激励之间的关系。

追问：你漏掉了哪个“可观测性”条件？

## 题 2：IMU 噪声参数过于乐观

类型：diagnosis

你把 `acc_n` 和 `gyr_n` 设置得比真实 IMU 噪声小很多。运行时视觉特征质量正常，但轨迹出现明显不合理漂移，优化结果似乎更愿意相信 IMU。

问题：在因子图里，噪声参数如何改变 IMU 残差的权重？这个错误为什么可能污染速度、偏置和位姿，而不只是让 IMU 因子“稍微不准”？

追问：如果你只调前端特征参数，为什么可能治不好这个问题？

## 题 3：外参在线估计失败

类型：cross-connect + diagnosis

你把 `estimate_extrinsic` 设为 2，启动时没有明显旋转，只是缓慢平移。系统一直停留在外参初始化附近，或者后续估计不稳定。

问题：为什么在线外参旋转初始化需要特定运动？请连接 `processImage()` 中的外参初始化逻辑、相邻帧视觉对应、IMU 预积分旋转和可观测性。

追问：这个问题的根因是 Ceres 求解器不够强，还是数据没有提供足够约束？

## 题 4：关键帧与边缘化选择

类型：cross-connect

`FeatureManager::addFeatureCheckParallax()` 根据跟踪数和视差决定 `MARGIN_OLD` 还是 `MARGIN_SECOND_NEW`。

问题：为什么低视差帧更适合被当作非关键帧处理？如果强行保留大量低视差帧，优化问题会多出什么成本，又不能增加哪些约束？

追问：这里的“保留信息”与“控制计算量”如何平衡？

## 题 5：视觉重投影残差里的逆深度

类型：cross-connect

`ProjectionFactor` 用某个特征在首观测帧的逆深度作为参数，然后把点从相机 i 变到 IMU、世界、IMU j、相机 j，最后比较归一化平面坐标。

问题：为什么单目 VIO 常用逆深度而不是直接优化 3D 点坐标？这个选择如何影响远点、初始化深度、边缘化和数值稳定性？

追问：如果某个特征估出的深度为负，为什么 `removeFailures()` 要删掉它？

## 题 6：时间偏移不是普通延迟

类型：counterfactual

你打开 `estimate_td`，系统开始优化相机和 IMU 的时间偏移。假设设备有快速角速度运动，同时图像时间戳比真实曝光时间晚。

问题：为什么时间偏移会表现成重投影误差，而不是简单地让整条轨迹平移一点？`ProjectionTdFactor` 为什么需要特征速度和图像行信息？

追问：如果场景几乎静止或运动很慢，时间偏移是否仍然容易估计？为什么？

## 题 7：边缘化先验的代价

类型：cross-connect + counterfactual

VINS-Mono 通过 `MarginalizationInfo` 把被滑出窗口的变量压成先验。如果历史线性化点后来被证明不理想，这个先验不会自动回到原始非线性问题重新来过。

问题：边缘化为什么既是实时性的关键，也是长期一致性的风险来源？请解释它和滤波方法的相似处。

追问：如果你把窗口无限增大，是不是就解决了一切？代价是什么？

## 题 8：回环为什么不直接修正一切

类型：diagnosis + cross-connect

pose graph 检测到回环后做 4DoF 优化，主要校正 yaw 和平移漂移，而不是完整重新优化所有视觉特征、IMU 预积分和历史状态。

问题：为什么这种设计对实时系统有吸引力？它牺牲了什么？它默认 IMU/VIO 已经可靠估计了哪个方向？

追问：如果 roll/pitch 或重力方向估错，4DoF pose graph 能否从根上修复？

## 题 9：前端 RANSAC 通过但后端仍失败

类型：diagnosis

某段数据中，`findFundamentalMat` 的 RANSAC 后还保留了不少点，特征图看起来也正常，但后端轨迹突然跳变并触发 failure detection。

问题：列出至少三种可能原因，并把它们分别归到前端测量、IMU 模型、初始化/可观测性、滑动窗口优化或边缘化中的某一类。

追问：你会先看哪个日志或变量？为什么不是先改优化器迭代次数？

## 题 10：把 VINS-Mono 改到一个低端滚快门相机 + 未同步 IMU 的平台

类型：cross-connect + counterfactual + diagnosis

你要把系统从 EuRoC 数据集迁移到一个低端滚快门相机、低频且未硬同步的 IMU。README 里说这类组合性能最差。

问题：请按优先级给出你会检查或调整的 8 件事，并说明每一件分别保护哪个模型：前端特征、外参/时间、IMU 噪声、滚快门、初始化、优化实时性、failure detection、pose graph。

追问：如果只能改一个硬件条件，你会优先换相机、提高 IMU 频率、做硬同步，还是改善标定？为什么？

## 诊断报告模板

答完 10 题后，用这个模板复盘：

强项：

- 我能独立解释的模型：
- 我能从源码定位的函数：
- 我能诊断的失败现象：

弱项：

- 我还停留在背概念的地方：
- 我无法连接的两个模型：
- 我需要重新读的源码：

建议重读顺序：

1. `src/feature_tracker/src/feature_tracker.cpp`
2. `src/vins_estimator/src/feature_manager.cpp`
3. `src/vins_estimator/src/factor/integration_base.h`
4. `src/vins_estimator/src/factor/projection_factor.cpp`
5. `src/vins_estimator/src/initial/initial_aligment.cpp`
6. `src/vins_estimator/src/estimator.cpp`
7. `src/pose_graph/src/pose_graph.cpp`

最后追问：

如果让你向另一个人解释 VINS-Mono，你会从“代码模块”开始讲，还是从“可观测性和约束来源”开始讲？为什么？
