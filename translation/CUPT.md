---
title: CUPT
---

# CUPT

## 摘要 

本文介绍了一种用于航位推算导航的新算法 - 恒速更新（CUPT），它是流行的零速度更新（ZUPT）的扩展。通过将廉价的IMU（Inertial Measurement Unit，惯性测量单元）安装在用户的鞋子上，所提出的算法不仅能够检测行走时的站立相位，还能捕获恒定速度情况（如扶梯）从而高效地降低IMU的误差。本文详细介绍了CUPT原型的概念，设计和测试。测试结果表明，它可以有效检测到恒定速度，其水平定位误差低于行驶总距离的0.45％，垂直误差低于0.25％。这种表现在现有文献中达到了最高的准确度。

##I导言

在包括GPS（全球定位系统）信号恶化环境等所有情况下无缝导航，是一个有挑战性的问题。随着机器人的发展，稳健的跟踪和导航的需求很迫切。机器人通常部署在GPS信号不可靠的室内环境中。不依赖任何周围的信息，航位推算法可以提供连续的导航信息，但有累积的误差，需要通过其他测量来纠正。

GPS和IMU的整合在导航应用中备受关注。日益发展的技术使得在室内环境中使用GPS成为可能。Godha和Lchapelle提出了一个系统，将鞋装IMU和GPS结合起来，以限制户外场景中的漂移错误。该系统具有使用IMU桥接室内导航解决方案的能力。

有一些非GPS方法可以跟踪和导航，这些通常需要外部参考的个人位置。Hisashi等人提出了一种图像序列匹配技术来识别位置和以前访问过的地方。同时进行的定位和测绘工作也在进行中，SLAM通常使用相机或激光雷达（光探测和测距）作为传感器。但是，与惯性传感器不同，上述这些传感器对环境非常敏感，并且在不利的运行条件下不可靠。

有许多不同的使用惯性传感器的PDR系统。最简单的就是计步器，它计算步数并估计平均步长。Cho和Park 提出了一种类似计步器的方法，该方法使用连接到用户靴子的双轴加速计和双轴磁力计。根据通过神经网络的加速度计读数来估计步长，并且使用卡尔曼滤波器来减少磁干扰的影响。虽然在户外的结果是合理的，但室内环境下的结果具有较大的误差，因为变化的磁干扰。

作为行人，行走时有站立阶段，可用于PDR漂移修正。因此，开发了用于PDR导航的各种ZUPT算法[4-17]。已经提出了使用加速度计进行步长检测的算法，主要包括三种类型：峰值检测，过零检测和平坦区域检测[4]。在文献[5]中，将步态模型化为马尔可夫过程，并且使用基于力传感器的隐马尔可夫模型过滤器来估计步态以确定何时应用ZUPT。类似地，在[6]中使用陀螺仪输出的分割来构建马尔科夫模型。基于加速度计和陀螺仪输出可以实现几乎不同的算法。在[7]中，零速度通过比较z轴加速度计和y轴陀螺仪输出与阈值来确定。在[8]中，零速度是根据加速度计和陀螺仪的规范以及加速度计的方差来确定的。

所有上述检测器可以概括为所谓的加速度移动方差检测器，加速度量检测器，角速率能量（ARE）检测器，它们都是广义似然比检验。Skog等人[10]开发了一种新颖的姿态假设最优检测器，但是，它仅限于2D情况。Ojeda和Borenstein等人[11]提出了一个基于鞋子的导航系统由15个状态的错误模型组成。他们的系统在2D和3D环境中都能很好地工作。使用ARE探测器和相关的信号处理算法，水平相对误差约为总行程的0.49％，但垂直误差总是大于1％。

但是，上述算法都不能在速度一致的情况下表现良好，例如在电梯或自动扶梯上。这是由于他们把恒速检测为零。本文提出了一种改进的鞋装PDR系统，并提出了一种新的算法CUPT。它不仅可以检测站立阶段为ZUPT，而且可以检测用户站在电梯或自动扶梯上时的速度。使用扩展卡尔曼滤波器（EKF）应用具有24个错误状态的模型来校正IMU错误。实验结果表明，该系统的定位精度在水平面上优于0.45％，在垂直方向上优于0.25％。这与ZUPT [11]，[12]和[16]报道的一些最高定位精度数据相比更好或相当。

## II数学模型

IMU通常由三个加速度计和三个正交模式的陀螺仪组成。 标准捷联惯性导航系统（INS）机械化用于我们的系统。

### *A.捷联式惯性导航机械化*

图1显示捷联INS机械化。姿态可以通过陀螺仪的旋转速率的积分来确定。矢量变换后，速度和位置可以通过加速度与重力补偿和科里奥利力的积分来确定。这些块的细节可以在[9]和[18]中看到。

![](https://i.loli.net/2018/04/23/5add884ba356d.png)

> 图1 捷联INS机械化的基本模块

然而，实际上，这种简单的整合将导致由于与测量相关的噪声和传感器的非线性导致位置误差随时间无限增长。为了使用IMU获得高精度的导航解决方案，应用ZUPT技术来减少错误。

### *B. ZUPT*

当一个人走路时，他的脚在静止阶段和摆动阶段之间交替。这种姿态间隔出现在零速度的每一步中，因此速度误差可以几乎周期性地复位。当连接在鞋上的传感器静止时，识别姿态间隔很重要，然后应用ZUPT将错误与EKF绑定。在每个估算周期中，一旦检测到零速度，ZUPT将更新的错误状态传递给INS，否则，错误状态将使用在上一个立场间隔中更新的错误状态。ZUPT不仅可以校正速度，还可以帮助限制相关位置和姿态误差并估计传感器偏差误差。图2显示了典型的INS-EKF-ZUPT PDR方法[8]中的主要模块。

![](https://i.loli.net/2018/04/23/5add8b7c17f20.png)

> 图2. 该框架用于行人推算的主要模块。

### *C. CUPT*

原则上，所有正确检测到的时刻都可以应用于EKF估计误差状态校正，包括但不限于零速度。这里速度不变，因此该算法被称为CUPT。所提出的算法能够检测站立时的站立相位，以及站在移动的电梯或移动的自动扶梯上时的恒定速度。

当真实速度条件恰好为零时，ZUPT仅仅是CUPT中的一个特例。如果采用ZUPT算法来测试等速度的情况，它将把恒定速度视为零速度并执行ZUPT。此时导航方案显然是错误的。

CUPT算法的核心概念是检测站立平台的速度并用它来校正INS漂移。显然，在不考虑速度条件的情况下，等速相位满足零速度相位的所有条件。CUPT不仅需要识别站立阶段，还需要识别站立阶段的速度以及速度是否恒定。

### *D. Kalman Filter*

EKF的系统方程的精确表达式取决于所选的状态以及用于描述它们的错误模型的类型。我们使用的EKF包含以下24个错误状态[19]：

$\delta x_{Nav} = [\delta r_N,\delta r_E,\delta r_D,\delta v_N,\delta v_E,\delta v_D,\delta \varphi_H,\delta \varphi_P,\delta \varphi_R,]^T$

$\delta x_{INS} = [\nabla _b,\nabla_f,\varepsilon _b,\varepsilon _f]$

$\delta x_{Grav} = [\delta g_N,\delta g_E,\delta g_D]^T$

其中$\delta x$是导航错误矢量，IMU传感器测量误差矢量和重力不确定度。$\nabla$是加速度计误差矢量，$\varepsilon$是陀螺错误向量。下标b表示偏差，下标f表示比例因子。

系统采用Psi角模型[19]：

$\delta \dot{r} = -\omega _{en} \times \delta r + \delta v$

$\delta \dot{v} = -(\omega_{ie} + \omega_{in} \times \delta v - \delta \varphi \times f + \delta g + \nabla)$

$\delta \dot{\varphi} = -\omega_{in} \times \delta \varphi + \varepsilon$

其中$\delta r,\delta v, \delta \varphi$分别是位置，速度和姿态误差矢量，$\delta g$是计算的重力矢量中的误差，f是特定的力矢量，$\omega_{ie}$地球转速矢量，$\omega_{en}$是转换速率矢量，$\omega_{in}$是从导航到惯性框架的角速度矢量。

通过上述方程的线性化获得动态矩阵。动态矩阵的详细表达式可以在[19]中找到。测量模型是：$z = H\delta x + n$，其中$H = [0 \ I  \ 0\ 0\ 0\ 0\ 0\ 0 ]$，n是测量噪声。

考虑错误状态$\delta x =  \hat{x} - x$，其中$\hat{x}$是估计的状态，x是真实状态，恒定速度的观察z是$\hat{x} - V$，零速度观察是$\hat{x} - 0$，其中$\hat{x}$由捷联惯性计算机计算。

在这种方法中，获得良好导航性能的关键问题是对于零速度和恒定速度获得可靠和稳健的站立相位检测。

## III恒定相位检测

如果检测器确定脚是静止的，则CUPT应用于EKF。这里“静止”意味着脚不仅与地面接触而且不移动，而且还相对于移动平台保持静止。无论采用何种更新技术，传感器输出的特性都是相同的，因此，可以使用零速度检测器检测两种情况下的站立相位。

在提出的方法中，没有必要确定相位间隔的确切开始和结束。相反，只要检测到一个步幅中的单个实例，就可以用它来消除传感器漂移，使错误不会累积。本文介绍的站姿间隔检测器在实时条件下同时使用加速度计和陀螺仪测量。实现的算法由以下4个条件组成：

- 加速度的大小，对于每个样本k：

  $|a_k| = [a_{xk}^2 + a_{yk}^2 + a_{zk}^2]^{0.5}$,

  $$C1 = \begin{cases}1&  th_{a min}<|a_k|<th_{a max} \\0 & otherwise \end{cases}$$

- Z轴加速度的大小$|a_{kz}|$，对于每个样本k：

  $$C2 = \begin{cases}1&  th_{az\ min}<|a_{kz}|<th_{az\ max} \\0 & otherwise \end{cases}$$

- 陀螺仪的大小$|\omega _k|$，对每个样本k：

  $|\omega _k| = [\omega_{xk}^2 + \omega _{yk}^2 + \omega_{zk}^2]^{0.5}$

  $$C3 = \begin{cases}1&  |\omega_k|<th_{\omega \ max} \\0 & otherwise \end{cases}$$

- 陀螺仪y轴的大小$|\omega_{ky}|$，对每个样本：

  $$C4 = \begin{cases}1&  th_{min}<|\omega_{ky}|<th_{\omega y\ max} \\0 & otherwise \end{cases}$$

z轴加速度和y轴角速度是步行事件的最重要指标。由于与地面接触的脚表示站立间隔，所以重力加速度和足部旋转的角速度在此持续时间内不变化。然而，由于鞋表面的不稳定倾斜，所测量的y和z不完全是0和g。此外，在理想情况下，水平面上的总角速度和加速度的幅度在站立阶段应为零。但事实上，它们不会是零，而是比给定阈值更低的值。阈值基于加速度计和陀螺仪输出在初始静止时间段的平均值加上一定的波动水平以确保其稳健性。当IMU处于稳定状态时，初始静止时段处于传感器数据采集的开始阶段。

必须同时满足这四个逻辑条件才能宣布一只脚静止。另外，应该使用另一个时间阈值来防止错误检测摆动阶段的某些过零点。这里我们设置一个大小为5的固定样本窗口。一旦样本被检测为站立阶段，并且下一个连续的四个样本都被检测为站立阶段，则应用CUPT来减小漂移。速度重置为当前已知状态，在EKF改进当前位置，速度和姿态后，位置，速度和姿态误差重置为零。加速度计和陀螺仪误差和重力不确定性会累积并反馈给EKF，从而允许滤波器在每次步幅之后校正速度误差。

以电梯为例，电梯在经过一段加速和制动之后再次静止。一旦电梯完成加速并在电梯开始减速之前出现恒定速度。在这种情况下，最重要的指标是z轴上的加速度。

对于自动扶梯的情况，水平速度是最重要的指标。当自动扶梯上的一个脚踏步，然后跟上自动扶梯升降的速度直到离开时，恒定（自动扶梯）的速度由扶梯上的脚实例决定。一旦水平速度在站立阶段超过阈值（在此我们设定为0.35m / s），则假定脚在自动扶梯上，并且我们将CUPT用作EKF的测量。尽管在自动扶梯的不同部分，垂直角度可能不同。在这里，我们将自动扶梯的角度设置为30度，这可以通过力分析得到垂直速度。

我们的系统也可以粗略估计步幅长度。根据[9]，当一个人走路时，他们的脚在一个站立阶段和一个移动步幅阶段之间交替，每个阶段持续约0.5秒。由于所有处于站立阶段的样本都已经被检测到，我们可以找到每一步脚落在地面上的位置，从而获得每一步的长度。

## IV实验

### *A.硬件描述*

我们用来自Intersense Incorporated的MEMS IMU Navchip来实现所提出的PDR算法。 Navchip是世界上最小的IMU，它提供了出色的测量结果。 Navchip的小尺寸使其易于连接到靴子，并且对行走没有影响。 表I列出了Navchip的规格（封装在外壳内）。

![](https://i.loli.net/2018/04/23/5addb7cf0c356.png)

我们将IMU安装在靴子的前表面上。在初始测试过程中，我们发现过度冲击和加速度测量溢出的现象。采取了减震垫和保护套等对策。之后的测试显示减少的震动和改进的测量。 图3显示了加速计测量之前（上图）和冲击对策之后。

![](https://i.loli.net/2018/04/23/5addb84274490.png)

> 图3. 减震前后加速度的比较

安装了IMU的行李箱被行人穿着进行测试。这也适用于踏板机器人。

### *B.测试*

在本节中，我们以两种情景呈现结果，所有轨迹都是闭环。另外，在步行之前的所有情况下，对象在起点静止约4秒钟。

- 在电梯上实验

  测试在有多层电梯的27层塔式建筑中进行。他走进电梯前走了一会儿。门关闭时，受检者停止走动并保持不动，直到门再次打开。电梯选择停靠在哪个楼层以及哪个电梯选择采取的都是随机的。由于开始（黑点）和停止点（蓝点）在现实中完全相同，因此下图中黑点和蓝点之间的距离表示相对误差。

  轨迹如下：

  a）从4层开始行走; 乘坐电梯到26层，然后步行到另一个电梯到达4层。
  b）从4层开始行走; 乘坐电梯到达第26层，然后由同一个电梯步行到第2层，然后由另一个电梯返回到第4层。
  c）从4层开始行走，乘坐电梯到24级，爬上楼梯到26级，再乘坐另一部电梯到达16级，走出电梯，然后回到4级。

  ![](https://i.loli.net/2018/04/23/5addb9ba8a5e6.png)

  > 图4.电梯轨迹1

  ![](https://i.loli.net/2018/04/23/5addb9f49e4a6.png)

  > 图5.电梯轨迹2

  ![](https://i.loli.net/2018/04/23/5addba47d7a1a.png)

  > 图6.电梯轨迹3

  左图是轨迹的3D视图，右图是红线画出的随时间变化的高度，黑点表示采样点已经应用了CUPT或ZUPT。

- 在自动扶梯上实验

  从较低的层面开始，主体走了一段时间，站在一只升起的自动扶梯上，IMU安装在一只脚上。然后从中层的自动扶梯下来，然后走到另一个出现的自动扶梯顶层，并通过两台下降的自动扶梯返回到起点。

  ![](https://i.loli.net/2018/04/23/5addbc9032f31.png)

  > 图7.自动扶梯轨迹

  图7是轨迹的3D视图，其清楚地表示四个自动扶梯和三个水平走向。

## V结果与讨论

图8显示了电梯中CUPT计算速度的一部分。电梯经过一段加速时间，保持恒定速度，然后减速到零速度。绿线表示垂直方向的速度，蓝线表示z轴上原始加速器的测量值。

![](https://i.loli.net/2018/04/23/5addbd2a43f8a.png)

> 图8.电梯中运动

图9a显示了应用ZUPT在地面上行走的步骤，但没有更新站立在自动扶梯上，这导致没有IMU漂移校正的速度误差较大。图9b显示了相同情况下CUPT计算速度的一部分，其与自动扶梯一起保持恒定速度，直到下降到地面。黑点表示采样点被检测为恒定速度或零速度，红色，蓝色和绿色线分别绘制北，东和垂直方向的速度。

![](https://i.loli.net/2018/04/23/5addbde23380e.png)

> 图9a.在没有更新的自动扶梯中运动

![](https://i.loli.net/2018/04/23/5addbe35e3870.png)

> 图9b.CUPT下的自动扶梯

表II和III分别总结了电梯和自动扶梯实验的返回位置误差。

![](https://i.loli.net/2018/04/23/5addbfb13d665.png)

![](https://i.loli.net/2018/04/23/5addbfb147317.png)

虽然这些测试结果相当不错，但仍然有很大的空间来进一步改进算法，以获得更好的性能并处理更多的挑战情况。在加速和制动部分处理电梯数据，CUPT遵循INS机械化计算位置，速度和姿态，不进行任何更新和修正，直到检测到恒定速度。但是，在某些情况下很难检测到恒定的速度。加速和制动部分的运动具有可变的加速度。特别是在制动过程中，电梯先减速至低速并保持恒定一段时间，然后减速至零速度，这需要更具体的分析。而且，电梯停靠也会影响高度测量的准确性。CUPT只在加速期结束的情况下实现，如果电梯频繁停车，加速时间较长，检测不明显，可能导致错误或缺少恒速检测。

而对于自动扶梯，自动扶梯的垂直角度是预先确定的，以计算垂直速度。这个角度可以通过传感器进行更复杂的调查。尽管表3中的高度相对误差很小，但水平相对误差并不是很好。这可能是由于自动扶梯两端垂直角度的变化造成楼梯向前移动但不再向上或向下移动，其中预定角度给测量更新带来误差。

## VI结论和未来的工作

本文提出了一种适用于移动平台的自含式PDR系统的CUPT算法，性能良好。对于电梯实验，水平面上的平均返回位置误差约为行驶总距离的0.37％，高度漂移约为0.18％。自动扶梯的返回位置误差稍大，平均为0.44％，垂直为0.23％。从测试结果中，我们可以得出结论：CUPT算法对于限制IMU错误的增长非常有效。我们强大的站立相位检测和恒速更新算法表现出令人满意的性能。这种CUPT算法可以在踏板机器人或行人正在行走时同时提供精确的导航解决方案。

该算法在步行条件下鲁棒性强，然而，对于跑步或跳跃等其他一些腿部运动来说表现不是很好，主要是由于过度的冲击和传感器测量溢出。我们仍在努力解决这些问题。今后，我们将通过集成磁场测量来研究消除航向误差的方法，更强大的CUPT算法和适用于更复杂环境的特定模型分析。