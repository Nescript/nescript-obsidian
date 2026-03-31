# bipedal_wheel_controller学习文件

围绕控制器中的函数、对象、变量等的作用进行汇总

## 主控制器模块(`BipedalController` 类)

### 1.核心对象与变量

#### 1.硬件接口对象 (Hardware Handles)

`imu_handle_`：IMU 传感器句柄，用于获取机身的真实姿态（Roll, Pitch, Yaw）和加速度

`{left/right}_{hip/knee}_joint_handle_`：左右腿的髋关节和膝关节句柄，用于位置/速度反馈及力矩输出

`{left/right}_wheel_joint_handle_`：左右轮毂电机的句柄，控制轮子转动

`joint_handles_`：存储所有被控关节句柄的数组，方便集中下发指令

`robot_state_handle_`：用于查询 tf（坐标变换）树状态的句柄

#### 2.数学与算法对象

`vmc_`：实例化 `VMC` 类的对象，用于处理五连杆的运动学正解和逆解

`kalmanFilterPtr_`：实例化卡尔曼滤波器对象，专门用于“车轮打滑检测 (Slippage detection)”，融合 IMU 加速度和轮速来估算真实的底盘线速度

`mode_manager_`:状态机管理器对象，负责管理和切换底盘的不同形态(平衡、坐下、恢复等)

`{left/right}_leg_angle_vel_lpFilterPtr_`:腿部角速度的低通滤波器，用于平滑 VMC 算出的虚拟腿角速度，减少扰动

#### 3.LQR 权重与拟合系数与状态向量 (State Vectors)

`x_left_`, `x_right_`：维度为 `[6 x 1]` 的矩阵，是 LQR 算法的输入 **\$\\mathbf{x} = [\\theta, \\dot{\\theta}, x, \\dot{x}, \\phi, \\dot{\\phi}]^T\$**（分别对应：虚拟腿倾角、腿角速度、轮端位移、轮端线速度、机身俯仰角、机身俯仰角速度）。

`q_`, `r_`：LQR 算法的对角权重矩阵 **\$Q\$** 和 **\$R\$**。

`coeffs_`：维度为 `[4 x 12]` 的矩阵，存放利用最小二乘法（三多项式拟合）算出的 K 矩阵（反馈增益矩阵）多项式系数，用于不同腿长下实时算出 LQR 的 K 矩阵。

#### 4. 数据融合与打滑检测变量 (Filter & Slippage Detection)

`A_`, `B_`, `H_`, `Q_`, `R_`, `X_`, `U_`：卡尔曼滤波的状态转移矩阵、控制矩阵、观测矩阵、过程噪声矩阵、观测噪声矩阵以及状态向量和控制向量

`wheel_vel_aver`：左右轮理论计算出的全局线速度的平均值

`slip_flag_`：打滑标志位，当理论轮速与卡尔曼估计的实际机身速度相差过大时置为 `true`

`R_wheel_`, `slip_alpha_`, `slip_R_wheel_`：正常情况下的观测协方差、打滑惩罚系数以及打滑情况下的协方差。发生打滑时，会动态调大 **R** 矩阵的值，使卡尔曼滤波不再信任打滑轮子的速度反馈

#### 5. 控制参数与动态配置 (Parameters & Configs)

`model_params_`：物理模型参数（机体质量 M、腿长、各部件转动惯量等）

`bias_params_`：传感器读数或安装等产生的静态偏差补偿值

`leg_threshold_params_`：各类状态转移判定的物理界限参数

`d_srv_`：动态调参（`dynamic_reconfigure`）服务，允许程序运行时实时更改 LQR 参数

`config_`, `config_rt_buffer_`：实时安全的参数缓存冲区，保证参数修改（ROS 非实时线程）不阻塞控制器高频主控循环（实时线程）

#### 6. 标志位与模式状态 (Flags & Logic States)

`balance_mode_`：记录当前底盘期望处于的状态机模式名（如 `STAND_UP`, `SIT_DOWN`）

`balance_state_changed_`：模式切换瞬间的标志位，触发状态机执行入口和出口切换逻辑

`overturn_`：翻车标志位。通过 IMU 解算的 Roll/Pitch 判断机体是否倾角过大且倒向地面，一旦置位强制进入恢复（`RECOVER`）或瘫痪保护模式

`legCmd_`, `jumpCmd_`：从话题接受的高级控制指令缓存（例如要求的跳跃动作或拉伸腿长）

#### 7.ROS 通信发布者/订阅者 (ROS Pub/Sub)

`leg_cmd_sub_`：订阅上位机或遥控器发出的腿长控制指令

`unstick_pub_`：发布车轮防卡死/越障标志功能

`upstair_status_pub_`：发布机器人是否处于上下楼梯（特定姿态控制）的状态

`legged_chassis_status_pub_，lqr_status_pub_ `等系列：这些使用 `realtime_tools::RealtimePublisher` 封装以确保硬同步中发布的数据不产生死锁，用于输出当前的各类物理估算值（机体 XYZ 轴姿态、LQR 输出偏差值与推力）供 rqt 绘图或调试分析

### 2.调用函数

#### 1. `init(...)`

**作用**：插件加载时的初始化入口。负责获取所有硬件句柄、实例化各核心类对象、建立ROS话题的跨线程通信及卡尔曼滤波器等组件的初始化

**内部调用与跳转**：

`ChassisBase::init(...)`：调用基类初始化（绑定底盘底层通信）

`setupModelParams()`, `setupLQR()`, `setupBiasParams()`, `setupControlParams()`, `setupThresholdParams()`：跳转到内部的参数配置函数

`boost::bind(&BipedalController::reconfigCB, ...)`：绑定动态调参的回调函数

`kalmanFilterPtr_->clear()`：重置分配的卡尔曼滤波器

#### 2.moveJoint(...)

作用：高频主控循环（如 1kHz）。作为控制器的大脑，负责状态机的跳转评估、传感器状态刷新并最终派发电机控制指令。
内部调用与跳转：
`mode_manager_->switchMode(...)`：更新状态机的目标状态（如坐下、恢复等）。

`updateEstimation(time, period)`：【同文件】跳转更新本周期的全身运动学与位姿状态。

`mode_manager_->getModeImpl()->execute(...)`：多态调用（如跳转到之前列举的 `SitDown::execute` 或 `Balance::execute）`，在该函数内部最终调用 setJointCommands 发送力矩

`pubState()`：【同文件】发布当前底盘特性

#### 3.stopping(...)

作用：当控制器被卸载或急停时触发的安全函数。
内部调用与跳转：

`setJointCommands(joint_handles_, {0,0...})`：强行向所有电机输出 0 的扭矩和速度，保证机器人脱力。

#### 4.updateEstimation(...)

作用：最庞大的数据处理核心。负责解析 IMU 计算欧拉角倾角、使用 VMC（虚拟模型控制） 解算腿部正运动学，并使用卡尔曼滤波进行轮速与加速度的融合，从而得出用来输入 LQR 的状态向量

内部调用与跳转：

`tf2::doTransform(), tf2::fromMsg(), quatToRPY()`：调用 tf2 库，将 IMU 与 base_link 对齐并生成机身 Roll, Pitch, Yaw 角度

`vmc_->leg_pos(), vmc_->leg_spd()`：跳转进入 vmc 模块解算虚拟腿的物理闭环连杆机构，得出虚拟腿长与腿偏角及角速度

`{left/right}_leg_angle_vel_lpFilterPtr_->input() / output()`：调用低通滤波器剔除速度计算中的噪声

`kalmanFilterPtr_->predict(), kalmanFilterPtr_->update(), kalmanFilterPtr_->getState()`：跳转进入卡尔曼滤波算法，传入机体加速度和车轮里程计，吐出最优前向速度

`mode_manager_->getModeImpl()->updateEstimation() / updateLegKinematics() / updateBaseState()`：将计算好的数据喂给当前运行的状态机对象，供 execute() 时使用

#### 5.clearStatus()

作用：状态清零函数，常用于状态切换（如刚起立时）清零历史累积误差

内部调用与跳转：直接通过变量运算重置 x_left_(2)（机身前向位置 X）

#### 6.setupLQR(...)

作用：加载 LQR 权重矩阵，并通过循环遍历一系列“腿长”，离线预先计算求解 LQR 的代数黎卡提方程

内部调用与跳转：
`generateAB(model_params_, a, b, length)`：调用外部物理模型库解析生成指定腿长下的状态转移矩阵𝐴和控制矩阵𝐵

`Lqr<double>::computeK() / getK()`：跳入 Eigen 封装的 LQR 求解器算出反馈增益矩阵𝐾

`polyfit(...)`：【同文件】对上面求出的大量𝐾矩阵进行拟合

#### 7.polyfit(...)

作用：使用最小二乘法将求解出的离散 K 矩阵，关于腿长关于腿长𝐿0拟合成多项式参数阵 coeffs_

内部调用与跳转：

`(A.transpose() * A).ldlt().solve(A.transpose() * B)`：直接调用 Eigen 库底层的线性代数分解器进行回归计算

#### 8.setupModelParams(...) / setupBiasParams(...) / setupControlParams(...) / setupThresholdParams(...)

作用：将 parameter server（ROS yaml配置）上的物理特性（引力、转动惯量、连杆长度）、零点偏置和状态越阶阈值拉取到内存中

内部调用与跳转：

调用 `controller_nh.getParam(...)` 从 ROS 服务器取参数

`std::make_shared<VMC>(l1, l2)`：在 setupModelParams 中基于配置的连杆参数实例化 VMC 对象。
四、 状态共享与ROS通信回调

#### 9.reconfigCB(...)

作用：ROS dynamic_reconfigure（动态调参）回调函数。允许在不重启控制器的情况下，动态修改 LQR 的Q、R 矩阵

内部调用与跳转：
`config_rt_buffer_.writeFromNonRT()`：将修改的参数写入非实时缓冲区，保证硬实时线程`(moveJoint)`不被死锁

内部重复了和 setupLQR 一样的流程：再次呼叫 `generateAB()` `lqr.computeK()` `polyfit()`现场重算控制律并更新

#### 10.odometry()

作用：对外暴露的里程计接口

内部调用与跳转：调用当前状态机节点 `mode_manager_->getModeImpl()->getRealxVel() `获取真实X线速度反馈给高层建图使用

#### 11.状态发布封装函数

`pubLQRStatus(...)`：实时将左右腿 LQR 的误差(e)、参考值(ref)和输出推力推送到 `/lqr_status`（用于绘制波形排错）。内部调用了 `trylock() `保证了实时的非阻塞特性

`pubLegLenStatus(...)`：只负责对外发布机器人当前是否判定处在上楼梯状态

`pubState(...)`：发布腿部车轮防卡死 (unstick) 状态变量。调用路径：`mode_manager_->getModeImpl()->getUnstick()`

## 状态机与模式模块(`ModeManager` 与各 Mode 子类)

**所在文件**：`mode_manager.cpp`, `normal.cpp`, `stand_up.cpp` 等 代码使用状态模式`State Pattern`对机器人的行为进行了解耦

## narmal.cpp

这里集合了 LQR 状态反馈控制、PID 姿态补偿、离地（防卡死）检测以及跳跃动作的状态机

### 1.构造函数Normal::Normal(...)

接收硬件关节句柄 `(joint_handles)`

接收并绑定各个专用 PID 控制器：腿长控制 `(pid_legs)`、偏航角速度控制`（pid_yaw_vel）`、腿角度同步/补偿 `(pid_theta_diff)`、机体防侧翻 Roll 角补偿 `(pid_roll)`

初始化两个大小为 4 的滑动平均滤波器`（MovingAverageFilter）`，用于平滑左右腿计算出的地面支持力，防止在离地检测中因噪声发生误判

### 2.核心函数execute(...)

#### 1.状态进入与基础判定

**入口初始化**：如果刚切入此模式，复位累积量，并将跳跃状态机（jump_phase）重置为 `IDLE`

**起立判定**：判定俯仰角和重力线偏差足够小（稳定）后，记录置位 `CompleteStand`，代表完全站稳

**运动状态判定**：检查速度指令与当前估算速度，判断是否处于“移动”标志(`MoveFlag`)

#### 2.辅助pid控制量计算

**偏航控制 (`T_yaw`)**：利用 PID 根据目标偏航角速度和真实偏航角速度算出一个补偿力矩

**分离腿角强制同步 (`T_theta_diff`)**：双腿运动时需要保持平行，利用 PID 根据左右虚拟腿角的差值算出一个力矩补偿，防止产生不期望的旋转

**横滚防侧翻 (`F_roll`)**：利用 PID 根据机体的 Roll 角尝试通过调整左右腿的相对推力来让机身保持横向水平

**差速/离心力转向补偿**：引入“摩擦圆”限制（`friction_circle_alpha`），当转弯离心力过大时，削弱偏航控制，优先保证不横滑摔倒

#### 3.LQR核心力矩计算

**多项式查表求 **K****：根据左右腿各自当前的**实际腿长** `[left/right]_pos_[0]`，代入多项式系数 `coeffs_` 矩阵中，实时快速计算出分属左右腿的 LQR 状态反馈矩阵 Kleft 和 **K**right

**状态向量误差计算**：用当前的系统状态 X**X** （减去安装偏置）减去期望状态 Xref**X**re**f**（此处 **X**re**f** 含有外部下发的速度目标）。

**算出车轮推力 u**：使用公式 `u=−K乘e**u**=**−**K**乘**e ``{u\_left = k\_left \* (-x\_left)}`得出 LQR 提供基础平衡与跟随的二维控制量构成的列向量（0 号元素用于驱动**轮子转矩 **T**，1号元素用于驱动**髋膝虚拟腿压矩 Tp）即求出了用于维持平衡的轮端目标扭矩 `u_left(0)`和 虚拟髋关节目标扭矩`u_left(1)`

#### 4.腿长控制与跳跃状态机

静力学解算 `(F_leg)`：计算腿部的直线推力（伸缩力）。将重力前馈、离心力惯性项前馈、PID腿长调节、横滚补偿 `(F_roll)`、弹簧配重 `(spring_force)` 加权结合在一起。

跳跃状态机：如果满足冷却时间并在此时按下了“跳跃”。则底层重写伸缩力`F_leg` 进入跳跃接管：

Phase 1 `LEG_RETRACTION`（蓄力缩腿）：提高PID输出，强制身体下降压缩一定周期

Phase 2 `JUMP_UP`（蹬地起跳）：向两腿输出极大的前馈力（基于腿长二次、三次多项式插值的 400N 量级的爆发力矩）

Phase 3 `OFF_GROUND`（腾空收腿）：施加负拉力促使在空中将腿收起越障，并在落地后进入 IDLE 冷却

#### 5.悬空与打滑检测 (Unstick detection)

触发判定：调用 `unstickDetection` 离地算法，看是否有某条腿因为越障或悬崖发生了悬空失去支撑

悬空处理：如果某条腿悬空，强制去除其参与横滚防侧翻的力矩项

发布状态：实时将计算过程中的各个 LQR 参数、离地信号传发到 ROS话题中

#### 6.输出与危险保护 (Control & Protection)

逆运动学与下发：将 VMC 虚拟的腿受力（直线推力F_leg 和 维持重心的扭矩T_p）通过逆向映射算式 `(leg_conv)`，反解为真实的髋关节电机扭矩和膝关节电机扭矩。然后统一利用 setJointCommands 派发

上楼梯跃迁：假如处于非跳跃状态，并且 Z 轴加速度有异常的失重/超重峰值，结合腿长与速度参数，此时判定机器人“撞”上楼梯，强制切换至 `UPSTAIRS` 模式

摔车硬保护：如果位姿过于极端（俯仰/横滚超过 1 弧度）或从框架接收到底盘急停（State 4），强制将所有状态清零并将控制权移交回 `SIT_DOWN`（待机/坐下）以实行安全断电


### 3.力学估算函数calculateSupportForce(...)

作用：基于动力学方程，反向推算当前车轮与地面之间的法向支持力（𝐹𝑛）

逻辑:

读取机身的 Z 轴线加速度、腿长的速度以及虚拟腿角度

通过低通滤波求出加速度微分项（`ddot_zM`机体Z向加速度, `ddot_theta`摆角角加速度）

沿着腿的虚拟支柱线，利用牛顿-欧拉法计算动力学平衡方程算得Fn,其中考虑了轮子质量、离心力、虚拟重力等因素的耦 

### 4.离地状态滤波器函数 unstickDetection(...)
作用：稳定判断某侧腿是否脱离了地面，防止“抬腿打空转”以及计算失真
逻辑：
调用上面的 `calculateSupportForce()` 求取当前瞬间腿部计算得到的支持力Fn
将力丢进滑动平均滤波器`(supportForceAveragePtr)`中平滑，如果平均后的地盘支持力 小于 10N（基本可以认为脱离地面）
时间消抖：使用`ROS::Time`定义了时间迟滞过滤器。支持力小于 10N 的状态必须持续超过 0.1s 才会被盖棺定论确认发生了悬空`(unstick_ = true)`，有效剔除了颠簸抖动带来的误触发错觉调误判



## mode_base.cpp
**该文件的核心作用是：提供一个统一的数据刷新接口集。主控类 controller.cpp 在每个控制周期只需调用这些基类函数，把最新算出的姿态、运动学状态、系统变量输入进当前运行的模式对象中，从而刷新模式内部的成员变量，供各自的 execute() 函数提取使用**

### 1. 状态空间矩阵更新：updateEstimation(...)
作用：更新供控制器使用的状态空间向量
详解：接收主控循环算出的左右两腿各自的 6×1 状态列向量 `(x_left, x_right)`。这个向量包含了 LQR 控制算法所需的系统状态（摆角、摆杆角速度、前向位移、前向线速度、机体俯仰角、俯仰角速度）。赋值给类成员后，在执行 LQR 反馈时就可以直接通过状态差值计算推力

### 2. 腿部运动学参数更新：updateLegKinematics(...)
作用：高效地更新各关节以及 VMC 虚拟腿的正逆运动学状态。
详解：函数的参数全是指针类型（包含真实的关节角度 angle，解算出的虚拟腿长度与摆角 `pos`，以及它们的变化率或即速度 spd）。因为这些参数在 C++ 中是容量固定的纯数组（元素通常大小为 2），所以在这里使用了底层的 std::memcpy 内存拷贝函数来取代 for 循环赋值。这样可以将 16 字节（2 * sizeof(double)）的数据瞬间块复制，最大化保障高频控制在微秒级的实时运行效率。

### 3. 底盘本体姿态更新：updateBaseState(...)
作用：刷新车体的 IMU 传感器融合结果 和位姿信息。
详解：
接收三轴角速度 `angular_vel_base`、三轴线加速度 `linear_acc_base`，以及解算后的欧拉角横滚 `roll`、俯仰 `pitch`、偏航 `yaw`
重点关注 yaw_total_ 的计算：由于欧拉角的 yaw 值一般被限制在[−π,π)之间，在发生整圈旋转时会产生突变截断。这里引用了 ROS 的 `angles::shortest_angular_distance` 方法，利用最短角路径将当前的偏航增量累加到 `yaw_total_ `上，硬算出了一个永远连续、不会跳变截断的绝对偏航角总里程，这对于底盘需要累计转圈或差速转向控制时非常有必要

### 4. 离地/悬空状态更新：updateUnstick(...)
作用：刷新左右车轮在当前周期的触地状态防打滑/卡死标志位。
详解：接收前面模式中通过动力学公式和阈值滤波器算出来的离地侦测结果 (true 为悬空离地/卡死，false 为接触地面正常发力)刷新这个变量后，模式在具体打计算控制指令时就可以据此切断某个轮子的力矩输出以免发生“起飞空转”

## stand_up.coo
**从sit_ddown状态下通过普通的PID控制过渡到LQR平衡控制，避免初值误差太大导致抽搐**

### 核心控制函数 execute(...)
作用：起立模式的主控函数，统筹两条腿进行起立阶段的动作调配
具体逻辑：
1.初入逻辑：记录第一次进入状态，设置未完成起立标志，并且调用 `detectLegState`，基于当前双腿的躺地夹角判断左腿和右腿分别处于什么状态（UNDER在身下、FRONT在前方或 BEHIND在后方

2.规划下发参数：分别调用 `setUpLegMotion`，评估左右腿当前应该去往的目标长度 (length_des) 和目标角度 (theta_des)，以及判断该腿当前是否应该处于停机等待 (stop_flag)

3.计算力矩：利用`computePidLegCommand`算出两条腿对应要求的实际关节输出，并通过`setJointCommands`将指令发布。起立阶段轮毂电机默认不需要发力

4.退出条件：判断当左右腿都达到了后方直立收缩预备状态 `(BEHIND)` 并且虚腿偏角足够小时，说明机器人此时的姿态已经可以近似视为处于“受控倒立摆”局部线性区了，因此关闭起立模式，无缝切入 LQR 对接好的 `NORMAL` 正常平衡模式

### 2. 状态诊断函数 detectLegState(...)
作用：判断单条腿当前跟地面的拓扑几何关系，以此作为有限状态机（FSM）跃迁的依据
具体逻辑：传入底盘针对某条腿的状态向量 x，读取里面记录的当前腿的虚拟绝对偏转倾角 x[0]
根据从 ROS 参数服务器载入的各角度阈值 (leg_state_threshold_) 判断。如果腿跟机身近似重合并向下压身，识别为 UNDER（压在下方）
**如果**腿往前踢且比较平，识别为 FRONT（由于关节正反向或者朝前倒）
**如果**在后方，则识别为 BEHIND

### 3. 行动步序生成函数 setUpLegMotion(...)
作用：核心运动序列机。根据当前腿处于什么样倒立状态，派发不同的起立策略。双腿交互配合，防止起立时互相打架。
具体逻辑：
1.如果是 UNDER (压在身下)：先让它把腿伸长到 0.36m 作为支撑，当腿伸长超过 0.35 后，判定它起到了抬高支点的作用，将该腿模式推进为 FRONT

2.如果是 FRONT (腿在前侧)：将虚拟腿期望角度设定为一个斜向下的角度（如90∘偏下一点以支撑机体重心），并继续保持伸长发力。当机台角度逐渐正向，并且腿部角度回归到了合理区域，判定可以进入最终收缩，切换状态到 BEHIND

3.如果是 BEHIND (腿在后侧，即近直立态)：处于起立准备末期。首先如果对面的另一条腿还在尴尬的 FRONT 阶段努力支撑抬高重心，那么这条腿就锁死 (stop_flag = true) 不要动。如果另一条腿也来到了近位直立期（非 FRONT），那么允许这两条腿同时把腿长目标强行缩短到 0.18 米，并朝向一个极小的微偏角发力——即蓄力下蹲，从而完成最后的身体顶起，交给 LQR 接管

### 4. VMC与PID组合控制函数 computePidLegCommand(...)
作用：依据上面序列机给出的期待长度和角度参数，使用纯 PID 计算当前需要的 VMC 虚拟推力，反解出实体电机的所需扭矩。
具体逻辑：
1.腿长推力 (F)：使用输入的 `length_pid` 和预设弹簧力 `feedforward_force` 相结合，计算虚拟支柱直冲的推力进行了 250N 的硬限幅进行保护

2.摆角力矩 (τ)：
如果腿位于最终的收缩起立期（BEHIND或UNDER），使用角度的位置环 PID（angle_pid）拉伸并紧紧跟随目标角度。

3.如果还处于刚开始起立的尴尬摔倒期（FRONT），采取角速度环 PID 取代角度环（angle_vel_pid），强行迫使虚拟腿按照恒定的角速度 (-5 rad/s) 翻转腿部

4.VMC分解：通过 `vmcPtr_->leg_conv` 把计算出的虚拟弹簧推力和力矩拆包转换给该腿实体的髋关节和膝关节。


## upstairs.cpp

**实现上台阶功能，在之前的`normal.cpp`中，我们注意到一个判定：当机器人在正常行驶中，如果 Z 轴（垂直方向）出现极大的突变负向加速度`（linear_acc_base_.z < -7.0）`，且处于较高行进速度时，控制器会认为机器人“撞上了楼梯或路沿”。此时，如果继续使用线性微调的 LQR 算法必然会导致翻车摔倒，因此控制器会将模式强制切入到这个 Upstairs 越障模式。这也是一个基于纯 PID 和 VMC 的脱机开环过渡状态。它的核心策略是：迅速收起双腿（缩短腿长），并向前翻折一定角度，使得底盘越过障碍物边缘，在完成姿态跨越后重新切回起立模式恢复平衡**

### upstairs构造函数
作用：初始化“上楼梯”模式对象
逻辑：接收底盘的硬件关节句柄 (joint_handles)，以及专门用于腿长控制和腿部摆角控制的 PID 控制器数组（这里直接复用了跟 StandUp 模式中相同的 PID 控制器 pid_legs_stand_up_ 和 pid_thetas_，因为二者同属于非 LQR 接管的姿态控制）

### 核心控制函数 execute(...)
作用：控制主函数
具体逻辑：
初入逻辑 (Entry)：当刚切入此模式时，重置完全站立标志位（`setCompleteStand(false)`，意味着放弃 LQR 平衡），获取虚拟模型（VMC）参数，并检测双腿当前的姿态 `detectLegState`
设定越障目标位姿：
无论双腿之前处于什么状态，在这里强行接管轮毂电机的驱动权
强行规定目标腿长 `length_des_l / r = 0.18`，将腿缩到极短
强行规定目标虚拟腿角度 theta_des 为从参数服务器获取的 `upstair_des_theta`
计算推力/力矩并下发：调用 `computePidLegCommand()` 根据上面的 0.18m 长度和配置角度计算出需要的推力，再经由 VMC 转化为两个关节电机的电流力矩并下发。
越障退出评估 (Exit)：
条件：当左右双腿的实际长度都被成功压缩到 0.2m 以下 (left_pos_[0] < 0.2)，且腿部的实际夹角超越了设定的安全门限角 `(upstair_exit_threshold)`
动作：这说明机体已经顺着冲量“爬”上或“扑”上了台阶。此时发送一个“完成上楼梯”的 ROS 消息。接着，将模式移交给 STAND_UP (起立模式)，让机器人在台阶上方重新像开机时那样从摔倒状态下爬起来。
