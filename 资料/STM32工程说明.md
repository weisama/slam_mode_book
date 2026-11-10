# STM32 v3.0 通用 Keil 示例

2D、3D 共用同一份应用和协议代码，正式工程只有两份：

- HAL：`HAL/MDK-ARM/stm32f103c8t6.uvprojx`，入口 `HAL/Core/Src/main.c`。
- 标准库：`标准库/Project.uvprojx`，入口 `标准库/User/main.c`。

工程依赖同级 `app/` 与 `protocol/`，移动时保留整个 `5_mode_v3.0`。标准库引用 `../app`、`../protocol`，HAL 从 MDK-ARM 引用 `../../app`、`../../protocol`。不再设置 `LIDAR_IS_2D`，开机不自动读参数或等待 `0x99`。原 2.0 示例不受修改。

目标为 STM32F103C8，USART1：PA9 TX 接模块 RX、PA10 RX 接模块 TX，双方共地，115200、8N1。使用适合板卡和模块的独立稳定电源，并确认双方串口电平兼容（STM32 为 3.3 V）；先断电接线。工程沿用 2.0 板卡时钟设置，HAL 使用 8 MHz HSE、72 MHz 系统时钟。通信帧没有版本号，两端协议必须匹配。

## 入口和使用

各工程的 `main.c` 中 `SELECTED_EXAMPLE` 选择 `EXAMPLE_LOCALIZATION`（定位接收）、`EXAMPLE_CAR`（小车手动发点）或 `EXAMPLE_DRONE`（无人机航点）。两个正式工程均默认 `EXAMPLE_CAR`。公共入口为 `example_start(SELECTED_EXAMPLE)`，不接收雷达类型。前台持续接收，可在调试器观察 `app_pose`、`app_position_command`、`app_velocity`、`app_goal_echo` 和 `app_status`。数值分别使用 cm、度、cm/s 和度/s；坐标系为 `base_init_map`。状态资源值 FF 表示未知，不是 255%；资源字段仍待协议审核。串口错误或接收队列溢出会丢弃缓存并重新同步，`app_rx_overflows` 记录次数。所有应用接口须由主循环上下文调用。

小车不依赖飞控状态。用户在主循环的明确操作入口调用 `app_send_goal(goal, now_ms)`；`goal` 是六个 int16 值（X、Y、Z、roll、pitch、yaw）。返回 1 表示本机已发送，0 表示发送受限或无人机例程活动。调用间隔至少 1000 ms。模块的目标回显仅确认收到帧，不代表规划接受或到达，且模块对不同目标有 1 秒发布限制。

需要无人机例程时，先在 `app/example.c` 配置持续有效的航点，再明确改为 `EXAMPLE_DRONE` 后重新编译。当前航点为 `(4,0,1)`、`(4,-2,1)`、`(0,0,1)` 米，源码使用厘米。通用示例调用 `app_start_drone(..., 0)`，不强制人工高度确认；底层 `app_start_drone(..., 1)` 与 `app_confirm_height()` 仍可供需要显式确认的应用使用，独立于 2D/3D。

自动航点仍要求新鲜位姿、飞控状态 3/4，以及 Z > 50 cm 持续 1 秒。位姿接收时效不能证明其内部高度源仍在更新：2D 的 Z 来自飞控，模块可能重复发送缓存的旧 Z。统一不增加对此情况的自动识别。模块当前仍把 2D 目标 Z 替换为起飞高度，规划模式仍仅支持 0；3D 使用航点指定的 Z。本次没有修改模块目标或规划层逻辑，没有实现新的 2D 当前高度保持功能。无人机暂停、重试规则见 [协议说明](串口开发说明.md)。解锁、起飞、降落由模块、飞控和外部操作者负责。实机飞行需另行验收。

命令与参数通过 `app_commands` 手动调用，例如主循环中的用户操作可执行 `command_example_start(&app_commands, 0x03, now_ms, 2000)` 读取参数；通过 `command_example_poll` 和 `app_commands.request` 查看结果。开机不自动读参数，读参数失败不是通用航点示例的启动门槛。存图、写参数、保存参数、重启均须由用户明确调用，启动不会发送这些操作。有关超时、应答含义及示例见 [协议说明](串口开发说明.md)。

## 模块配置和排查

使用与本目录协议匹配的模块版本，在模块上启用主控串口（参数 `0x12=1`）；导航还需配置地图、定位及导航使能 `0x14`。飞控使用者自行完成模块与飞控的连接、飞控位置和定点配置，并验证高度来源及控制链。2D 导航的规划模式 `0x25` 必须为 0。安装位姿、起飞高度 `0x23`、规划及运动参数应依现场设置；查阅[当前参数表](通讯协议.md#参数id定义表)，更改和保存由人手动触发。

本机正式工程的本地调试选项已配置为 ST-Link / SWD，两个工程均生成 HEX。个人 `.uvoptx` 不纳入版本控制；新电脑首次打开时，在 Alt+F7 中确认 Debug 为 ST-Link Debugger、Settings 的 Port 为 SWD，Utilities 勾选 Use Debug Driver。

常见问题：没有数据时检查供电、共地、交叉接线、波特率、串口使能和协议版本；`app_rx_overflows` 增长时检查主循环是否及时处理接收。状态 1 Hz，无人机默认 2 秒超时；位姿默认 500 ms 超时，数据不新鲜或飞控状态不是 3/4 时暂停。目标回显等待 300 ms，最多发送 5 次，超时进入失败状态；确认链路后由操作者处理。命令超时只表示未收到完整响应，存图无回显可能是失败、拒绝或通信丢失；重启回显只表示收到请求。实际时序以设备测量为准。


## 下载和打开

下载[例程压缩包](../mode_v3.0_stm32f103c8t6.zip)，完整解压到可写目录，不要直接在压缩包内打开工程。保留5_mode_v3.0下的app、protocol、HAL和标准库目录。包内包含两种库的工程、共用源码及测试源码，不含个人IDE设置、编译产物或现场测试副本。

使用Keil µVision 5、ARM Compiler 5及STM32F1xx器件支持包。新电脑首次打开时，在Alt+F7中确认Debug为ST-Link Debugger、Settings的Port为SWD，Utilities勾选Use Debug Driver。编译、烧录后再进入debug观察接收变量。工程构建不等于已完成本机型的串口联调或飞行验收。

[通信协议](通讯协议.md) · [串口开发说明](串口开发说明.md)
