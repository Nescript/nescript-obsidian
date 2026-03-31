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
```cpp
void BipedalController::updateEstimation(const ros::Time& time, const ros::Duration& period)
{
  geometry_msgs::Vector3 angular_vel_imu, linear_acc_imu;
  angular_vel_imu.x = imu_handle_.getAngularVelocity()[0];
  angular_vel_imu.y = imu_handle_.getAngularVelocity()[1];
  angular_vel_imu.z = imu_handle_.getAngularVelocity()[2];
  linear_acc_imu.x = imu_handle_.getLinearAcceleration()[0];
  linear_acc_imu.y = imu_handle_.getLinearAcceleration()[1];
  linear_acc_imu.z = imu_handle_.getLinearAcceleration()[2];
  // 读取imu的信息

  tf2::Transform odom2imu, imu2base, odom2base;
  // 准备变换

  geometry_msgs::Vector3 angular_vel_base{}, linear_acc_base{};
  double roll{}, pitch{}, yaw{};
  try
  {
    tf2::doTransform(angular_vel_imu, angular_vel_base,
                     robot_state_handle_.lookupTransform("base_link", imu_handle_.getFrameId(), time));
                    // 这一句是将imu的角速度变换到base_link坐标系,该函数一共三个参数,第一个输入第二个输出,第三个就是应用的变换

    geometry_msgs::TransformStamped imu2base_msg;
    imu2base_msg = robot_state_handle_.lookupTransform(imu_handle_.getFrameId(), "base_link", time);
    // 这里用msg类型是因为lookupTransform拿到的就是消息类型，所以下一句根据msg建立对应的tf类型

    tf2::fromMsg(imu2base_msg.transform, imu2base);
    tf2::Quaternion odom2imu_quaternion;
    tf2::Vector3 odom2imu_origin;
    odom2imu_quaternion.setValue(imu_handle_.getOrientation()[0], imu_handle_.getOrientation()[1],
                                 imu_handle_.getOrientation()[2], imu_handle_.getOrientation()[3]);
                                 // 这里imu的旋转就是imu相对于odom的旋转，即odom2imu（odom经过这样的旋转变成imu的位姿）
    odom2imu_origin.setValue(0, 0, 0);
    // 假设原点不偏移
    odom2imu.setOrigin(odom2imu_origin);
    odom2imu.setRotation(odom2imu_quaternion);
    // 一个Transform包含原点偏移和旋转两个部分，这里分别填充，使得odom2imu成为了一个可用的变换
    odom2base = odom2imu * imu2base;
    // 叠加变换得到odom2base
    quatToRPY(toMsg(odom2base).rotation, roll, pitch, yaw);
    // 现在提取出 roll pitch 和 yaw
    
    geometry_msgs::TransformStamped imu2odom_msg;
    imu2odom_msg.transform = tf2::toMsg(odom2imu.inverse());
    imu2odom_msg.header.stamp = time;
    tf2::doTransform(linear_acc_imu, linear_acc_base, imu2odom_msg);

    tf2::Vector3 z_body(0, 0, 1);
    tf2::Vector3 z_world = tf2::quatRotate(odom2base.getRotation(), z_body);
    overturn_ = (abs(pitch) > 0.65 || abs(roll) > 0.4) && z_world.z() < 0.0;
  }
  catch (tf2::TransformException& ex)
  {
    ROS_WARN("%s", ex.what());
    setJointCommands(joint_handles_, { 0, 0, { 0., 0. } }, { 0, 0, { 0., 0. } });
    return;
  }

  // vmc
  double left_angle[2]{}, right_angle[2]{}, left_pos[2]{}, left_spd[2]{}, right_pos[2]{}, right_spd[2]{};
  // [0]:hip_vmc_joint [1]:knee_vmc_joint
  left_angle[0] = left_hip_joint_handle_.getPosition() + M_PI;
  left_angle[1] = left_knee_joint_handle_.getPosition();
  right_angle[0] = right_hip_joint_handle_.getPosition() + M_PI;
  right_angle[1] = right_knee_joint_handle_.getPosition();

  // gazebo
  //  left_angle[0] = left_hip_joint_handle_.getPosition() + M_PI_2;
  //  left_angle[1] = left_knee_joint_handle_.getPosition() - M_PI_2;
  //  right_angle[0] = right_hip_joint_handle_.getPosition() + M_PI_2;
  //  right_angle[1] = right_knee_joint_handle_.getPosition() - M_PI_2;

  // [0] is length, [1] is angle
  vmc_->leg_pos(left_angle[0], left_angle[1], left_pos);
  vmc_->leg_pos(right_angle[0], right_angle[1], right_pos);
  vmc_->leg_spd(left_hip_joint_handle_.getVelocity(), left_knee_joint_handle_.getVelocity(), left_angle[0],
                left_angle[1], left_spd);
  vmc_->leg_spd(right_hip_joint_handle_.getVelocity(), right_knee_joint_handle_.getVelocity(), right_angle[0],
                right_angle[1], right_spd);
  left_leg_angle_vel_lpFilterPtr_->input(left_spd[1]);
  right_leg_angle_vel_lpFilterPtr_->input(right_spd[1]);
  //  left_leg_angle_lpFilterPtr_->input(left_pos[1]);
  //  right_leg_angle_lpFilterPtr_->input(right_pos[1]);

  //  left_pos[1] = left_leg_angle_lpFilterPtr_->output();
  //  right_pos[1] = right_leg_angle_lpFilterPtr_->output();
  left_spd[1] = left_leg_angle_vel_lpFilterPtr_->output();
  right_spd[1] = right_leg_angle_vel_lpFilterPtr_->output();

  // Slippage_detection
  leftWheelVel = (left_wheel_joint_handle_.getVelocity() + angular_vel_base.y + left_spd[1]) * wheel_radius_;
  rightWheelVel = (right_wheel_joint_handle_.getVelocity() + angular_vel_base.y + right_spd[1]) * wheel_radius_;
  leftWheelVelAbsolute =
      leftWheelVel + left_pos[0] * left_spd[1] * cos(left_pos[1] + pitch_) + left_spd[0] * sin(left_pos[1] + pitch_);
  rightWheelVelAbsolute = rightWheelVel + right_pos[0] * right_spd[1] * cos(right_pos[1] + pitch_) +
                          right_spd[0] * sin(right_pos[1] + pitch_);

  double wheel_vel_aver = (leftWheelVelAbsolute + rightWheelVelAbsolute) / 2.;
  R_(0, 0) = slip_flag_ ? slip_R_wheel_ : R_wheel_;
  if (i >= sample_times_)
  {  // oversampling
    i = 0;
    X_(0) = wheel_vel_aver;
    X_(1) = linear_acc_base.x;
    kalmanFilterPtr_->predict(U_);
    kalmanFilterPtr_->update(X_, R_);
  }
  else
  {
    kalmanFilterPtr_->predict(U_);
    i++;
  }
  auto x_hat_vel = kalmanFilterPtr_->getState();
  slip_flag_ = abs(x_hat_vel(0) - wheel_vel_aver) > 3.0;

  // update state
  x_left_[3] = state_ != RAW ? x_hat_vel(0) : 0;
  if (abs(x_left_[3]) <= 0.6f && abs(vel_cmd_.x) <= 0.1f)
  {
    x_left_[2] += state_ != RAW ? x_left_[3] * period.toSec() : 0;
  }
  else
  {
    x_left_[2] = -bias_params_->x;
  }
  x_left_[0] = (left_pos[1] + pitch);
  x_left_[1] = left_spd[1] + angular_vel_base.y;
  x_left_[4] = -pitch;
  x_left_[5] = -angular_vel_base.y;
  x_right_ = x_left_;
  x_right_[0] = (right_pos[1] + pitch);
  x_right_[1] = right_spd[1] + angular_vel_base.y;

  if (legged_chassis_status_pub_->trylock())
  {
    legged_chassis_status_pub_->msg_.roll = roll;
    legged_chassis_status_pub_->msg_.pitch = x_left_[4];
    legged_chassis_status_pub_->msg_.d_pitch = x_left_[5];
    legged_chassis_status_pub_->msg_.yaw = yaw;
    legged_chassis_status_pub_->msg_.d_yaw = angular_vel_base.z;
    legged_chassis_status_pub_->msg_.left_leg_length = left_pos[0];
    legged_chassis_status_pub_->msg_.right_leg_length = right_pos[0];
    legged_chassis_status_pub_->msg_.x = x_left_[2];
    legged_chassis_status_pub_->msg_.x_dot = x_left_[3];
    legged_chassis_status_pub_->msg_.left_leg_theta = x_left_[0];
    legged_chassis_status_pub_->msg_.left_leg_theta_dot = x_left_[1];
    legged_chassis_status_pub_->msg_.right_leg_theta = x_right_[0];
    legged_chassis_status_pub_->msg_.right_leg_theta_dot = x_right_[1];
    legged_chassis_status_pub_->msg_.linear_acc_base.push_back(linear_acc_base.x);
    legged_chassis_status_pub_->msg_.linear_acc_base.push_back(linear_acc_base.y);
    legged_chassis_status_pub_->msg_.linear_acc_base.push_back(linear_acc_base.z);
    legged_chassis_status_pub_->unlockAndPublish();
  }

  if (legged_chassis_mode_pub_->trylock())
  {
    legged_chassis_mode_pub_->msg_.mode = balance_mode_;
    legged_chassis_mode_pub_->msg_.mode_name = mode_manager_->getModeImpl()->name();
    legged_chassis_mode_pub_->unlockAndPublish();
  }

  mode_manager_->getModeImpl()->updateEstimation(x_left_, x_right_);
  mode_manager_->getModeImpl()->updateLegKinematics(left_angle, right_angle, left_pos, left_spd, right_pos, right_spd);
  mode_manager_->getModeImpl()->updateBaseState(angular_vel_base, linear_acc_base, roll, pitch, yaw);
}

void BipedalController::pubState()
{
  std_msgs::Bool msg;
  msg.data = mode_manager_->getModeImpl()->getUnstick();
  unstick_pub_.publish(msg);
}

void BipedalController::stopping(const ros::Time& time)
{
  balance_mode_ = BalanceMode::SIT_DOWN;
  balance_state_changed_ = false;
  setJointCommands(joint_handles_, { 0, 0, { 0., 0. } }, { 0, 0, { 0., 0. } });

  ROS_INFO("[balance] Controller Stop");
}
```
## pubState
发布unstick信息，这是啥
## stopping
电机command置0,然后模式切换到SIT_DOWN
## setupModelParams
初始化模型参数，建立了一个pair和一个model_param，来快速输入和统一维护模型参数
## setupLQR

