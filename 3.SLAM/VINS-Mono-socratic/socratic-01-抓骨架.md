# VINS-Mono 苏格拉底学习 01：抓骨架

学习对象：`/root/gpufree-data/VINS-Mono`

源码锚点：
- `src/feature_tracker/src/feature_tracker.cpp`
- `src/vins_estimator/src/estimator.cpp`
- `src/vins_estimator/src/factor/integration_base.h`
- `src/vins_estimator/src/factor/projection_factor.cpp`
- `src/vins_estimator/src/initial/initial_aligment.cpp`
- `src/pose_graph/src/pose_graph.cpp`
- `src/config/euroc/euroc_config.yaml`

核心问题：

> 在 VINS-Mono 这个领域里，所有专家都认同的五个核心思维模型是什么？

## 1. 可观测性模型：系统能估什么，不能凭空估什么

它回答的核心问题是：单目相机和 IMU 放在一起之后，哪些状态可以被数据约束住，哪些状态必须靠运动激励、先验或外参校准才能变得可观测？

VINS-Mono 的状态不只是位姿。`Estimator` 里维护了窗口内的 `Ps/Rs/Vs/Bas/Bgs`，还维护相机-IMU 外参 `ric/tic`、特征逆深度 `para_Feature`、时间偏移 `td`。这说明系统不是在“跟踪相机轨迹”这么简单，而是在同时解释：位姿、速度、IMU 偏置、尺度、重力、特征深度、传感器外参和时间同步。

它和相邻模型的关系：
- 它决定初始化模型为什么必须先恢复尺度、重力和陀螺偏置。
- 它决定前端模型为什么需要视差，而不是只要有很多角点就行。
- 它决定优化模型里哪些变量能放开优化，哪些必须固定，例如 `estimate_extrinsic: 0/1/2` 和 `estimate_td`。

追问你自己：如果设备只原地纯旋转，单目 VIO 的哪些量会失去约束？为什么 IMU 也不能自动补齐全部信息？

## 2. 前端测量模型：把图像变成可优化的几何约束

它回答的核心问题是：图像流里的像素变化，怎样变成后端可以信任的 bearing 观测和特征轨迹？

VINS-Mono 的前端不是直接把图像送进优化器。`feature_tracker` 先用 KLT 光流跟踪特征，用 `goodFeaturesToTrack` 补点，用 mask 维持空间分布，用 `findFundamentalMat` 做 RANSAC 剔除外点，再发布归一化平面点、像素坐标、特征 ID 和速度。`FeatureManager::addFeatureCheckParallax()` 再根据跟踪数和平均视差决定当前帧是否足够关键。

它和相邻模型的关系：
- 它给投影因子提供跨帧同一特征的观测。
- 它给初始化提供足够视差的两帧相对位姿。
- 它也可能制造失败：低纹理、运动模糊、滚快门、曝光变化会先污染前端，再传到后端。

追问你自己：为什么“特征数量多”不等于“初始化容易成功”？哪个变量需要视差来约束？

## 3. IMU 预积分模型：把高频惯性测量压成窗口边

它回答的核心问题是：IMU 频率远高于相机，怎样在不把所有 IMU 原始采样都塞进优化器的情况下保留运动约束？

`IntegrationBase` 用中值积分维护 `delta_p/delta_q/delta_v`、雅可比和协方差，并记录对加速度计偏置、陀螺仪偏置的敏感性。后端在相邻关键帧之间添加 `IMUFactor`，用一个因子表达这段 IMU 积分对两个状态的约束。偏置变了以后，预积分可以通过雅可比修正，必要时 `repropagate()`。

它和相邻模型的关系：
- 它把前端的离散视觉帧连接成连续运动模型。
- 它为初始化模型提供陀螺偏置、尺度、重力估计的信息。
- 它也限制了配置质量：`acc_n/gyr_n/acc_w/gyr_w/g_norm` 不合理，优化器会在错误噪声模型下“自信地犯错”。

追问你自己：如果 IMU 噪声参数设得过小，后端会更相信谁？这种“更相信”会在轨迹上表现成什么？

## 4. 滑动窗口因子图模型：局部最优地解释所有约束

它回答的核心问题是：如何把视觉重投影误差、IMU 约束、外参、时间偏移、先验和重定位约束放进同一个优化问题？

`Estimator::optimization()` 构造 Ceres 问题：窗口内每帧有 pose 和 speed-bias 参数块，特征用逆深度参数化，视觉用 `ProjectionFactor` 或 `ProjectionTdFactor`，IMU 用 `IMUFactor`，旧窗口信息用 `MarginalizationFactor`。位姿参数用局部参数化处理四元数更新，视觉残差外面套 Cauchy loss。

它和相邻模型的关系：
- 它消费前端测量和 IMU 预积分。
- 它依赖初始化给出足够好的尺度、重力和位姿初值。
- 它输出的局部轨迹仍会漂移，所以需要边缘化和 pose graph 模型维持实时性与全局一致性。

追问你自己：为什么 VINS-Mono 不直接优化从第一帧到当前帧的所有状态？如果这么做，准确性和实时性会怎样交换？

## 5. 有界记忆与全局校正模型：局部窗口负责实时，pose graph 负责漂移

它回答的核心问题是：系统如何在计算量有限时持续运行，又如何修正长期漂移？

局部层面，`slideWindow()` 根据 `MARGIN_OLD` 或 `MARGIN_SECOND_NEW` 决定丢旧关键帧还是丢次新非关键帧；`MarginalizationInfo` 把被丢变量的信息压成先验留在窗口里。全局层面，`pose_graph` 用 DBoW/BRIEF 检测回环，建立局部边和回环边，再做 4DoF 优化，重点校正 yaw 和平移漂移。

它和相邻模型的关系：
- 它是滑动窗口优化的延续：不让局部窗口无限长。
- 它承认 VIO 是局部一致但会漂移的系统。
- 它也把“前端误匹配”和“回环误检”变成必须管理的风险。

追问你自己：为什么 pose graph 只做 4DoF，而不是完整 6DoF？这里隐含了对重力方向的什么信任？

## 五个模型之间的网络

可以把 VINS-Mono 看成一条闭环链路：

图像前端产生特征轨迹和视差判断；IMU 预积分把高频惯性测量压成相邻帧约束；初始化用视觉结构、IMU 预积分、尺度、重力和偏置把系统从“没有尺度的单目几何”拉进 metric 世界；滑动窗口因子图在有限窗口内联合优化状态；边缘化和 pose graph 一边控制计算量，一边修正长期漂移。

一句话 skeleton summary：

> VINS-Mono 的核心不是“相机定位”，而是在可观测性约束下，用前端特征、IMU 预积分、初始化、滑动窗口优化和 pose graph 共同维持一个实时、有尺度、可恢复漂移的视觉惯性状态估计器。

现在反问你：

这五个模型里，你最熟悉哪个？最陌生的是哪个？为什么你认为自己熟悉，能不能用源码里的一个函数证明？
