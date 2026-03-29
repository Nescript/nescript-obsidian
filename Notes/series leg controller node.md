文件位置 /bipedal_wheel_controller/controller.cpp

继承自chassis base，根据之前的印象，chassis base实现了底盘的通用逻辑，功率限制。其子控制器要完成movejoint成员函数，这个函数每个周期会运行一次，实现控制循环

1. **What does this code touch?** (inputs and outputs)  
	**这段代码涉及哪些方面？** （输入和输出）
2. **What does it change?** (state and side effects)  
	**它会改变什么？** （状态和副作用）
3. **What does it assume?** (defaults and invariants)  
	**它做了哪些假设？** （默认值和不变式）
4. **Where does it fail?** (exceptions, errors, edge cases)  
	**它在哪里失效？** （异常情况、错误、极端情况）
5. **What does it hide?** (abstractions and libraries)  
	**它隐藏了什么？** （抽象和库）

# 涉及什么方面
这段代码涉及LQR矩阵的的动态调整，控制循环，
## init
- 腿部关节句柄的获取，imu句柄的获取
- 动态调参服务器初始化
- mode_manager、model_params、control_params、leg_threshold_params_ 的实例化，还有别的滤波器之类的实例
- 从参数服务器初始化上面所有的东西
- 话题订阅和发布者注册
## moveJoint
- 控制循环,每周期执行
- 主要是切换状态，如果状态没有变化，那么就给mode manager switchmode上一个周期的状态
- 接收ChassisCmd,如果FALLEN就主动SIT_DOWN
- overturn是用来追踪是否翻到的标志，这里允许在没有自动检测到overtunn时通过chassiscmd来主动起身进RECOVERY
- 每周期评估自己的状态
- 这里有一句 `mode_manager_->getModeImpl()->execute(this, time, period);` 尚不知目的，这个mode_manager是一个要追踪的路径
- 每周期发布状态
## updateEstimation
- 读取imu数据，填充角速度和线速度，
- 将底盘imu数据变换到底盘坐标系（巧妙的一步，底盘imu的装配误差和别的什么误差由此可以去urdf里的offset中修正）
- 将关节电机编码器的角度数据转化到vmc模型下的角度
- 接着使用vmc计算腿部伸缩的速度和角度 leg_pos 和 leg_spd 存疑问一下
- 计算wheel vel,两轮速度平均值
- 有一个打滑适应的设计：配合卡尔曼滤波，这是一条路径，整个 updateEstimation 值得单独开一个md追踪具体实现
## pubState
发布unstick信息，这是啥
## stopping
电机command置0,然后模式切换到SIT_DOWN
## setupModelParams
初始化模型参数，