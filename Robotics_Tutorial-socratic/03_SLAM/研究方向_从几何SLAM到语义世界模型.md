<h1><span style="color: #e2b714">研究方向</span>：持久语义地图与长期自主导航</h1>

<span style="color: #888">申请人: 唐昊 &nbsp;|&nbsp; 申请方向: <span style="color: #5dade2">机器人自主导航</span> · <span style="color: #5dade2">语义 SLAM</span> · <span style="color: #5dade2">长期环境理解</span></span>

---

<h2><span style="color: #888">一、</span>我关心<span style="color: #e74c3c">什么问题</span></h2>

传统 SLAM 回答的是<span style="color: #888">"我在哪里，周围有什么"</span>——定位与几何建图。这个问题在过去二十年已经被解决了大半：FAST-LIO2、ORB-SLAM3 等系统在结构化环境中的定位精度已经到了厘米级。

<span style="color: #e74c3c; font-weight: bold">但我关心的不是这个问题。</span>

我关心的是：

> <span style="color: #e2b714; font-size: 1.1em"><b>当机器人进入一个从未见过的环境，它如何知道"该去哪"和"该看什么"。</b></span>

快递机器人进入陌生小区，FAST-LIO 能给它一面精确的点云墙——但这面墙<span style="color: #e74c3c">不告诉它</span>入口在哪、电梯在哪、快递柜在哪。一个建筑可能有十几个外观相似的门口，其中只有两个是真的入口。机器人需要在有限的电池和时间约束下，自己决定——<span style="color: #5dade2">下一眼看哪扇门？从哪个角度？看多少眼够？</span>

<table style="border: 1px solid #e74c3c; border-radius: 4px; padding: 8px 16px; margin: 16px 0; background: rgba(231, 76, 60, 0.08)">
<tr><td style="border: none">
<b>这不是一个定位问题。</b>这是一个<span style="color: #e74c3c; font-weight: bold">环境理解与信息获取决策</span>的问题。
</td></tr>
</table>

---

<h2><span style="color: #888">二、</span>方向<span style="color: #5dade2">从哪来</span></h2>

我大二开始做 <span style="color: #e2b714">Light-Map 语义导航</span>（MLISE 2026 一作），核心想法是用 VLM 识别门牌号，结合 OSM 地图先验和贝叶斯信念更新，让机器人在未知建筑外自主找到入口。

做完之后我发现一个问题：

> <span style="color: #e74c3c"><b>每次探索的成果没有被保存下来。</b></span>

机器人找到入口、完成了配送，然后地图就丢了——下一次任务，它又要从头找起。

这让我意识到，Light-Map 只解决了 <span style="color: #888">"一次性探索"</span> 的问题。真正难的是 <span style="color: #5dade2; font-weight: bold">持久化的环境理解</span>——地图不仅需要记住上一次探索的成果，还需要随着时间演化。三个月后桌子搬走了、门牌换了，地图必须知道自己哪些部分过期了。

<h3>SLAM 领域的<span style="color: #e2b714">三个争论</span></h3>

<table>
<tr>
<td style="border: none; vertical-align: top; padding-right: 12px"><span style="color: #e2b714; font-weight: bold">①</span></td>
<td style="border: none"><b>几何 vs 学习</b><br><span style="color: #888">核心定位应不应该端到端学习？短期几何仍是安全网，但语义感知层（VLM）已经是学习方法的天下。</span></td>
</tr>
<tr>
<td style="border: none; vertical-align: top; padding-right: 12px"><span style="color: #e2b714; font-weight: bold">②</span></td>
<td style="border: none"><b>SLAM 解决了吗</b><br><span style="color: #888">定位建图已趋于成熟，但长期环境理解与任务语义建图仍缺乏统一框架。</span></td>
</tr>
<tr>
<td style="border: none; vertical-align: top; padding-right: 12px"><span style="color: #e2b714; font-weight: bold; background: rgba(226,183,20,0.12); padding: 2px 6px; border-radius: 3px">③</span></td>
<td style="border: none"><b>地图应该长什么样</b><br><span style="color: #888">是点云的集合，还是</span> <span style="color: #5dade2">"这里有快递柜、入口在南侧、门牌 302 在走廊尽头右侧"</span> <span style="color: #888">这样的语义记忆？</span></td>
</tr>
</table>

<br>

<span style="color: #e2b714; font-weight: bold">③ 是我真正想做的。</span>它背后是一个更根本的问题：

> <span style="color: #e2b714; font-size: 1.05em"><b>地图究竟是用于定位的几何载体，还是用于决策的世界模型？</b></span>
>
> <span style="color: #888">如果是后者，那建图的目标就不是精度，而是任务效率。</span>

---

<h2><span style="color: #888">三、</span>我想做<span style="color: #5dade2">什么方向</span></h2>

<span style="color: #5dade2; font-size: 1.05em; font-weight: bold">任务导向的持久语义地图</span>
<span style="color: #888">（Task-oriented Persistent Semantic Mapping）</span>

让机器人在多次任务中持续积累和更新环境语义信息，并利用这些信息指导探索和导航决策。

<h3>三个关键特征</h3>

<table style="border: 1px solid #5dade2; border-radius: 4px; margin-bottom: 12px">
<tr><td style="border: none; padding: 12px 16px; background: rgba(93,173,226,0.06)">
<span style="color: #5dade2; font-weight: bold; font-size: 1.05em">特征 1</span><br>
<span style="color: #e2b714; font-weight: bold">地图存的不再是点云，是任务相关的语义实体。</span><br>
<br>
一个配送机器人需要知道的是：入口在哪里（置信度多少）、电梯在哪、302 房间在走廊哪个位置、哪些区域我已经探索过了。这些语义实体以概率形式存在——<span style="color: #5dade2">"入口在南侧的概率 0.8"</span>，因为 VLM 的识别本身就是不确定的。<br>
<br>
核心抽象：<span style="color: #e2b714"><b>语义因子</b></span> —— 连接机器人位姿和语义实体，用 VLM 的识别结果作为观测，更新语义实体的置信度和空间分布。这是对传统 SLAM 因子图（位姿-位姿、位姿-路标点）的扩展。
</td></tr>
</table>

<table style="border: 1px solid #5dade2; border-radius: 4px; margin-bottom: 12px">
<tr><td style="border: none; padding: 12px 16px; background: rgba(93,173,226,0.06)">
<span style="color: #5dade2; font-weight: bold; font-size: 1.05em">特征 2</span><br>
<span style="color: #e2b714; font-weight: bold">地图是会过期的，需要随时间演化。</span><br>
<br>
一个月前看到的门牌号，如果之后再没确认，不确定性应该上升——这叫 <span style="color: #5dade2">贝叶斯衰减</span>。当新观测与旧地图冲突（"这里地图说是 302，但 VLM 看到了 305"），系统应该自动触发<span style="color: #5dade2">重探索</span>。<br>
<br>
现有 SLAM 系统大多假设静态世界——Dynamic SLAM 聚焦动态目标过滤，Lifelong SLAM 关注长期地图维护，但<span style="color: #e2b714">缺乏面向任务决策的统一持久语义地图框架</span>——地图应该记住什么、忘记什么、什么时候重新相信自己，这三个问题尚未被系统性地处理。
</td></tr>
</table>

<table style="border: 1px solid #5dade2; border-radius: 4px; margin-bottom: 12px">
<tr><td style="border: none; padding: 12px 16px; background: rgba(93,173,226,0.06)">
<span style="color: #5dade2; font-weight: bold; font-size: 1.05em">特征 3</span><br>
<span style="color: #e2b714; font-weight: bold">探索决策不仅考虑当前任务，还考虑未来任务的信息收益。</span><br>
<br>
Light-Map 中视角调度的目标函数是最大化<span style="color: #888">当前</span>探索的信息增益。升级版应该考虑 <span style="color: #5dade2">长期信息增益</span>——"看这一眼，对未来所有配送任务有多大帮助"——这是一个带衰减的长期信息规划问题。<br>
<br>
因为地图是持久的，<span style="color: #e2b714">今天的观测不只是服务于今天，它也服务于未来。</span>
</td></tr>
</table>

<h3>系统架构</h3>

<div style="font-family: monospace; line-height: 2.0; padding: 8px 16px; background: rgba(255,255,255,0.03); border-radius: 4px">
<span style="color: #e2b714; font-weight: bold">先验层</span> &nbsp;←&nbsp; 任务先验与长期经验记忆（快递柜通常在一楼；这个小区入口在南侧概率 0.8）<br>
&nbsp;&nbsp;<span style="color: #888">↓ 先验约束</span><br>
<span style="color: #5dade2; font-weight: bold">语义层</span> &nbsp;←&nbsp; 门牌 302 位于 (x,y) 置信度 0.73 &nbsp;|&nbsp; 入口 A 置信度 0.80<br>
&nbsp;&nbsp;<span style="color: #888">↓ 语义观测因子</span><br>
<span style="color: #27ae60; font-weight: bold">几何层</span> &nbsp;←&nbsp; 点云 + 占用栅格 + ESDF（FAST-LIO，成熟技术，不做创新）
</div>

| 层 | 角色 | 用什么 | 创新程度 |
|---|---|---|---|
| <span style="color: #27ae60">几何层</span> | 安全网 | FAST-LIO + 占用栅格 | 成熟技术，不创新 |
| <span style="color: #5dade2">语义层</span> | 核心创新 | 语义因子图 + 贝叶斯衰减 + 冲突检测 | <b>核心贡献</b> |
| <span style="color: #e2b714">先验层</span> | 决策引导 | 任务先验 + 历史经验统计 | <b>核心贡献</b> |

---

<h2><span style="color: #888">四、</span>我有<span style="color: #27ae60">什么积累</span></h2>

这个方向不是我凭空想的。三条积累线，每条都有代码、有论文、有真机：

<table>
<tr>
<td style="border: none; vertical-align: top; padding-right: 16px; padding-top: 6px"><span style="color: #e2b714; font-weight: bold; font-size: 1.2em">01</span></td>
<td style="border: none">
<span style="color: #e2b714; font-weight: bold">语义导航 — Light-Map</span> &nbsp; <span style="color: #888">MLISE 2026 一作</span><br>
<span style="color: #888">VLM 语义识别 → 贝叶斯信念更新 → 视角调度的闭环</span><br>
核心贡献：多因子联合效用函数，统一优化视角选择、访问顺序和停止条件。<br>
<span style="color: #27ae60">SR 93.80% &nbsp; SPL 42.67%</span><br>
<br>
<span style="color: #5dade2">→ 这个工作是我做持久语义地图的直接地基。它证明了语义观测可以指导探索决策，下一步是让它有记忆。</span>
</td>
</tr>

<tr>
<td style="border: none; vertical-align: top; padding-right: 16px; padding-top: 6px"><span style="color: #e2b714; font-weight: bold; font-size: 1.2em">02</span></td>
<td style="border: none">
<span style="color: #e2b714; font-weight: bold">几何 SLAM 全栈 — RoboMaster 哨兵</span> &nbsp; <span style="color: #888">RM 全国二等奖</span><br>
<span style="color: #888">FAST-LIO + Nav2 + TEB 从 Gazebo 到真机完整部署</span><br>
8 场比赛 7 场完成自主任务。<br>
<br>
<span style="color: #5dade2">→ 我理解几何 SLAM 能做什么、不能做什么、在什么条件下会退化。所以我可以踏实地说"几何层用成熟技术"——因为我自己做过，知道它的边界在哪。</span>
</td>
</tr>

<tr>
<td style="border: none; vertical-align: top; padding-right: 16px; padding-top: 6px"><span style="color: #e2b714; font-weight: bold; font-size: 1.2em">03</span></td>
<td style="border: none">
<span style="color: #e2b714; font-weight: bold">VLA 与具身操控 — LeRobot</span> &nbsp; <span style="color: #888">ACT / π0.5 / SmolVLA 真机部署</span><br>
<span style="color: #888">多类模仿学习和 VLA 模型在低成本机械臂上的部署验证</span><br>
ACT 特定区域任务成功率 > 80%。<br>
<br>
<span style="color: #5dade2">→ 这条线让我理解"地图最终是为了行动服务的"——任务地图的输出，长期来看可以作为 VLA 模型的结构化世界表征。这是更远的目标，当前先聚焦在持久语义地图本身。</span>
</td>
</tr>
</table>

---

<h2><span style="color: #888">五、</span>为什么<span style="color: #e2b714">值得做</span></h2>

<h3><span style="color: #888">5.1 先厘清：什么是"长期自主导航"</span></h3>

传统导航、长期 SLAM、长期自主导航——三个概念经常被混用。

<div style="font-family: monospace; line-height: 1.8; padding: 8px 16px; background: rgba(255,255,255,0.03); border-radius: 4px">
<span style="color: #888">层次 1: 单次导航（传统导航）</span><br>
&nbsp;&nbsp;时间: 分钟~小时 &nbsp;|&nbsp; 状态: <span style="color: #e74c3c">无记忆</span>，任务结束清零<br>
&nbsp;&nbsp;例: FAST-LIO + Nav2 跑一次配送<br>
&nbsp;&nbsp;<span style="color: #27ae60">已解决</span><br>
<br>
<span style="color: #888">层次 2: 长期 SLAM（Long-term SLAM）</span><br>
&nbsp;&nbsp;时间: 天~月 &nbsp;|&nbsp; 状态: 地图持久化，但<span style="color: #888">地图是几何的</span><br>
&nbsp;&nbsp;例: 仓库跑了三个月，地图自动适应货架移动<br>
&nbsp;&nbsp;<span style="color: #5dade2">已有大量工作</span>，但主要是几何层面的维护<br>
<br>
<span style="color: #e2b714; font-weight: bold">层次 3: 长期自主导航（Long-term Autonomous Navigation）</span><br>
&nbsp;&nbsp;时间: 周~年 &nbsp;|&nbsp; 状态: <span style="color: #e2b714">有记忆、能遗忘、会主动验证</span><br>
&nbsp;&nbsp;例: 配送机器人服务一个小区一年——<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;入口在哪？快递柜在哪？哪条路成功率最高？<br>
&nbsp;&nbsp;<span style="color: #e74c3c">尚缺乏统一框架——这是我要做的</span>
</div>

<br>

| | <span style="color: #888">长期 SLAM</span> | <span style="color: #e2b714">长期自主导航</span> |
|---|---|---|
| 地图存什么 | 几何（点云/占用） | <span style="color: #e2b714">语义实体 + 任务经验</span> |
| "长期"指什么 | 地图随物理环境演化 | <span style="color: #e2b714">知识随任务积累</span> |
| 核心操作 | 更新/删除几何体素 | <span style="color: #e2b714">更新/衰减/冲突检测语义信念</span> |
| 评价指标 | 定位精度不随时间退化 | <span style="color: #e2b714">任务效率随任务次数提升</span> |
| 遗忘的意义 | 过时的几何该删 | <span style="color: #e2b714">过时的信念该衰减，不确定时主动去看</span> |

<p style="border-left: 3px solid #e2b714; padding-left: 16px">
<b>长期 SLAM 解决"地图旧了怎么办"。长期自主导航解决"机器人怎么越跑越聪明"。</b><br>
三个特征合起来就是：机器人服务同一个环境越久，它的内部世界模型就越准，主动探索的需求就越少，任务执行就越高效。<span style="color: #e2b714">"长期自主"不是跑得久，是越跑越不需要人管。</span>
</p>

<h3><span style="color: #888">5.2 地图会过期</span></h3>

今天几乎所有 SLAM 系统都默认一个假设：<span style="color: #e74c3c">世界是静态的。</span>建一次图，用一辈子。

但现实是——桌子会搬，门牌会换，快递柜会挪。三个月前建的图，今天可能已经部分失效。目前几乎没有 SLAM 系统有<span style="color: #e2b714">"遗忘"机制</span>——它们要么假设世界不变，要么干脆不用持久地图，每次从头建。

<span style="color: #e2b714; font-weight: bold">什么时候相信旧地图？什么时候触发重探索？旧观测和新观测冲突时信哪个？</span>这三个问题是长期自主导航的基础问题，但现有工作多聚焦于动态目标过滤或地图增量更新，缺少统一的语义地图生命周期管理框架。

<h3><span style="color: #888">5.3 时机：语义感知成本正在归零</span></h3>

VLM 的快速发展让语义感知的成本急剧下降——两年前识别门牌号还是不可靠的，现在已经到了可以工程化的阶段。这意味着 SLAM 不再需要"猜"物体的语义——<span style="color: #5dade2"><b>它可以问了。</b></span>

<p style="border-left: 3px solid #e2b714; padding-left: 16px; margin: 16px 0">
但这个"问"的结果<b>怎么存、怎么随时间更新、怎么在信息过时后触发重探索</b>——目前<span style="color: #e74c3c">没有现成的框架</span>。<br>
这正是我想做的。
</p>

<h3><span style="color: #888">5.4 更大图景（长期愿景）</span></h3>

长期来看，如果机器人能持续积累"入口在哪、快递柜在哪、哪条路配送成功率最高"这样的任务知识，它就不再只是做定位——它在构建一个越来越准确的环境认知。持久语义地图是这个方向的第一步：先把地图随时间演化这件事做对。

> 我做的不只是"语义 SLAM"。我关心的是<span style="color: #5dade2; font-weight: bold">机器人如何在开放世界中长期地、可靠地积累环境知识</span>。

<h3><span style="color: #888">5.5 让导航有记忆</span></h3>

传统导航系统（包括 VLN）都是<span style="color: #e74c3c">失忆的</span>——每次任务结束，记忆清零。下一次任务从头感知、从头理解、从头决策。这就像一个有超强感知能力但没有海马体的人——他能看清一切，但什么都记不住。

<span style="color: #e2b714; font-weight: bold">持久语义地图，本质上是给导航系统装一个长期记忆模块。</span>上次探索过的建筑、找到过的入口、确认过的门牌号，不需要重新发现。

这和大语言模型从"无状态 function"到"有状态 agent"的范式转换是同一件事——LLM 有了 RAG 和长期上下文之后，从一次对话就忘变成了能记住用户偏好的 agent。导航系统也一样：<span style="color: #5dade2">几何 SLAM 给了它"现在在哪"的感知力，语义记忆给了它"这里以前是什么"的认知力。</span>

<table>
<tr>
<td style="padding: 10px 16px; background: rgba(231,76,60,0.06); border-right: 1px solid rgba(255,255,255,0.05)">
<span style="color: #e74c3c; font-weight: bold">传统导航</span><br>
<span style="color: #888">stateless function</span><br>
每次任务：感知 → 决策 → 执行 → 遗忘
</td>
<td style="padding: 10px 16px; background: rgba(39,174,96,0.06)">
<span style="color: #27ae60; font-weight: bold">持久语义导航</span><br>
<span style="color: #888">stateful agent</span><br>
每次任务：感知 → 查记忆 → 决策 → 执行 → 更新记忆
</td>
</tr>
</table>

<br>
从 VLN 的角度看这个关系最直观：当前的 VLN 是一条指令 → 一次导航 → 结束。有了持久语义地图——
- <span style="color: #888">第一次</span>收到"去 302"，agent 需要探索和建图
- <span style="color: #5dade2">第十次</span>收到"去 302"，直接查记忆走
- <span style="color: #e2b714">如果地图说 302 在走廊右侧（置信度 0.73），但 VLM 在左侧看见了 302</span>——冲突检测触发，地图更新

<p style="border-left: 3px solid #e2b714; padding-left: 16px">
<b>持久语义地图是 VLN agent 的长期记忆底座。</b>VLN 解决"理解指令并执行"的问题，持久地图解决"记住上一次执行中了解到的环境信息"的问题。两者共享同一套语义实体（门牌号、入口、房间功能），但职责不同。
</p>

---

<h2><span style="color: #888">六、</span>我想跟<span style="color: #5dade2">什么样的导师</span>做</h2>

<table>
<tr style="background: rgba(93,173,226,0.06)">
<td style="border: none; padding: 10px 16px"><span style="color: #5dade2; font-weight: bold">如果导师做 SLAM</span></td>
<td style="border: none; padding: 10px 16px">在传统因子图的基础上扩展<span style="color: #e2b714">语义因子</span>，把 SLAM 的能力边界从几何定位推到语义环境理解</td>
</tr>
<tr style="background: rgba(226,183,20,0.06)">
<td style="border: none; padding: 10px 16px"><span style="color: #e2b714; font-weight: bold">如果导师做具身智能/VLA</span></td>
<td style="border: none; padding: 10px 16px">带几何 SLAM 工程能力 + 持久语义地图，为具身智能提供<span style="color: #e2b714">可跨任务复用的环境记忆</span></td>
</tr>
<tr style="background: rgba(39,174,96,0.06)">
<td style="border: none; padding: 10px 16px"><span style="color: #27ae60; font-weight: bold">如果导师做自主导航/规划</span></td>
<td style="border: none; padding: 10px 16px">Light-Map 直接扩展：从<span style="color: #888">单次探索</span>到<span style="color: #27ae60">持久记忆</span>，从纯导航到<span style="color: #27ae60">信息获取决策</span></td>
</tr>
</table>

<br>

<p style="text-align: center">
<span style="color: #e2b714; font-size: 1.05em"><b>三条线的共同点是同一个问题：</b></span><br>
<span style="color: #e2b714; font-size: 1.1em"><b>如何让机器人在开放世界中持久地积累、更新<br>和利用环境知识，支撑长期自主导航。</b></span>
</p>

---

<span style="color: #888; font-size: 0.9em">本文基于对 SLAM 领域的系统分析（<a href="https://laohao78.github.io/fangfalun/2026-05-21-ai-socratic-learning/">苏格拉底学习法三步骤</a>），核心判断来自 SLAM 领域当前最活跃的三个争议，尤其是"地图究竟是几何载体还是决策世界模型"这一根本问题。</span>
