# DM Motor Hardware Interface for ROS2 Control

ROS2 Control hardware interface plugin for DM motors (DM-J4310-2EC and DM4340) supporting multiple CAN interfaces.

## TODO

**fix:**

与move_group同时启动时，会出现 servo_node 等待 robot state update 的问题。

一直打印提示：`[servo_node]: Waiting to receive robot state update. `

参考：

[generic_system.hpp](https://github.com/ros-controls/ros2_control/blob/jazzy/hardware_interface/include/mock_components/generic_system.hpp)

[generic_system.cpp](https://github.com/ros-controls/ros2_control/blob/jazzy/hardware_interface/src/mock_components/generic_system.cpp)

[franka_hardware](https://github.com/frankarobotics/franka_ros2/blob/jazzy/franka_hardware/include/franka_hardware/franka_hardware_interface.hpp)

## Features

- Support for DM-J4310-2EC and DM4340 motors
- MIT control mode for smooth position/velocity control
- **Multiple CAN interface support:**
  - SocketCAN (Linux native CAN interface)
  - USB-CAN adapter (WitMotion USB-CAN and compatible devices)
- Real-time motor state feedback (position, velocity, torque, temperature)
- Configurable per-joint motor parameters

## Motor Configuration

### Joint Assignment
- Joint 1 (J1): DM-J4310-2EC
- Joint 2 (J2): DM4340
- Joint 3 (J3): DM4340
- Joint 4 (J4): DM-J4310-2EC
- Joint 5 (J5): DM-J4310-2EC
- Joint 6 (J6): DM-J4310-2EC

### Motor Parameters

**DM-J4310-2EC:**
- Max Position: ±12.5 rad
- Max Velocity: 30.0 rad/s
- Max Torque: 10.0 Nm

**DM4340:**
- Max Position: ±12.5 rad
- Max Velocity: 8.0 rad/s
- Max Torque: 28.0 Nm

## Prerequisites

Choose one of the following CAN interface options:

### Option 1: SocketCAN (Linux Native CAN)

#### 1.1 Install SocketCAN utilities

```bash
sudo apt-get update
sudo apt-get install can-utils
```

#### 1.2 Configure CAN Interface

Create a script to configure the CAN interface (e.g., `/etc/systemd/system/can-setup.sh`):

```bash
#!/bin/bash
# Setup CAN interface
sudo ip link set can0 type can bitrate 1000000
sudo ip link set up can0
```

Make it executable:
```bash
chmod +x /etc/systemd/system/can-setup.sh
```

To automatically setup CAN on boot, create a systemd service (`/etc/systemd/system/can-setup.service`):

```ini
[Unit]
Description=Setup CAN interface
After=network.target

[Service]
Type=oneshot
ExecStart=/etc/systemd/system/can-setup.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable can-setup.service
sudo systemctl start can-setup.service
```

#### 1.3 Verify CAN Interface

```bash
# Check if CAN interface is up
ip link show can0

# Monitor CAN traffic
candump can0

# Send test frame
cansend can0 123#DEADBEEF
```

### Option 2: USB-CAN Adapter (WitMotion and Compatible)

#### 2.1 Connect USB-CAN Adapter

Connect your USB-CAN adapter to the computer. The adapter should appear as a serial device.

#### 2.2 Find Serial Port

```bash
# List available serial ports
ls /dev/ttyUSB* /dev/ttyACM*

# Or use dmesg to see the device
dmesg | grep tty
```

Common ports:
- `/dev/ttyUSB0` - Most USB-serial adapters
- `/dev/ttyACM0` - Some USB-CDC adapters
- `COM3`, `COM4`, etc. - Windows (not officially supported yet)

#### 2.3 Set Serial Port Permissions

Add your user to the `dialout` group to access serial ports:

```bash
sudo usermod -a -G dialout $USER
```

Log out and log back in for changes to take effect.

Alternatively, create a udev rule (`/etc/udev/rules.d/99-usb-can.rules`):

```
# WitMotion USB-CAN adapter
SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", ATTRS{idProduct}=="7523", MODE="0666", GROUP="dialout"
```

Reload udev rules:
```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

#### 2.4 Test USB-CAN Connection

You can test the USB-CAN adapter using the Python driver:

```bash
cd ~/ws/mockway_robotics/tools/motor_gui
python3 dm_motor_driver.py
```

Or use a serial terminal:
```bash
# Install minicom if not already installed
sudo apt-get install minicom

# Connect to the adapter (adjust port and baudrate)
minicom -D /dev/ttyUSB0 -b 921600

# Send AT command to test
AT+AT
# Should receive: OK
```

## Building

```bash
cd ~/ws/mockway_robotics
colcon build --packages-select dmmotor_hardware_interface
source install/setup.bash
```

## Configuration

The hardware interface is configured in the URDF/xacro file (`moveit_mockway_config/config/mockway_description.ros2_control.xacro`).

### Hardware Parameters

#### For SocketCAN

```xml
<hardware>
    <plugin>dmmotor_hardware_interface/DMMototHardwareInterface</plugin>
    <param name="can_interface_type">socketcan</param>  <!-- Optional, default is socketcan -->
    <param name="can_port">can0</param>
    <param name="can_baudrate">1000000</param>
    <param name="position_kp">40.0</param>
    <param name="position_kd">1.0</param>
</hardware>
```

#### For USB-CAN Adapter

```xml
<hardware>
    <plugin>dmmotor_hardware_interface/DMMototHardwareInterface</plugin>
    <param name="can_interface_type">usb_can</param>
    <param name="can_port">/dev/ttyUSB0</param>           <!-- Serial port -->
    <param name="serial_baudrate">921600</param>          <!-- Serial baudrate, default: 921600 -->
    <param name="can_baudrate">1000000</param>            <!-- CAN baudrate -->
    <param name="position_kp">40.0</param>
    <param name="position_kd">1.0</param>
</hardware>
```

**Parameter Reference:**
- `can_interface_type`: Interface type - `socketcan` or `usb_can` (default: `socketcan`)
- `can_port`: For SocketCAN: interface name (e.g., `can0`). For USB-CAN: serial port (e.g., `/dev/ttyUSB0`)
- `can_baudrate`: CAN bus baudrate in bps (typically 1000000 for 1 Mbps)
- `serial_baudrate`: (USB-CAN only) Serial port baudrate (default: 921600)
- `position_kp`: Position control proportional gain [0-500]
- `position_kd`: Position control derivative gain [0-5]

### Joint Parameters

For each joint, specify:
- `motor_id`: CAN ID of the motor (1-127)
- `master_id`: CAN ID for receiving feedback (typically 0)
- `motor_type`: Either `DM_J4310_2EC` or `DM4340`

Example:
```xml
<joint name="joint1">
    <command_interface name="position"/>
    <state_interface name="position"/>
    <state_interface name="velocity"/>
    <param name="motor_id">1</param>
    <param name="master_id">0</param>
    <param name="motor_type">DM_J4310_2EC</param>
</joint>
```

## Usage

### Starting the Hardware Interface

The hardware interface is automatically loaded by ros2_control when you launch MoveIt:

```bash
ros2 launch moveit_mockway_config demo.launch.py
```

### Motor Control Modes

The hardware interface uses **MIT control mode** for position control:

```
torque = kp * (p_cmd - p_actual) + kd * (v_cmd - v_actual)
```

Control parameters (`position_kp` and `position_kd`) can be tuned in the xacro file.

### Monitoring Motor Status

```bash
# Check joint states
ros2 topic echo /joint_states

# Monitor controller status
ros2 control list_controllers

# View hardware component status
ros2 control list_hardware_components
```

## Troubleshooting

### CAN Interface Issues

#### SocketCAN Issues

**Problem:** `Failed to open CAN interface can0`

**Solutions:**
1. Check if CAN interface exists: `ip link show can0`
2. Bring up the interface: `sudo ip link set up can0`
3. Verify bitrate: `ip -details link show can0`
4. Check CAN hardware connection

#### USB-CAN Issues

**Problem:** `Failed to open serial port /dev/ttyUSB0`

**Solutions:**
1. Check if device exists: `ls -l /dev/ttyUSB0`
2. Verify USB connection: `dmesg | tail`
3. Check permissions: `sudo usermod -a -G dialout $USER` (log out/in after)
4. Try different USB ports
5. Check USB cable quality

**Problem:** `Warning: Failed to enter AT mode`

**Solutions:**
1. Verify serial baudrate is correct (typically 921600)
2. Try power-cycling the USB-CAN adapter
3. Check if adapter is in correct mode
4. Some adapters may work without AT mode confirmation - check if CAN communication works

**Problem:** `Permission denied when accessing serial port`

**Solutions:**
1. Add user to dialout group: `sudo usermod -a -G dialout $USER`
2. Log out and log back in
3. Or use udev rules (see USB-CAN setup section)
4. Temporary fix: `sudo chmod 666 /dev/ttyUSB0`

**Problem:** `No feedback from motors`

**Solutions:**
1. Check CAN wiring and termination resistors (120Ω at both ends)
2. Verify motor power supply
3. Monitor CAN bus: `candump can0`
4. Check motor IDs match configuration
5. Ensure motors are properly enabled

**Problem:** `Motors not responding to commands`

**Solutions:**
1. Verify motors are enabled (check error codes)
2. Check motor ID configuration
3. Ensure CAN baudrate matches (1 Mbps)
4. Verify MIT control parameters are reasonable

### Error Codes

Motor error codes are reported in feedback:
- `0x0`: Disabled
- `0x1`: Enabled (normal operation)
- `0x8`: Over-voltage
- `0x9`: Under-voltage
- `0xA`: Over-current
- `0xB`: MOS over-temperature
- `0xC`: Coil over-temperature
- `0xD`: Communication loss
- `0xE`: Overload

## Safety Notes

1. **Emergency Stop**: Always have an emergency stop mechanism
2. **Workspace Limits**: Ensure joint limits are properly configured
3. **Temperature Monitoring**: Monitor motor temperatures during operation
4. **Power Supply**: Use appropriate power supply (check motor specifications)
5. **Initial Testing**: Test with low velocities and accelerations initially

## Development and Debugging

### Enable Debug Logging

```bash
ros2 launch moveit_mockway_config demo.launch.py --ros-args --log-level dmmotor_hardware_interface:=debug
```

### Test Individual Motors

Use the Python driver for testing:

```bash
cd ~/ws/mockway_robotics/tools/motor_gui
python3 dm_motor_driver.py
```

## References

- [ROS2 Control Documentation](https://control.ros.org/)
- [SocketCAN Documentation](https://www.kernel.org/doc/Documentation/networking/can.txt)
- DM Motor Manual (refer to manufacturer documentation)

## License

MIT License - Copyright (c) 2025 Mockway Robotics

# DM 电机 ROS2 Control 硬件接口

用于 DM 电机（DM-J4310-2EC 和 DM4340）的 ROS2 Control 硬件接口插件，支持多种 CAN 接口。

## TODO

**修复：**

与 move_group 同时启动时，会出现 servo_node 等待 robot state update 的问题。

一直打印提示：`[servo_node]: Waiting to receive robot state update.`

参考：

[generic_system.hpp](https://github.com/ros-controls/ros2_control/blob/jazzy/hardware_interface/include/mock_components/generic_system.hpp)

[generic_system.cpp](https://github.com/ros-controls/ros2_control/blob/jazzy/hardware_interface/src/mock_components/generic_system.cpp)

[franka_hardware](https://github.com/frankarobotics/franka_ros2/blob/jazzy/franka_hardware/include/franka_hardware/franka_hardware_interface.hpp)

## 功能特性

- 支持 DM-J4310-2EC 和 DM4340 电机
- MIT 控制模式，实现平滑的位置/速度控制
- **多 CAN 接口支持：**
  - SocketCAN（Linux 原生 CAN 接口）
  - USB-CAN 适配器（维特智能 USB-CAN 及兼容设备）
- 实时电机状态反馈（位置、速度、扭矩、温度）
- 可配置的各关节电机参数

## 电机配置

### 关节分配

- 关节 1（J1）：DM-J4310-2EC
- 关节 2（J2）：DM4340
- 关节 3（J3）：DM4340
- 关节 4（J4）：DM-J4310-2EC
- 关节 5（J5）：DM-J4310-2EC
- 关节 6（J6）：DM-J4310-2EC

### 电机参数

**DM-J4310-2EC：**

- 最大位置：±12.5 rad
- 最大速度：30.0 rad/s
- 最大扭矩：10.0 Nm

**DM4340：**

- 最大位置：±12.5 rad
- 最大速度：8.0 rad/s
- 最大扭矩：28.0 Nm

## 前置条件

选择以下 CAN 接口方案之一：

### 方案一：SocketCAN（Linux 原生 CAN）

#### 1.1 安装 SocketCAN 工具

```bash
sudo apt-get update
sudo apt-get install can-utils
```

#### 1.2 配置 CAN 接口

创建配置 CAN 接口的脚本（例如 `/etc/systemd/system/can-setup.sh`）：

```bash
#!/bin/bash
# 配置 CAN 接口
sudo ip link set can0 type can bitrate 1000000
sudo ip link set up can0
```

赋予可执行权限：

```bash
chmod +x /etc/systemd/system/can-setup.sh
```

如需开机自动配置 CAN，创建 systemd 服务（`/etc/systemd/system/can-setup.service`）：

```ini
[Unit]
Description=配置 CAN 接口
After=network.target

[Service]
Type=oneshot
ExecStart=/etc/systemd/system/can-setup.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

启用服务：

```bash
sudo systemctl daemon-reload
sudo systemctl enable can-setup.service
sudo systemctl start can-setup.service
```

#### 1.3 验证 CAN 接口

```bash
# 查看 CAN 接口是否已启动
ip link show can0

# 监听 CAN 流量
candump can0

# 发送测试帧
cansend can0 123#DEADBEEF
```

### 方案二：USB-CAN 适配器（维特智能及兼容设备）

#### 2.1 连接 USB-CAN 适配器

将 USB-CAN 适配器连接到电脑，适配器将显示为串口设备。

#### 2.2 查找串口号

```bash
# 列出可用的串口
ls /dev/ttyUSB* /dev/ttyACM*

# 或使用 dmesg 查看设备
dmesg | grep tty
```

常见端口：

- `/dev/ttyUSB0` — 大多数 USB-串口适配器
- `/dev/ttyACM0` — 部分 USB-CDC 适配器
- `COM3`、`COM4` 等 — Windows（暂未官方支持）

#### 2.3 设置串口权限

将用户添加到 `dialout` 组以访问串口：

```bash
sudo usermod -a -G dialout $USER
```

注销后重新登录使更改生效。

或者，创建 udev 规则（`/etc/udev/rules.d/99-usb-can.rules`）：

```
# 维特智能 USB-CAN 适配器
SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", ATTRS{idProduct}=="7523", MODE="0666", GROUP="dialout"
```

重新加载 udev 规则：

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

#### 2.4 测试 USB-CAN 连接

可以使用 Python 驱动测试 USB-CAN 适配器：

```bash
cd ~/ws/mockway_robotics/tools/motor_gui
python3 dm_motor_driver.py
```

或使用串口终端：

```bash
# 如未安装 minicom 则先安装
sudo apt-get install minicom

# 连接适配器（根据实际端口和波特率调整）
minicom -D /dev/ttyUSB0 -b 921600

# 发送 AT 命令测试
AT+AT
# 应返回：OK
```

## 编译

```bash
cd ~/ws/mockway_robotics
colcon build --packages-select dmmotor_hardware_interface
source install/setup.bash
```

## 配置

硬件接口在 URDF/xacro 文件中配置（`moveit_mockway_config/config/mockway_description.ros2_control.xacro`）。

### 硬件参数

#### SocketCAN 配置

```xml
<hardware>
<plugin>dmmotor_hardware_interface/DMMototHardwareInterface</plugin>
<param name="can_interface_type">socketcan</param>  <!-- 可选，默认为 socketcan -->
<param name="can_port">can0</param>
<param name="can_baudrate">1000000</param>
<param name="position_kp">40.0</param>
<param name="position_kd">1.0</param>
</hardware>
```

#### USB-CAN 适配器配置

```xml
<hardware>
<plugin>dmmotor_hardware_interface/DMMototHardwareInterface</plugin>
<param name="can_interface_type">usb_can</param>
<param name="can_port">/dev/ttyUSB0</param>           <!-- 串口号 -->
<param name="serial_baudrate">921600</param>          <!-- 串口波特率，默认：921600 -->
<param name="can_baudrate">1000000</param>            <!-- CAN 波特率 -->
<param name="position_kp">40.0</param>
<param name="position_kd">1.0</param>
</hardware>
```

**参数说明：**

- `can_interface_type`：接口类型 — `socketcan` 或 `usb_can`（默认：`socketcan`）
- `can_port`：SocketCAN 时为接口名（如 `can0`）；USB-CAN 时为串口号（如 `/dev/ttyUSB0`）
- `can_baudrate`：CAN 总线波特率（单位 bps，通常为 1000000，即 1 Mbps）
- `serial_baudrate`：（仅 USB-CAN）串口波特率（默认：921600）
- `position_kp`：位置控制比例增益 [0-500]
- `position_kd`：位置控制微分增益 [0-5]

### 关节参数

每个关节需指定：

- `motor_id`：电机的 CAN ID（1-127）
- `master_id`：接收反馈的 CAN ID（通常为 0）
- `motor_type`：`DM_J4310_2EC` 或 `DM4340`

示例：

```xml
<joint name="joint1">
<command_interface name="position"/>
<state_interface name="position"/>
<state_interface name="velocity"/>
<param name="motor_id">1</param>
<param name="master_id">0</param>
<param name="motor_type">DM_J4310_2EC</param>
</joint>
```

## 使用方法

### 启动硬件接口

启动 MoveIt 时，ros2_control 会自动加载硬件接口：

```bash
ros2 launch moveit_mockway_config demo.launch.py
```

### 电机控制模式

硬件接口使用 **MIT 控制模式** 进行位置控制：

```
torque = kp * (p_cmd - p_actual) + kd * (v_cmd - v_actual)
```

控制参数（`position_kp` 和 `position_kd`）可在 xacro 文件中调整。

### 监控电机状态

```bash
# 查看关节状态
ros2 topic echo /joint_states

# 查看控制器状态
ros2 control list_controllers

# 查看硬件组件状态
ros2 control list_hardware_components
```

## 故障排查

### CAN 接口问题

#### SocketCAN 问题

**问题：** `Failed to open CAN interface can0`

**解决方法：**

1. 检查 CAN 接口是否存在：`ip link show can0`
2. 启动接口：`sudo ip link set up can0`
3. 验证波特率：`ip -details link show can0`
4. 检查 CAN 硬件连接

#### USB-CAN 问题

**问题：** `Failed to open serial port /dev/ttyUSB0`

**解决方法：**

1. 检查设备是否存在：`ls -l /dev/ttyUSB0`
2. 验证 USB 连接：`dmesg | tail`
3. 检查权限：`sudo usermod -a -G dialout $USER`（执行后注销重新登录）
4. 尝试更换 USB 端口
5. 检查 USB 线缆质量

**问题：** `Warning: Failed to enter AT mode`

**解决方法：**

1. 验证串口波特率是否正确（通常为 921600）
2. 尝试重新插拔 USB-CAN 适配器
3. 检查适配器是否处于正确模式
4. 部分适配器可能无需 AT 模式确认即可工作 — 检查 CAN 通信是否正常

**问题：** `Permission denied when accessing serial port`

**解决方法：**

1. 将用户添加到 dialout 组：`sudo usermod -a -G dialout $USER`
2. 注销后重新登录
3. 或使用 udev 规则（见 USB-CAN 配置章节）
4. 临时解决：`sudo chmod 666 /dev/ttyUSB0`

**问题：** `No feedback from motors`

**解决方法：**

1. 检查 CAN 接线和终端电阻（两端各接 120Ω）
2. 验证电机供电
3. 监听 CAN 总线：`candump can0`
4. 检查电机 ID 是否与配置一致
5. 确保电机已正确使能

**问题：** `Motors not responding to commands`

**解决方法：**

1. 验证电机是否已使能（检查错误码）
2. 检查电机 ID 配置
3. 确保 CAN 波特率匹配（1 Mbps）
4. 验证 MIT 控制参数是否合理

### 错误码

电机反馈中报告的错误码：

- `0x0`：已禁用
- `0x1`：已使能（正常运行）
- `0x8`：过压
- `0x9`：欠压
- `0xA`：过流
- `0xB`：MOS 过温
- `0xC`：线圈过温
- `0xD`：通信丢失
- `0xE`：过载

## 安全注意事项

1. **急停**：始终配备急停机制
2. **工作空间限制**：确保关节限位已正确配置
3. **温度监控**：运行过程中监控电机温度
4. **电源供应**：使用合适的电源（参考电机规格）
5. **初始测试**：初次测试时使用较低的速度和加速度

## 开发与调试

### 启用调试日志

```bash
ros2 launch moveit_mockway_config demo.launch.py --ros-args --log-level dmmotor_hardware_interface:=debug
```

### 测试单个电机

使用 Python 驱动进行测试：

```bash
cd ~/ws/mockway_robotics/tools/motor_gui
python3 dm_motor_driver.py
```

## 参考资料

- [ROS2 Control 文档](https://control.ros.org/)
- [SocketCAN 文档](https://www.kernel.org/doc/Documentation/networking/can.txt)
- DM 电机手册（参考厂商文档）

## 许可证

MIT License - Copyright (c) 2025 Mockway Robotics