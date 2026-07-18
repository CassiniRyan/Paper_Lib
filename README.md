# Paper 整理

# List 1：Perceptive Locomotion / Parkour / Humanoid

## A Hybrid Autoencoder for Robust Heightmap Generation from Fused Lidar and Depth Data for Humanoid Robot Locomotion



## AME\-2: Agile and Generalized Legged Locomotion via Attention\-Based Neural Map Encoding

## ANYmal Parkour Learning Agile Navigation for Quadrupedal Robots

## APEX: Learning Adaptive High\-Platform Traversal for Humanoid Robots

## Architecture Is All You Need Diversity\-Enabled Sweet Spots for Robust Humanoid Locomotion

## Attention\-Based Map Encoding for Learning Generalized Legged Locomotion

## BeamDojo Learning Agile Humanoid Locomotion on Sparse Footholds

## Collision\-Free Humanoid Traversal in Cluttered Indoor Scenes

## CReF: Cross\-modal and Recurrent Fusion for Depth\-conditioned Humanoid Locomotion

## Deep Whole\-body Parkour

## ⭐Distillation\-PPO A Novel Two\-Stage Reinforcement Learning Framework for Humanoid Robot Perceptive Locomotion

### 核心问题

这篇论文关注人形机器人的感知运动控制，目标是让人形机器人在台阶、斜坡、沟壑和不规则地形上稳定行走。核心问题不是单纯训练一个 blind locomotion policy，而是如何把地形感知信息有效接入强化学习控制器，同时避免纯模仿学习和纯端到端强化学习各自的问题。

传统两阶段方法通常先训练 teacher policy，再用 DAgger 或 supervised loss 让 student policy 模仿 teacher。这种方法训练稳定，但 student 很难超过 teacher，并且 teacher 在仿真中的 privileged information 可能不真实。端到端 RL 方法允许 student 自己探索，但在 POMDP 下训练难度高，视觉或地形输入维度大，训练不稳定。本文提出 Distillation\-PPO，即在 student 阶段同时使用 teacher supervision 和 PPO reward，让 student 既不偏离 teacher，又可以继续通过 RL 优化自己的行为。

### 整体训练流程如何工作

D\-PPO 分为两个阶段。

第一阶段训练 teacher policy。teacher 在仿真中使用干净、准确的地形 scan dots 和本体状态信息进行训练。这里的 scan dots 来自仿真器中的高程图，因此地形信息没有真实感知噪声。teacher 的作用是先学到一个相对可靠的地形适应策略，例如遇到台阶时抬腿，遇到斜坡时调整姿态，遇到障碍时改变步态。

第二阶段训练 student policy。student 使用带噪声的 scan dots 和历史本体状态，模拟真实机器人部署时的感知输入。student 的 loss 不是单纯模仿 teacher，而是同时包含 distillation loss 和 PPO loss。distillation loss 让 student 的动作接近 teacher，起到正则化和引导作用；PPO loss 让 student 继续根据环境 reward 自己探索和优化，从而避免被 teacher 的错误动作完全限制。

整体损失为：

Ltotal = α Ldistillation \+ β LPPO

其中论文实验中设置 α = 0\.5，β = 0\.5。也就是说，student 的训练一半来自 teacher action supervision，一半来自 RL reward。这个设计的关键是：teacher 提供训练稳定性，PPO 提供性能上限和对噪声输入的适应能力。

### Teacher 如何使用地形信息

teacher 输入包括本体感受信息、历史状态和干净的 scan dots。scan dots 是从 elevation map 中采样得到的局部地形高度网格，论文中将其表示为 m ∈ R441，也就是以机器人为中心的 21 × 21 高度采样点。每个采样点代表机器人附近某个位置的地形高度。

teacher 使用 Conv1D 对 scan dots 进行压缩，将 441 维地形高度向量编码成 32 维 perception latent。同时，系统还使用一个 history encoder 将 50 帧历史状态压缩成 32 维 history latent。然后将 perception latent、history latent 和当前本体状态一起输入 MLP，输出动作。

这个设计的作用是把高维地形高度压缩成低维地形特征，让策略网络可以根据局部地形提前调整步态。例如，在平地上保持较直膝姿态以获得更大视野；遇到障碍或台阶时弯曲膝盖、提高摆腿高度，并通过手臂摆动保持平衡。论文图 1 展示了 Tien Kung 在高平台、斜坡、小沟和障碍地形上的动作差异。

### Student 如何处理真实感知噪声

student 的网络结构与 teacher 对称，但输入的 scan dots 被加入噪声。这样做是为了模拟真实部署时 elevation map 误差，例如深度相机噪声、LIO 位姿误差、遮挡、腿部遮挡、点云稀疏和高程图融合误差。

论文指出，student 训练时噪声强度很关键。如果噪声太大，策略会倾向于不信任感知，退化成通过脚接触来处理地形；如果噪声太小，仿真和真实感知之间的 gap 太大，实机部署容易失败。因此，D\-PPO 用 teacher action 对 student 做约束，使 student 在噪声输入下仍然朝着合理地形适应策略学习；同时用 PPO reward 让 student 在 noisy POMDP 中继续优化，而不是机械复制 teacher。

这也是本文区别于普通 DAgger 的地方。普通 DAgger 往往只让 student 拟合 teacher action，而 D\-PPO 允许 student 在 teacher 正则化下继续通过 RL 改善动作。

### scan dots 如何由高程图得到

论文使用 elevation map 表示局部地形，再从高程图中采样 scan dots 输入策略。具体做法是：在以机器人为中心的局部 elevation map 上，按一米间隔采样地形高度，得到 441 维 scan dots 向量。这个向量相当于一个局部地形高度网格，编码了机器人前后左右一定范围内的台阶、斜坡、沟壑和障碍物。

在仿真中，teacher 直接从 simulator 获取精确 scan dots；student 则使用加噪后的 scan dots。实机中，scan dots 来自真实 perception pipeline：Fast\-LIO2 输出机器人 pose，深度相机生成点云，点云和 pose 一起输入 GPU elevation mapping，最后从重建出的高程图中采样 scan dots。论文图 3 和图 4 展示了机器人在高平台、斜坡、台阶和不规则地形上的高程重建结果。

因此，scan dots 不是原始深度图，也不是原始 LiDAR 点云，而是经过位姿估计、高程图融合和空间采样后的低维地形输入。

### 实机感知系统如何工作

实机感知链路包括 LIO、深度相机、遮挡剔除和 elevation mapping。

首先，机器人使用 Livox MID\-360 和 Xsens IMU，通过 Fast\-LIO2 获取机器人位姿。这个 pose 用于把传感器点云变换到全局或局部 map frame。然后，系统使用 Orbbec 355L 深度相机获取 depth image，并将 depth image 转换成 point cloud。由于人形机器人的腿部会遮挡深度相机视野，论文使用关节角 q 和 forward kinematics 估计机器人身体 bounding box，并在 depth image 中剔除被身体遮挡的区域，避免把自己的腿错误写入高程图。

之后，系统将 point cloud 和 pose 输入 elevation mapping framework。每个 grid cell 使用一维 Kalman filter 更新高度。高度更新公式与前面 GPU elevation mapping 相同：新点不会直接覆盖旧高度，而是根据点测量方差和 cell 当前方差加权融合。点测量噪声采用距离相关模型 σp² = αd d²，即离传感器越远，高度测量越不可信。

最后，控制器从高程图中采样 scan dots，作为 student policy 的地形输入。

### 高程图噪声在训练中如何建模

论文在 student policy 训练阶段，对每个 grid 添加 Gaussian noise，以缩小仿真高程图和真实传感器高程图之间的差距。这个做法比较直接，但从真实系统角度看，它只建模了部分感知误差。

真实高程图误差至少来自以下环节：Fast\-LIO2 位姿误差、深度相机噪声、腿部遮挡剔除误差、点云投影误差、Kalman 高度融合延迟、距离相关测量噪声、遮挡导致的空洞，以及 scan dots 采样误差。论文主要通过 Gaussian noise 近似这些误差，并依赖 D\-PPO 的 teacher supervision \+ PPO fine\-tuning 提高 student 对噪声的鲁棒性。

如果复现或扩展这篇工作，建议不要只加独立高斯噪声，还应加入结构化噪声，例如高度图 patch dropout、z drift、roll/pitch 投影误差、scan dots 延迟、局部空洞、边缘模糊和腿部遮挡残留。这些更接近实机 elevation map 的输出分布。

### 网络结构如何组织

teacher 和 student 都包含几个编码模块。

scan encoder 用 Conv1D 压缩 scan dots，输出 perception latent。history encoder 使用历史本体状态，输出 history latent。论文图 2 中还显示了 velocity、friction、scan 和 observation 相关 latent 表示，其中 history information 被大量用于估计不可直接观测的状态。actor 网络根据 latent 和本体状态输出动作，critic 网络输出 value。

student 初始化时继承 teacher 的结构和部分参数，这类似 self\-distillation。这样 student 不是从零开始训练，而是在 teacher 已学到的地形适应策略基础上，继续适应 noisy perception。训练过程中 student 同时接收 PPO gradient 和 distillation gradient，因此 policy 和 history encoder 都会被进一步优化。

图 2 的关键点是：teacher 和 student 结构对称，teacher action 作为监督信号，student action 进入 RL environment 收集 rollout，最终用 Ldistillation \+ LPPO 联合更新。

### 动作空间如何定义

论文中 humanoid 的 action 是 18 维向量，表示 18 个驱动关节的期望位置。策略不是直接输出 torque，而是输出每个 actuated joint 的 target position，底层控制器再根据目标关节位置执行。

这种 action design 在 sim\-to\-real 中相对常见，因为位置目标比 torque 输出更容易稳定部署，也更容易利用机器人已有低层伺服控制能力。对于人形机器人，动作包括腿部和手臂关节，因此策略可以通过手臂动作辅助平衡，而不仅仅控制双腿。

### observation 如何组成

策略 observation 包含本体状态、命令、周期信号和地形 scan dots。

本体状态包括当前线速度和角速度，x、y、yaw 方向的平均速度，pelvis 相对 local frame 的朝向，腿部和手臂各关节的位置与速度，以及上一时刻 action。命令 ct = \(vx, vy, ωyaw\)，用于指定期望速度和转向。周期信号包括 gait cycle 的 sin/cos、swing phase ratio ρ 等，用于构建周期步态。地形输入是 scan dots，用于感知台阶、斜坡和障碍。

这种 observation 组合体现了本文的控制思路：基础步态由周期信号和本体状态维持，复杂地形适应由 scan dots 提供提前感知，历史信息用于估计速度、摩擦或隐藏状态。

### reward 如何设计

论文的 reward 包括 periodic reward、tracking reward 和 regularization reward。

periodic reward 用于生成稳定步态。它根据左右脚的 swing phase 和 stance phase 分别约束脚部速度和脚底接触力。在 stance phase，希望脚稳定接触地面，因此约束脚部速度；在 swing phase，希望脚离地摆动，因此约束脚底接触力。论文使用 Von Mises distribution 的期望来构造相位指示函数，使步态相位更加平滑。

tracking reward 用于让机器人跟踪期望速度命令，包括 x、y 和 yaw 方向速度。形式上是期望速度与实际速度误差的指数惩罚。

regularization reward 用于提高 sim\-to\-real 稳定性，包括 action differential、DoF limits、DoF velocity、DoF acceleration、arm DoF penalty、orientation differential、torso yaw 和 torques 等。它们限制动作变化、关节速度、关节加速度、力矩和身体姿态，避免策略在仿真中学到过激或不安全动作。论文 Table I 列出了这些正则项。

### 为什么 D\-PPO 比单纯蒸馏或单纯 RL 更适合

单纯蒸馏训练容易，但 student 只能模仿 teacher。遇到 teacher 没见过的场景，或者 teacher 在仿真 privileged information 下学到的动作不适合真实感知输入时，student 没有通过 reward 修正行为的能力。因此，噪声鲁棒性较差。

单纯 RL 可以让 student 直接在 noisy perception 下优化，但视觉 / 地形输入使 POMDP 难度变高，探索效率低，训练不稳定。特别是对于人形机器人，状态维度高、动力学不稳定、摔倒代价高，纯 RL 更难收敛。

D\-PPO 把二者结合。teacher supervision 降低探索难度，PPO reward 允许 student 继续优化并适应噪声。论文 Table II 总结为：Only Distillation 训练容易但鲁棒性差，Only RL 训练困难但鲁棒性中等，D\-PPO 训练容易、控制性能好、噪声鲁棒性好。

### 实验结果如何说明方法有效

论文在 Isaac Gym 中使用 4096 个并行环境，在单张 RTX 4090 上训练 teacher 和 student。实机机器人为 Tien Kung，人形机器人使用 Livox MID\-360、Xsens IMU 和 Orbbec 355L depth camera 构建感知系统。

实机实验展示了机器人在高平台、台阶、斜坡、小沟和不规则地形上的行走能力。图 3 展示机器人跨越高平台时，不同运动阶段对应的地形重建和姿态变化；图 4 展示不同地形下的高精度地形重建；图 5 展示机器人在楼梯和斜坡上的行走过程。结果表明，D\-PPO 训练出的策略能够根据 scan dots 调整步态和姿态，而不是只依赖脚底接触后的被动反应。

### 对仿真和复现的启发

如果复现这篇方法，关键不是只实现 D\-PPO loss，而是要保证训练时的 scan dots 分布接近实机部署时的 scan dots 分布。teacher 可以使用干净地形，但 student 必须看到接近真实 elevation map 的噪声。

更合理的仿真流程是：

仿真地形 → 生成局部 elevation map → 加入感知噪声、位姿误差、遮挡空洞和延迟 → 采样 scan dots → 输入 student policy。

如果要更接近实机，应进一步模拟：

Fast\-LIO2 pose drift 和 jitter；

depth camera 距离相关噪声；

腿部遮挡和遮挡剔除不完全；

高程图 Kalman 融合带来的时间滞后；

台阶边缘的高度模糊；

scan dots 的随机缺失和空间错位；

感知更新频率低于控制频率导致的 map latency。

尤其对人形机器人，腿部遮挡是很重要的问题。仿真中如果不模拟自遮挡，student 会过度依赖完美前方地形；实机中一旦腿部进入深度图，scan dots 会出现错误或空洞，策略可能误判地形高度。

### 总结评价

这篇论文的核心贡献是把 teacher\-student distillation 和 PPO fine\-tuning 结合起来，用于人形机器人的感知运动控制。它不是简单地让 student 模仿 teacher，而是把 teacher action 作为正则项，同时保留强化学习的 reward\-driven exploration。这样既提高了训练稳定性，又避免 student 被 teacher 的能力上限完全限制。

技术上，本文使用 elevation map 采样 scan dots 作为地形输入，通过 Conv1D 压缩地形高度，通过 history encoder 提取隐状态，再结合周期步态 reward 和正则化 reward 训练人形机器人策略。实机系统中，Fast\-LIO2、深度相机、身体遮挡剔除和 GPU elevation mapping 共同生成 scan dots。

局限在于，论文对感知噪声的建模仍然较简单，主要是在 grid 上加入 Gaussian noise。真实高程图噪声更复杂，包括遮挡、延迟、位姿漂移、边缘模糊、腿部误投影和高度图残影。如果要进一步提高 sim\-to\-real 鲁棒性，student 训练中的感知扰动应更接近真实 perception pipeline，而不仅是独立高斯噪声。



## DPL: Depth\-only Perceptive Humanoid Locomotion via Realistic Depth Synthesis and Cross\-Attention Terrain Reconstruction

## Extreme Parkour with Legged Robots

## FastStair Learning to Run Up Stairs with Humanoid Robots

## Gait\-Adaptive Perceptive Humanoid Locomotion with Real\-Time Under\-Base Terrain Reconstruction

## Gallant Voxel Grid\-based Humanoid Locomotion and Local\-navigation across 3D Constrained Terrains

## GaussGym An open\-source real\-to\-sim framework for learning locomotion from pixels

## GeoLoco: Leveraging 3D Geometric Priors from Visual Foundation Model for Robust RGB\-Only Humanoid Locomotion

## GRAIL: Generating Humanoid Loco\-Manipulation from 3D Assets and Video Priors

## High\-speed control and navigation for quadrupedal robots on complex and discrete terrain \(2025\.06\)\(Science Robotics 2025\)

## ⭐Hiking in the Wild A Scalable Perceptive Parkour Framework for Humanoids



## Humanoid Parkour Learning

## Hybrid Internal Model Learning Agile Legged Locomotion with Simulated Robot Response

## Learning Agile Locomotion on Risky Terrains

## Learning Autonomous and Safe Quadruped Traversal of Complex Terrains Using Multi\-Layer Elevation Maps

## ⭐Learning Humanoid Locomotion with Perceptive Internal Model

### 0\. 背景

这篇文章关注**人形机器人在复杂地形上的感知式运动控制**。相比四足机器人，人形机器人自由度更高、支撑面积更小、稳定性更差，因此仅依赖本体感知的 blind policy 很难稳定通过连续楼梯、高台和 gap 等地形。作者提出 **Perceptive Internal Model, PIM**，核心思想是不用 RGB、深度图或原始点云直接控制，而是构建机器人中心的 **elevation map**，再从局部高度图中采样高度输入策略。

### 1\. 方法

PIM 建立在 **Hybrid Internal Model, HIM** 基础上。HIM 原本只用本体历史观测预测机器人线速度和下一步状态 latent；PIM 进一步把当前地形高度观测加入 state prediction，使模型能根据地形变化预测机器人下一步动力学状态。

策略输入包括速度命令、本体感知、上一时刻动作、局部 elevation samples，以及 PIM 输出的速度估计和 latent。训练时 critic 可以访问仿真中的 privileged information，例如真实线速度；部署时只使用 actor。论文强调这是**单阶段训练**，不需要 teacher\-student、多阶段蒸馏或视觉 encoder 训练，RTX 4090 上约 3 小时可完成训练。

### 2\. 网络结构

网络主要由 **Perceptive Source Encoder、Target Encoder、Policy Network 和 Critic Network** 组成。

**Source Encoder** 输入本体历史和当前 elevation map，输出预测线速度和 source latent。**Target Encoder** 输入下一时刻本体观测，输出 target latent。训练时通过线速度回归和类似 SwAV/HIM 的对比学习，使 source latent 与同轨迹的 target latent 对齐。

**Policy Network** 接收当前观测和 PIM 输出，生成关节动作。**Critic Network** 负责 value estimation，并在训练阶段使用 privileged information。论文第 4 页的框架图展示了这一路径：感知与本体观测进入 PIM，PIM 输出状态估计，再接入 policy 和 critic 做 PPO 优化。

### 3\. Reward 构成

Reward 设计围绕速度跟踪、稳定性、能耗、动作平滑、关节限制和足端接触展开。主要包括线速度/角速度跟踪，z 方向速度惩罚、roll/pitch 角速度惩罚、身体朝向和高度约束，关节加速度、功率、力矩、动作变化率和平滑性惩罚，以及关节位置/速度/力矩限位。

足端相关 reward 是本文比较关键的部分，包括 **feet clearance、feet stumble、feet slip、feet lateral distance、feet ground parallel、feet parallel、feet contact force 和 contact momentum**。其中 **feet lateral distance** 避免双脚横向距离过近，**feet ground parallel** 通过脚底多个采样点到地面的高度方差，鼓励脚底与地面平行。第 5 页 Table I 给出了完整 reward 列表和权重。

### 4\. Perception：雷达、摄像头还是 RGB？

本文使用的是 **elevation map**，不是 RGB，也不是直接 depth image。训练时直接使用仿真中的 ground\-truth obstacle height map，因此不需要渲染深度图，也避免了 sim depth 和 real depth 的 domain gap。

部署时有两种传感器配置：一种是 **Mid\-360 LiDAR \+ FAST\-LIO/FAST\-LIO2**，同时提供点云和 LiDAR\-inertial odometry；另一种是 **RealSense T265 \+ D435**，T265 提供 odometry，D435 提供 depth/point cloud。论文指出两者在稳定运动时都能构建可用 elevation map，但 LiDAR 方案在快速或不规则运动中更鲁棒。

具体输入不是完整地图，而是在机器人中心、重力对齐坐标系下，从约 **0\.8 m × 1\.2 m** 区域采样 **96 个高度点**。

### 5\. 步态约束

这篇文章**不使用 AMP**，也**不使用 motion capture mimic**，没有 DeepMimic 式的人类动作追踪或 adversarial motion prior。它也没有显式 foothold planner；落脚选择是 policy 根据 elevation map 隐式学出来的。

额外约束主要来自两个训练技巧：**action curriculum** 和 **symmetry regularization**。Action curriculum 在训练早期限制手臂、腰部等不关键关节的动作范围，随后逐渐放开，以降低高自由度探索难度。Symmetry regularization 对本体观测、感知观测和动作做左右镜像约束，使 policy 和 value 在镜像状态下保持一致，从而提高步态协调性。

### 6\. 总结与评价

总体来看，PIM 的价值在于选择了一个适合 locomotion 的中间感知表示：机器人中心的 elevation map，并把它用于 internal model 的状态预测，而不只是简单拼接到 policy observation。相比端到端视觉策略，它更轻量、训练成本更低、sim\-to\-real gap 更小，也更适合真实机器人低延迟部署。

实验上，方法在 Unitree H1 和 Fourier GR\-1 上完成了 15 cm 连续楼梯、高台、gap、斜坡等任务。第 6 页实验图显示 H1 可以连续上楼梯、跳上木箱平台并跨越 gap，GR\-1 也能通过类似复杂地形，说明方法具有一定跨平台泛化能力。

局限也比较明确。首先，它高度依赖 elevation mapping 和 odometry 质量；如果 FAST\-LIO 漂移、点云地面分割失败或地图更新延迟过大，policy 输入会直接退化。其次，elevation map 主要描述几何高度，对低摩擦、软地面、动态障碍物和语义信息表达有限。第三，虽然实验展示了复杂地形通过能力，但它仍然是局部地形反应式控制，不包含长程导航、全局路径规划或显式安全验证。最后，尽管训练流程比多阶段视觉方法简单，reward、curriculum 和地形生成仍然有明显工程调参成本。

一句话概括：这篇文章不是靠模仿人类动作或复杂视觉网络做人形 parkour，而是用 **LiDAR/elevation map \+ PIM 状态预测 \+ PPO whole\-body policy** 形成一个较轻量、部署友好的人形复杂地形 locomotion 框架；优点是高效、直接、sim\-to\-real 友好，缺点是依赖地图质量，且缺少显式规划和安全保证。

## Learning Perceptive Humanoid Locomotion over Challenging Terrain

## Learning robust perceptive locomotion for quadrupedal robots in the wild

## Learning Robust Autonomous Navigation and Locomotion for Wheeled\-Legged Robots

## Learning Vision\-Based Bipedal Locomotion for Challenging Terrain

## Legged Locomotion in Challenging Terrains using Egocentric Vision

## Locomotion Beyond Feet

## MeshMimic \- Geometry\-Aware Humanoid Motion Learning through 3D Scene Reconstruction

## Mind Your Steps: A General Learning Framework for Accurate Humanoid Foothold Tracking

## MoRE: Mixture of Residual Experts for Humanoid Lifelike Gaits Learning on Complex Terrains

## Now You See That Learning End\-to\-End Humanoid Locomotion from Raw Pixels

## Omni\-Perception Omnidirectional Collision Avoidance for Legged Locomotion in Dynamic Environments

## Parkour in the Wild Learning a General and Extensible Agile Locomotion Policy Using Multi\-expert Distillation and RL Fine\-tuning

## Perceptive Humanoid Parkour Chaining Dynamic Human Skills via Motion Matching

## PIE: Parkour with Implicit\-Explicit Learning Framework for Legged Robots

## Real\-Time Polygonal Semantic Mapping for Humanoid Robot Stair Climbing

## Robot Parkour Learning

## RPL: Learning Robust Humanoid Perceptive Locomotion on Challenging Terrains

## SSR: Scaling Surefooted and Symmetric Humanoid Traversal to the Open World

## START: Traversing Sparse Footholds with Terrain Reconstruction

## TAGA: Terrain\-aware Active Gaze Learning for Generalizable Agile Humanoid Locomotion

## TTT\-Parkour: Rapid Test\-Time Training for Perceptive Robot Parkour

## VB\-Com: Learning Vision\-Blind Composite Humanoid Locomotion Against Deficient Perception

## Visual Imitation Enables Contextual Humanoid Control

## Walk the PLANC Physics\-Guided RL for Agile Humanoid Locomotion on Constrained Footholds

## Walking with Terrain Reconstruction Learning to Traverse Risky Sparse Footholds





# List 2: Elevation / Mapping / LIO

## ⭐Elevation Mapping for Locomotion and Navigation using GPU

### 0\. 核心问题

这篇论文的关键不是简单地“使用 LiDAR 生成高程图”，而是说明真实机器人上点云如何经过一系列滤波、融合、清理和后处理，最后变成控制器可用的局部 elevation map。这个过程会引入非常具体的感知误差：远距离高度噪声、遮挡空洞、位姿漂移、地图残影、边缘模糊、错误填补和高度层偏差。因此，仿真中如果只给控制器输入完美 heightfield，或者只加独立高斯噪声，都不能模拟真实部署时的高程图分布。

### 1\. 点云如何进入高程图

系统输入是传感器点云和机器人位姿估计。每一帧点云首先根据机器人当前 pose 和传感器外参，从 sensor frame 变换到 map frame。也就是说，高程图中的高度不是传感器直接测得的，而是经过位姿估计投影后的结果。

这个步骤非常关键，因为只要 base pose、LiDAR 外参或时间同步有误差，点云投影到地图后就会产生系统性高度误差。例如，z 方向 odometry 漂移会让整张地图出现上下错位；roll / pitch 误差会把平地投影成斜坡；时间延迟会让机器人运动时点云与实际地形错位。

因此，高程图误差的第一来源不是 sensor noise，而是：

点云测量误差 \+ 位姿估计误差 \+ 外参误差 \+ 时间同步误差。

### 2\. 每个点如何更新一个 cell

点云被变换到 map frame 后，每个点根据其 x\-y 坐标落入一个二维 grid cell。该点的 z 值作为该 cell 的新高度观测。系统不会直接用新点覆盖旧高度，而是做加权融合：

h\_new = weighted average\(h\_old, pz\)

权重由两个不确定性决定：一个是地图中该 cell 已有高度估计的不确定性，另一个是当前点测量的不确定性。当前点的测量方差被建模为：

σp² = αd d²

其中 d 是点到传感器的距离。这个公式意味着：近处点对地图影响大，远处点对地图影响小。随着同一个 cell 被多次观测，该 cell 的方差会下降，地图对新观测会变得更“固执”。如果一个 cell 长时间没有被观测，它的方差会随时间增加，表示旧信息逐渐不可靠。

这个过程带来的真实现象是：高程图有时间记忆。它不是每一帧都重新生成，而是在历史地图上持续融合。因此它既会抑制随机噪声，也会保留旧错误，产生残影和更新延迟。

### 3\. 异常点如何被拒绝

当新点的高度和当前 cell 高度差异过大时，系统不会立即相信该点，而是通过 Mahalanobis distance 做 outlier rejection。也就是说，如果当前 cell 已经有较稳定的高度估计，一个突然高很多或低很多的点可能会被拒绝。

这样做可以防止孤立噪声点污染地图，但也会带来副作用。例如新出现的障碍、台阶边缘或动态物体不会总是被立刻写入地图。地图是否更新，取决于新点和旧高度之间的差异、cell 当前方差、点测量方差以及 outlier threshold。

在仿真中，这意味着不能把高程图看作真实地形的即时观测，而要模拟“新信息进入地图需要一定置信度”的过程。

### 4\. 多个高度落入同一 cell 时如何处理

2\.5D 高程图每个 cell 只能保存一个高度，但真实点云中，同一个 x\-y cell 可能包含多个 z 值。例如墙面、楼梯边缘、栏杆、桌腿、石块边缘都会出现这种情况。如果直接平均，垂直结构会被压成错误高度。

论文的处理方式是：统计同一个 cell 内点的数量。如果一个 cell 内出现大量点，说明这里可能是垂直边缘或墙面。为了避免高度被低处点拉低，系统会在特定条件下忽略低于当前高度估计的点，从而让边缘更锐利。

这说明真实高程图对台阶和墙边的处理不是几何真实投影，而是带有启发式规则。仿真中应模拟台阶边缘的 aliasing：有时边缘被抹平，有时高度偏高，有时局部 cell 出现尖峰。

### 5\. 高度漂移如何补偿

如果机器人位姿估计存在 z 方向漂移，新一帧点云会整体高于或低于旧地图。论文不是仅更新局部 cell，而是先计算新点云与当前地图之间的平均高度误差，然后把整张 elevation layer 整体上移或下移。

具体做法是：对新点云中的点，查找其投影 cell 的当前地图高度，计算两者高度差。为了避免台阶、墙面或障碍物影响估计，只在 traversability 较高、相对平坦的区域统计误差。最后取平均误差，对整张地图做高度校正。

这个机制会带来一个很重要的仿真现象：地图高度可能不是平滑变化，而是被周期性整体校正。也就是说，policy 看到的高度图可能突然整体上移或下移几厘米。对于腿式机器人，这种变化会影响 base height reference、落脚高度估计和 swing foot clearance。

仿真中应加入两类误差：

第一类是慢变 z drift，使地图逐渐偏离真实高度；

第二类是 drift compensation correction，使地图在某些时刻被整体拉回，产生小幅跳变。

### 6\. 旧障碍如何被清除

真实环境中动态障碍移动后，如果只依赖 Kalman 更新，旧障碍会在地图中保留较长时间。论文用 ray casting 做 visibility cleanup。

具体做法是：对每个观测点，从传感器原点向该点连一条 ray。ray 经过的空间应该是空的，因为传感器光线成功穿过这些位置到达了最终测量点。如果 ray 穿过某个 cell，而该 cell 的地图高度明显高于 ray 当前高度，就说明地图中这个高障碍可能已经不存在，于是清除该 cell。

可以理解为：

如果 LiDAR 光线穿过去了，那么这里不应该还存着一个挡住光线的旧障碍。

但为了避免误删，系统还检查 cell 的最近更新时间和 surface normal。只有满足一定条件时才删除。这样会产生清理延迟和局部 jitter。

仿真中如果不跑真实 ray casting，也应模拟动态障碍残影、局部 cell 被清空、清空后变成 unknown、随后又被 inpainting 填补的过程。

### 7\. 未观测区域如何处理

高程图中会有很多未观测 cell，例如障碍物背后、楼梯边缘后方、机器人身体遮挡区域、传感器视野外区域。论文使用两种方式处理这类区域。

第一种是 upper bound。ray casting 穿过某个未知 cell 时，说明该 cell 的真实地面不应高于 ray 的高度，否则传感器应该已经观测到它。因此系统给未知区域计算一个“最高可能高度”。这对判断负障碍和遮挡区域很有用。

第二种是 inpainting。对于 locomotion controller，空洞会导致优化或高度采样不稳定，因此系统可以用周围边界信息填补空洞。论文中提到的 inpainting 会将 occluded region 用边界处的高度信息补全。

这意味着真实控制器看到的不是“unknown”，而可能是被算法猜出来的高度。这个高度可能合理，也可能错误。仿真中必须模拟这种错误填补，尤其是在台阶后方、坑洞、障碍物背面和视野边缘。

### 8\. raw map 如何变成 virtual floor

图 9 展示了一个典型的后处理链路：

raw map → inpainted map → gaussian smoothed map → virtual floor

raw map 是点云融合后的直接结果，保留了较多传感器噪声、遮挡空洞和竖直尖峰。inpainted map 会填补 raw map 中的空洞，使地图连续。gaussian smoothed map 在填补后的地图上做平滑，减少局部尖峰和高频噪声。virtual floor 是进一步平滑和抽象后的地形层，用作更稳定的 base reference 或优化参考。

这个过程的实际效果是：地图越来越连续、越来越平滑，但也越来越偏离真实几何细节。台阶会被圆滑化，小障碍可能被抹掉，坑洞可能被填平，边缘位置可能移动。因此如果控制器使用的是 virtual floor，它看到的是一个“控制友好”的地形参考，而不是原始几何地形。

仿真中应先明确控制器输入到底对应哪一层。如果实机输入 raw 或 inpainted map，仿真应保留更多噪声和空洞；如果实机输入 virtual floor，仿真应加入更强的平滑、边缘模糊和高度偏置。

### 9\. 仿真中如何复现这种高程图

最理想的方法是在仿真器中直接模拟 LiDAR 或 depth camera，生成点云，然后运行与实机相同的 elevation mapping pipeline。这样点云稀疏性、遮挡、距离噪声、ray casting、地图融合、inpainting、smoothing 和延迟都会自然出现。

完整流程应为：

仿真地形 mesh / heightfield

→ 传感器 ray casting

→ 加入 range noise、dropout、外参误差和 pose noise

→ 生成 noisy point cloud

→ 运行 elevation mapping

→ 输出 raw / inpainted / smoothed / virtual floor map

→ 控制器从对应 map layer 中采样高度。

如果工程成本较低，只想在 height map 层面近似，也应按下面顺序做扰动：

第一，加入距离相关高度噪声。远处 cell 噪声更大，近处 cell 噪声更小。

第二，加入局部 patch dropout。空洞应成片出现，而不是单个 cell 独立随机消失。

第三，加入姿态和外参投影误差。roll / pitch 小误差会导致地图出现整体斜坡，z 偏差会导致整张地图上下平移。

第四，加入时间融合。当前高程图应由上一帧地图和当前观测融合得到，而不是直接等于当前真实地形。

第五，加入 outlier 和残影机制。新障碍不要瞬时出现，旧障碍不要瞬时消失。

第六，加入 inpainting 和 smoothing。空洞区域用邻域高度填补，之后做 median、Gaussian 或 box blur，使边缘模糊化。

第七，加入随机延迟和低频更新。真实地图更新频率低于控制器频率，控制器经常会重复使用旧地图。

### 10\. 推荐的近似噪声模型

对于强化学习训练，可以使用如下近似：

h\_obs\(t\) = Smooth\(Inpaint\(Fuse\(h\_obs\(t\-1\), h\_true\(t\) \+ sensor\_noise \+ pose\_error \+ dropout\)\)\)

其中 sensor\_noise 包括距离相关高度噪声，pose\_error 包括 z drift、roll / pitch 投影误差和外参误差，dropout 表示遮挡和传感器缺失，Fuse 表示 Kalman\-like temporal fusion，Inpaint 表示空洞填补，Smooth 表示 Gaussian / median / box blur 等后处理。

这比直接 h\_obs = h\_true \+ Gaussian noise 更接近真实系统，因为它保留了真实 elevation mapping 中最关键的结构化误差：时间滞后、局部空洞、边缘模糊、整体漂移和错误填补。

### 11\. 总结

这篇论文的高程图不是从点云直接投影得到的静态地形，而是一个经过多帧融合、异常点拒绝、漂移补偿、ray casting 清理、空洞填补和平滑处理后的动态地图。真实部署中，控制器看到的地形不是完美地形，也不只是带白噪声的地形，而是 perception pipeline 的输出分布。

因此，在仿真中最重要的是复现“点云如何被处理成高程图”的过程，而不是简单复现“高程图是什么”。如果最终实机使用该 pipeline，最好的做法是在仿真中也运行同一套 elevation mapping；如果不能运行，也应在 height map 层面模拟距离噪声、遮挡空洞、pose drift、时间融合、残影、inpainting、smoothing 和 map latency。

## FAST\-LIO: A Fast, Robust LiDAR\-inertial Odometry Package by Tightly\-Coupled Iterated Kalman Filter

### 背景

FAST\-LIO 解决的是 LiDAR\-inertial odometry 问题，即如何融合 LiDAR 点云和 IMU 数据，实时估计机器人或无人机的 6\-DoF 位姿。它本身不是高程图构建方法，而是给后续 mapping、navigation、elevation mapping 提供高频、低漂移的 pose estimate。

对于腿式机器人或移动机器人来说，FAST\-LIO 可以放在 elevation mapping 前面：FAST\-LIO 输出机器人位姿，elevation mapping 使用这个位姿把 LiDAR 点云投影到 map frame，再融合成局部高程图。因此，FAST\-LIO 的误差会直接影响后续高程图质量，例如 z drift 会造成假台阶，roll / pitch 误差会把平地投影成斜坡，pose latency 会导致点云和地形错位。

### 整体流程：LiDAR 和 IMU 如何被融合

FAST\-LIO 的输入是两类数据：高频 IMU 测量和 LiDAR 点云。IMU 通常频率较高，例如 100–250 Hz；LiDAR 点云则按 scan 累积处理，系统可以在 10–50 Hz 输出 odometry。

完整流程如下：

IMU forward propagation → LiDAR scan accumulation → feature extraction → backward propagation / motion compensation → residual computation → iterated Kalman filter state update → map update。

具体来说，系统先用 IMU 在两个 LiDAR scan 之间做前向积分，得到当前 scan 结束时刻的状态预测。然后对当前 scan 内的 LiDAR 点做运动补偿，把不同时间采样到的点统一投影到 scan\-end time。接着，将补偿后的点和历史地图中的 plane / edge feature 做匹配，计算点到面或点到线残差。最后，用 iterated EKF 将这些 LiDAR residual 和 IMU 预测融合，更新当前状态。

### IMU 如何做前向传播

FAST\-LIO 将 IMU frame 作为 body frame。系统状态包括姿态、位置、速度、陀螺仪 bias、加速度计 bias 和重力向量，总状态维度为 18。IMU 的角速度和加速度用于连续积分，预测机器人从上一个 LiDAR scan 到当前 scan 结束时刻的状态。

IMU 前向传播的作用是给 LiDAR scan registration 提供一个较好的初值。尤其在快速运动或 LiDAR 特征退化时，单独依赖点云配准会不稳定，而 IMU 可以提供短时间内较可靠的运动预测。

前向传播同时会传播 covariance。运动越剧烈、IMU 噪声越大、bias 不确定性越大，预测 covariance 越大。这个 covariance 后面会影响 Kalman update 中 IMU prediction 和 LiDAR residual 的权重分配。

### LiDAR 点为什么需要运动补偿

LiDAR 一个 scan 中的点不是同一时刻采样的，而是在一个时间窗口内逐点采样得到的。例如 solid\-state LiDAR 或机械 LiDAR 都存在 scan 内时间差。如果机器人在 scan 期间运动，直接把这一帧点云当作同一时刻采样，会产生 motion distortion：墙面会弯曲，平面会错位，边缘会变厚。

FAST\-LIO 的处理方法是 backward propagation。系统先用 IMU 前向积分得到 scan\-end time 的状态，然后从 scan\-end time 反向传播到每个 LiDAR feature point 的采样时刻，估计该点采样时刻相对于 scan\-end time 的相对位姿。之后，每个点都被投影到 scan\-end frame。

这个步骤的实际含义是：

每个点先根据自己的 timestamp 做 deskew，再统一放到当前 scan 结束时刻。

如果这个步骤做得不好，点云会带有运动畸变，后续点到面 / 点到线 residual 会变大，最终影响 odometry 和地图质量。

### LiDAR feature 如何提取

FAST\-LIO 不直接使用所有原始点，而是从原始点云中提取两类特征：planar feature 和 edge feature。平面点来自局部 smoothness 较高的区域，边缘点来自局部 smoothness 较低的区域。

这些 feature points 会参与后续 residual 计算。使用 feature 的目的有两个：一是减少计算量，二是保留对位姿约束最有用的几何结构。论文实验中，在一个 scan 内可以使用上千个有效 feature points，同时保持实时运行。

不过，这也意味着 FAST\-LIO 的效果依赖环境几何结构。如果环境缺少有效平面和边缘，或者 LiDAR FoV 太小，系统会出现退化方向，此时 IMU 的约束会变得更重要。

### residual 如何计算

运动补偿后，当前 scan 的 feature points 被认为都位于 scan\-end time。系统将这些点根据当前状态估计变换到 global frame，然后在历史 feature map 中查找邻近点。

对于 planar feature，系统在地图中找邻近点拟合局部平面，并计算当前点到这个平面的距离。对于 edge feature，系统在地图中找邻近点拟合边缘方向，并计算当前点到这条线的距离。

因此 LiDAR residual 的本质是：

当前点经过位姿变换后，应该落在历史地图中的对应平面或边缘上。

如果 residual 大，说明当前状态估计和地图不一致。iterated EKF 会通过调整状态，使这些 residual 尽可能变小。

### 为什么是 tightly\-coupled

FAST\-LIO 是 tightly\-coupled LIO。它不是先独立完成 LiDAR scan\-to\-map registration，再把 registration 的 pose result 和 IMU 融合；而是直接把 LiDAR feature residual 作为 measurement 放进 Kalman filter，与 IMU prediction 在同一个状态估计框架里融合。

这种做法的好处是：LiDAR 点云对 pose、velocity、bias、gravity 等状态的约束可以在同一个滤波器中共同优化。当 LiDAR 在某些方向退化时，滤波器仍然可以根据 IMU prediction 和 covariance 做合理融合，而不是盲目相信一次 scan registration 的结果。

对机器人实际部署来说，tightly\-coupled 的优势主要体现在快速运动、震动、点云稀疏、FoV 小和局部几何退化场景中。

### iterated EKF 如何更新状态

FAST\-LIO 使用 iterated extended Kalman filter。每次 LiDAR scan 到来后，系统不是只做一次线性化更新，而是重复执行：

用当前状态估计投影点云 → 查找地图对应平面 / 边缘 → 计算 residual 和 Jacobian → Kalman update → 得到新状态 → 再重新计算 residual。

迭代直到状态变化足够小为止。这样可以降低非线性线性化误差，特别是在初始预测和真实 pose 有一定偏差时，比普通 EKF 更稳。

最终输出的是当前 scan\-end time 的最优状态估计，包括姿态、位置、速度、bias 和 gravity。这个状态随后用于把当前 feature points 加入地图，也用于下一帧 IMU 前向传播的初值。

### 为什么 FAST\-LIO 计算快

传统 Kalman gain 计算需要求逆一个与 measurement dimension 相关的矩阵。LiDAR feature points 很多时，measurement dimension 很大，计算量会迅速上升。FAST\-LIO 的关键加速点是改写 Kalman gain 的计算形式，使矩阵求逆主要依赖 state dimension，而不是 measurement dimension。

由于系统状态维度只有 18，而一个 scan 中的有效 feature points 可能超过 1000 个，这个改写大幅降低了计算量。论文实验中，当 feature 数量从 307 增加到 1802 时，旧公式计算 Kalman gain 从 7\.1 ms 增加到 1621 ms，而新公式只从 0\.07 ms 增加到 1\.16 ms。这个差异是 FAST\-LIO 能在 onboard computer 上实时运行的关键。

### map 如何更新

状态更新完成后，当前 scan 中经过运动补偿的 feature points 会使用最新 pose 变换到 global frame，然后加入历史 feature map。下一帧 LiDAR 到来时，系统会从这个地图中取局部 sub\-map，用于查找当前点的 plane / edge correspondence。

这里的 map 不是 elevation map，而是 feature point map。它主要服务于 odometry，即提供 scan\-to\-map residual。后续如果要做高程图，需要另一个 elevation mapping 模块使用 FAST\-LIO 输出的 pose 和原始 / 去畸变点云进行地形融合。

### 初始化如何做

FAST\-LIO 初始化时要求系统静止几秒。论文实验中使用约 2 秒静止数据来估计 IMU bias 和 gravity vector。如果使用 Livox 这类 non\-repetitive scanning LiDAR，静止时还可以得到较高分辨率的初始局部地图，有利于后续 scan\-to\-map registration。

初始化质量会影响后续状态估计。gravity 方向不准会影响 roll / pitch，IMU bias 不准会导致积分漂移，进而影响 LiDAR motion compensation 和 odometry 稳定性。

### 实验结果

论文在 UAV 和手持场景中验证 FAST\-LIO。UAV 实验中，系统运行在 DJI Manifold 2\-C onboard computer 上，使用 Livox Avia LiDAR。室内飞行实验可以达到最高 50 Hz odometry output，平均有效 feature points 约 270，平均运行时间约 6\.7 ms，32 m 轨迹上的漂移约 0\.08 m，小于 0\.3%。

在大角速度室内手持实验中，FAST\-LIO 相比 LOAM 和 LOAM\+IMU 更稳定。10 Hz scan rate 下，LOAM 使用 1107 个 feature points 需要 59 ms，LOAM\+IMU 需要 44 ms，而 FAST\-LIO 使用 1430 个 feature points 只需要 23 ms。

在室外 140 m 手持实验中，FAST\-LIO 的漂移约 0\.07 m，小于 0\.05%。与 LINS 对比时，FAST\-LIO 平均处理时间约 7\.3 ms，而 LINS 约 34\.5 ms，同时 FAST\-LIO 使用更多 feature points，因此地图精度更好。

### 对高程图和仿真的意义

如果前面的 elevation mapping 使用 FAST\-LIO 提供位姿，那么高程图质量会强依赖 FAST\-LIO 的输出。FAST\-LIO 的误差会通过点云投影进入 elevation map。

具体影响包括：

z 方向 odometry 漂移会让整张高程图上下偏移，形成假台阶或地图断层；

roll / pitch 误差会把平地投影成斜坡；

yaw 或 xy drift 会导致同一地形在地图中横向错位；

motion compensation 不充分会让点云边缘变厚，台阶和墙面在高程图中出现模糊；

pose latency 会导致机器人运动时点云落入错误 cell；

IMU bias 或 vibration 会导致短时间内姿态估计抖动，使高程图出现高频噪声。

因此，在仿真中如果要模拟完整感知链路，不能只模拟 LiDAR range noise，还要模拟 FAST\-LIO 这类 LIO 前端带来的 pose noise、deskew error 和时间延迟。

### 仿真中应该如何建模 FAST\-LIO 误差

更真实的仿真流程应是：

模拟 LiDAR scan 和 IMU → 加入 IMU bias、white noise、振动噪声、LiDAR range noise 和 timestamp offset → 运行 FAST\-LIO 或等价 LIO → 输出 noisy pose → 用 noisy pose 将点云送入 elevation mapping → 得到最终高程图。

如果不运行真实 FAST\-LIO，也可以在 pose 层面近似建模：

第一，加入 IMU\-like 高频姿态噪声。roll / pitch 噪声会直接影响地形高度投影，尤其远距离 cell 更明显。

第二，加入慢变 drift。xy drift 影响地图对齐，z drift 影响高程图高度基准，yaw drift 影响局部地图旋转。

第三，加入 scan\-to\-scan jitter。每个 LiDAR scan 的 pose 可以有小幅随机跳动，模拟 LIO update 过程中的估计抖动。

第四，加入 latency。pose 不是当前真实 pose，而是延迟若干毫秒的估计 pose。

第五，加入 deskew residual error。scan 内运动补偿不完美时，点云边缘会被拉宽，台阶和墙面会变厚。可以在点云层面对每个点使用略有误差的 timestamp pose 投影，或者在 height map 中模拟边缘模糊。

第六，加入退化场景误差。在长走廊、平面少、几何重复、视野小或点云稀疏场景中，某些方向的 pose covariance 应增大，例如沿走廊方向漂移更大，yaw 更不稳定。

### 总结评价

FAST\-LIO 的核心价值在于把 LiDAR feature residual 和 IMU prediction 放入同一个 iterated EKF 中紧耦合优化，并通过新的 Kalman gain 计算形式解决了大量 LiDAR feature points 带来的计算瓶颈。它的工程意义很强：可以在 onboard computer 上实时输出高频、低漂移的 odometry，并适应快速运动、震动和一定程度的 LiDAR 退化。

对于腿式机器人 perceptive locomotion，FAST\-LIO 通常不直接提供地形高度，而是作为前端 pose estimator。它的输出决定了后续点云投影和高程图融合的质量。因此，在 sim\-to\-real 中，必须把 LIO pose error 纳入感知噪声建模。真实高程图误差并不只来自 LiDAR 测距，而是来自 LiDAR 测量、IMU 积分、motion compensation、state estimation、map registration 和 elevation mapping 的级联误差。

## ⭐MEM: Multi\-Modal Elevation Mapping for Robotics and Learning

### 核心问题

MEM 解决的是传统 elevation map 只包含几何高度信息的问题。对于导航和运动控制，单纯高度图可以表达台阶、斜坡和障碍物，但不能表达地面材质、语义类别、颜色、视觉特征、可通行性等信息。例如，高草中的人可能在几何高度上不明显，但语义分割可以检测到；水泥路、草地、泥地在高度上可能相似，但对机器人通行代价完全不同。

这篇论文的核心是：在原有 GPU elevation mapping 框架上，增加 multi\-modal layers，使同一个 2\.5D 地图不仅存储 elevation，还能存储 RGB、semantic class probabilities、visual features、traversability 或其他任务相关信息。

因此，它不是重新设计高程图，而是回答一个更具体的问题：不同来源、不同格式、不同语义含义的数据，如何统一关联到 elevation map 的 cell 上，并用合适的融合算法长期更新。

### 整体流程：多模态数据如何进入地图

MEM 的整体 pipeline 可以分成三步：

Data Association → Fusion FAlgorithm → Post\-processing。

第一步是 data association，即把传感器数据和地图 cell 对齐。输入可以是 multi\-modal point cloud，也可以是 multi\-modal image。point cloud 中每个点除了 x、y、z，还可以带 RGB、语义概率、特征向量等字段；image 中每个 pixel 可以带 RGB、semantic probability、feature embedding 等通道。

第二步是 fusion algorithm。完成数据和 cell 的对应关系后，系统根据数据类型选择不同的融合方式，把新观测更新到地图 layer 中。例如，RGB 可以用 latest 或 exponential averaging，语义类别概率可以用 Dirichlet Bayesian inference，连续特征向量可以用 Gaussian Bayesian inference。

第三步是 post\-processing。地图中已有的 elevation、RGB、semantics、features 等 layer 可以被插件进一步处理，生成 traversability、line detection、PCA visualization 或其他 task\-specific layer。图 2 展示了这个结构：图像和点云输入先经过 association，再经过 fusion 进入 multi\-layer elevation map，最后由 plugin 给下游任务使用。

### 点云数据如何融合进 MEM

对于 multi\-modal point cloud，关联方式最直接。每个点根据自己的 x\-y 坐标落入某个 elevation map cell。这个点的 z 可以用于更新 elevation layer，点上附带的额外字段则用于更新对应的 multi\-modal layers。

例如，一个语义点云中的每个点可以包含：

x, y, z, p\_ground, p\_vegetation, p\_human

那么系统会先根据 x\-y 找到对应 cell，然后把该点的语义概率融合到这个 cell 的 semantic layer 中。如果多个点落入同一个 cell，系统先对这些点的观测做 cell 内聚合，再执行 layer 更新。

这种方式适合 RGB\-D camera、stereo camera、LiDAR semantic point cloud、depth\-aligned semantic segmentation 等输入。几何和非几何信息可以在同一次 point cloud update 中进入地图。

### 图像数据如何融合进 MEM

图像输入比点云更复杂，因为 monocular image 本身没有直接的 3D 坐标。MEM 的做法不是把每个 pixel 反投影到空间，而是反过来：把 elevation map 中的 cell 投影到图像平面。

具体流程是：

首先，系统遍历相机视野 FoV 内的 elevation map cells。对于每个 cell，系统已知它在 map 中的 x、y 和当前 elevation height，因此可以得到这个 cell 的 3D 位置。然后利用相机外参和内参，通过 pinhole camera model 将该 3D cell 投影到 image plane，得到对应 pixel 坐标。

但是，仅仅投影还不够，因为一个 cell 可能被前方地形或障碍物遮挡。论文使用 ray\-casting visibility check 判断 cell 是否真的可见。具体来说，从相机 focal point 到目标 cell 连一条 ray，然后检查 ray 经过的中间 cells。如果所有中间 cell 的 elevation 都低于这条 ray 的高度，则目标 cell 被认为可见；否则说明它被遮挡，不应该使用该 pixel 的信息更新该 cell。

这个过程的关键是：图像语义不是无条件刷到地图上，而是只更新从相机视角可见的 cell。图 3 展示了这个过程：地图中的 cell 被投影到 semantic image 上，同时通过 ray casting 判断是否 occluded。

### 为什么采用 cell\-to\-image，而不是 image\-to\-cell

MEM 选择把 cell 投影到 image，而不是把每个 pixel 投影到 map。原因是机器人局部 elevation map 中潜在可见的 cells 数量通常少于 image pixels 数量。比如一个 250×250 的局部地图只有 62,500 个 cell，而图像可能有几十万到上百万个 pixel。遍历 cell 并行投影通常更高效，也更适合 GPU 实现。

此外，cell\-to\-image 还可以直接利用已有 elevation layer 做遮挡判断。也就是说，当前地图中的几何高度被用来决定图像信息能不能写入某个 cell。这使得 MEM 可以把 monocular RGB / semantic / feature image 融合进 2\.5D 地图，而不要求图像本身提供深度。

### 不同模态如何选择融合算法

MEM 的一个重点是：不同类型的数据不能用同一种融合方式。连续值、颜色、特征向量和语义概率的统计性质不同，因此需要不同的 fusion algorithm。

论文实现了四类融合方式：Latest、Exponential Averaging、Gaussian Bayesian Inference 和 Dirichlet Bayesian Inference。

Latest 最简单，只保留当前最新观测。如果同一 cell 中有多个点，则先对这些点做平均。这个方式适合快速更新，但不利用历史信息，所以对噪声更敏感。

Exponential Averaging 使用指数滑动平均：

θt = w at \+ \(1 \- w\) θt\-1

其中 at 是当前观测，θt\-1 是上一时刻 cell 中存储的值，w 是用户设定权重。w 越大，地图越相信新观测；w 越小，地图越平滑但更新更慢。这个方法可用于 RGB、连续值或概率类信息，但它不是严格概率推断，因此输出不一定能解释为真实 posterior probability。

Gaussian Bayesian Inference 用于连续特征向量，例如视觉 embedding、PCA feature、learned feature map 等。它假设某个 cell 的观测来自一个高斯分布，并用新观测更新该 cell 的均值和方差。相比简单平均，它保留了不确定性信息，更适合连续特征长期融合。

Dirichlet Bayesian Inference 用于语义类别概率。每个 cell 对 K 个类别维护一个 Dirichlet 分布参数 α。每次有新的 class probability 或 one\-hot semantic measurement 落入该 cell，就把观测累加到 α 上，再归一化得到该 cell 的类别概率。这样可以把多帧语义观测累积成更稳定的 semantic map。

### 语义概率如何被长期融合

对于语义分割，网络每帧输出的是 pixel\-wise class probabilities，例如 ground、vegetation、human 等类别。MEM 会先通过 point cloud association 或 image projection association 把这些概率对应到 map cell，然后使用 Dirichlet Bayesian inference 更新 cell 的类别分布。

这个过程可以理解为：每个 cell 都维护一个关于类别的投票统计量。某个 cell 多次被识别为 vegetation，它的 vegetation 概率就会逐渐升高；如果后续观测变成 ground，概率不会瞬间改变，而是逐步调整。图 4 对比了 exponential averaging 和 Bayesian inference 的行为：exponential averaging 对新观测响应更直接，而 Bayesian inference 的变化更平滑、更保守。

这对机器人很重要，因为单帧语义分割通常有噪声。通过地图级融合，系统可以把多帧不稳定的语义预测变成更稳定的空间语义层。

### 视觉特征如何被融合

MEM 不只支持固定语义类别，也支持高维视觉特征。例如，可以先用 ViT 从 RGB 图像中提取每个 pixel 的 feature embedding，然后把这些 feature channels 作为 multi\-modal image 输入地图。每个 feature 维度被视为连续值，用 Gaussian Bayesian Inference 融合进对应 cell。

融合后的 feature layers 可以进一步用于下游学习任务。论文中的农业场景使用 ViT features 和 elevation layers 作为输入，经过一个 ERFNet 预测果树行线。也就是说，MEM 不只是显示语义图，而是把多帧、多视角视觉特征聚合到 robot\-centric map 中，再交给神经网络做任务推理。

这种设计的好处是：下游网络不必直接处理原始图像序列，而可以处理一个已经空间对齐、时间融合、包含几何和视觉信息的局部地图。

### RGB 信息如何进入地图

RGB 可以来自 colorized point cloud，也可以来自 monocular RGB image。对于 point cloud，RGB 跟随每个 3D 点直接落入对应 cell；对于 image，RGB 通过 cell\-to\-image projection 和 visibility check 写入对应 cell。

融合方式可以选择 latest 或 exponential averaging。latest 会让颜色快速反映当前图像，但对光照变化、运动模糊和错误投影敏感；exponential averaging 会让颜色层更稳定，但也会产生滞后。

图 7 展示了 outdoor colorization：系统同时使用多个 point clouds 和多个 images，将环境颜色融合进 elevation map。这个 color layer 可以用于人类可视化，也可以作为 learning task 的输入。

### Post\-processing 如何工作

MEM 的 post\-processing 是 plugin\-based。用户可以选择已有 map layers 作为输入，生成新的 layer，修改已有 layer，或者给外部模块输出结果。由于所有 layer 都已经在 GPU 上，plugin 可以直接读取 elevation、RGB、semantic、feature 等数据，减少 CPU\-GPU 数据传输。

图 5 展示了两个例子。第一个是 line detection plugin：输入 elevation 和 feature layers，输出农业场景中的左右树行线。第二个是 PCA plugin：输入高维 ViT feature layers，输出低维 PCA layer 用于可视化。

这说明 MEM 的地图不是终点，而是一个统一的空间记忆结构。不同任务可以在这个结构上继续计算 task\-specific representation。

### 实验中如何证明实时性

论文在 10 m × 10 m、4 cm resolution 的地图上测试，地图大小为 250 × 250 cells。输入点云规模为 230,400 points。RTX 4090 上完整 update 约 2\.6 ms，Jetson Orin 上约 23\.6 ms，对应 Jetson Orin 上约 42\.3 Hz 的更新能力。主要耗时来自 height update \& ray casting，在 Jetson Orin 上约 17\.5 ms；multi\-modal update 本身约 1\.5 ms。

这说明加入 multi\-modal layers 后，整体耗时仍然和原 GPU elevation mapping 在同一数量级。论文还测试了 layer 数量增加时的性能，multi\-modal update 时间随 layer 数量近似线性增长。Jetson Orin 上 20 个 layers 时，exponential averaging 和 Bayesian inference 的总处理时间约 29 ms。

### 和普通 elevation mapping 的区别

普通 elevation mapping 的输出主要是 elevation、variance、normal、traversability 等几何相关层。MEM 的区别在于，它把地图 cell 扩展成一个多模态信息容器。每个 cell 可以同时包含：

height、variance、RGB、semantic probabilities、visual features、traversability、PCA layer、task\-specific prediction。

因此，MEM 更像是一个 robot\-centric spatial memory。它把不同传感器、不同网络、不同时间的输出对齐到同一个局部坐标系中，并在 cell 级别进行融合。

### 对仿真的意义

如果真实系统使用 MEM，那么仿真中不仅要模拟几何高程图噪声，还要模拟多模态信息的噪声和对齐误差。语义层、颜色层、视觉特征层并不是完美标签，而是经过相机投影、遮挡判断、神经网络预测和时间融合后的结果。

因此，仿真中至少需要考虑以下问题。

第一，semantic prediction noise。语义分割网络会误分类，尤其在高草、阴影、反光、远距离、小目标和运动模糊场景中。仿真语义层不应该直接使用 ground\-truth label，而应加入 confusion matrix，例如 vegetation 和 ground 混淆，human 和 vegetation 混淆。

第二，projection error。图像信息写入地图 cell 依赖相机外参、内参、位姿估计和 elevation height。如果 pose 或 calibration 有误差，语义和颜色会被刷到错误 cell 上。仿真中应加入 pixel\-to\-cell misalignment，尤其在物体边界附近。

第三，occlusion error。MEM 用当前 elevation map 做 ray\-casting visibility check。如果 elevation map 本身有错误，visibility 判断也会错误。仿真中应允许被遮挡区域被错误更新，或者可见区域没有被更新。

第四，temporal fusion delay。Dirichlet Bayesian inference 和 exponential averaging 都会让语义层具有时间记忆。真实语义 map 不会瞬间改变类别；错误语义也可能残留一段时间。因此仿真中应模拟 semantic persistence 和 delayed correction。

第五，multi\-layer inconsistency。几何层和语义层可能不同步。例如 elevation 已更新，但 semantic 还来自上一帧；或者 RGB 投影正确，但 height 因为遮挡产生空洞。控制器或 planner 使用 MEM 时，应适应这种 layer 间不同步。

第六，feature noise。对于 ViT / CNN feature layers，仿真中很难直接构造真实 feature 分布。如果需要学习型下游任务，最好在仿真中也跑相同视觉 encoder；否则至少要模拟 feature dropout、feature smoothing 和 domain shift。

### 推荐的仿真建模方式

最真实的方式是完整复现感知链：

仿真 RGB / depth / LiDAR → 运行语义分割或视觉 encoder → 生成 multi\-modal image / point cloud → 用 noisy pose 和 calibration 投影到 MEM → 用相同 fusion algorithm 更新 map → 下游控制器或 planner 读取 MEM layers。

如果工程成本较低，也可以在 map layer 层近似：

对于 elevation layer，加入距离相关噪声、空洞、z drift、姿态误差、inpainting 和 smoothing。

对于 semantic layer，使用 ground\-truth label 经过 confusion matrix 采样，并加入空间边界模糊、随机 patch 错误和时间融合。

对于 RGB layer，加入光照变化、颜色噪声、投影错位和局部缺失。

对于 feature layer，加入高斯扰动、dropout、低通滤波和 domain randomization。

对于不同 layer 之间，加入随机时间延迟和空间错位，使它们不完全同步。

一个更接近真实 MEM 的近似模型可以写成：

真实几何 / 语义 / RGB → sensor noise → network prediction noise → projection association error → visibility mask error → fusion over time → post\-processing → MEM output。

重点不是让每个 layer 都完美，而是让下游策略看到与实机一致的多模态地图误差。

### 总结评价

MEM 的主要价值在于把 2\.5D elevation map 从纯几何地图扩展成多模态地图。它通过统一的 data association 和 layer\-specific fusion algorithm，把 point cloud、monocular image、RGB、semantic probability 和 visual feature 都融合到同一个 robot\-centric map 中。由于实现基于 GPU，它在 Jetson Orin 上仍然可以保持实时更新。

它的局限也来自这个设计。图像信息的正确性依赖相机标定、位姿估计和当前 elevation layer；语义层依赖前端网络质量；多帧融合会增强稳定性，但也会保留错误和产生滞后。此外，2\.5D 地图仍然无法完整表达多层结构和复杂遮挡。

对于机器人学习和 sim\-to\-real，MEM 提醒我们：真实感知输入不是单一高度图，而是多层、有延迟、有噪声、有错位的空间记忆。如果实机使用 MEM，仿真就应该同时建模 elevation、semantics、RGB、features 的噪声、投影误差和时间融合行为。







# List 3：Navigation

## Learning Robust Autonomous Navigation and Locomotion for Wheeled\-Legged Robots

## Learning to Evolve: Multi\-modal Interactive Fields for Robust Humanoid Navigation in Dynamic Environments

## NeuPAN: Direct Point Robot Navigation with End\-to\-End Model\-based Learning

## STATE\-NAV: Stability\-Aware Traversability Estimation for Bipedal Navigation on Rough Terrain

## SysNav: Multi\-Level Systematic Cooperation Enables Real\-World, Cross\-Embodiment Object Navigation
