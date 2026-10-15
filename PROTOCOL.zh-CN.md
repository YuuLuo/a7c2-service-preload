# 服务认证交换与指令执行边界

以下顺序对应 ILCE-7CM2 官方 2.01 的观察结果。认证帧结构可与 [Sony-PMCA-RE 的 Senser 实现](https://github.com/ma1co/Sony-PMCA-RE/blob/master/pmca/usb/sony.py)核对；其 [USB 服务入口](https://github.com/ma1co/Sony-PMCA-RE/blob/master/pmca/commands/usb.py)也展示了两次认证与中间的设备重新枚举。

## USB 会话和认证

主机占用接口 0 的 bulk IN/OUT 端点，先与海量存储身份 `054C:0E8A` 交互。第一次认证后，相机重新枚举为 `054C:0336`；主机打开这个新设备并再次认证，然后才使用服务指令。

| 阶段 | 交换内容 | 作用 |
| --- | --- | --- |
| 启动 | Vendor control OUT：`bmRequestType=0x43`、request `1`、value `0x37FF`、index `0xD7AA` | 启动 Senser。这台机身可能在控制传输状态阶段返回 stall，但仍继续提供认证。 |
| 挑战，重复三轮 | 主机发送逻辑命令 `1`；相机返回结果 `2` 或 `8` 和挑战数据 | `2` 选择与机身兼容的 SHA-1 分支，`8` 选择 SHA-256。 |
| 应答，每轮一次 | 主机发送逻辑命令 `3`、单字节标记 `1` 和算出的摘要；相机返回结果 `4` | 确认这一轮。 |
| 完成 | 主机发送逻辑命令 `5`；相机返回结果 `6` 和成功字节 `1` | 完成当前 USB 身份的认证。 |
| 结束 | Vendor control OUT：`bmRequestType=0x43`、request `1`、value `0xC800`、index `0x2855` | 结束 Senser 会话，启动返回普通 USB 模式的过程。 |

每个认证消息是 516 字节的大端帧：16 位取反的命令、16 位 salt、512 字节数据。回复结果按 `((~cmd) & 0xFFFF) - salt` 还原。摘要输入取决于设备当前身份：

- `0E8A`：取挑战数据的前四字节计算摘要。
- `0336`：将 512 字节挑战数据接上该轮对应的 SHA-1 或 SHA-256 [密钥表项](https://github.com/ma1co/Sony-PMCA-RE/blob/master/pmca/usb/constants.py)，再计算摘要。
- 结果为 `2` 时需要与 Senser 实现一致的 SHA-1 长度处理，不能直接换成普通 SHA-1 调用。[兼容函数](https://github.com/ma1co/Sony-PMCA-RE/blob/master/pmca/usb/crypto.py)见原始实现。

`0E8A` 认证完成后设备会重新枚举，所以 `0336` 必须用新设备句柄和新的序列状态再次完成交换。仅看到 Start 请求或新的 USB ID，不能说明认证已成功。第二次认证后，主机发送带 12 字节小端头的 Senser 数据包：32 位负载长度、16 位功能号、16 位序列号、四个 8 位字段。序列号从 `1` 开始，每个请求递增。功能号 `0x10` 用于 ProductInfo，研究中的 Jiritsu 请求使用 `0x40`。所观察会话的 ProductInfo 返回 88 字节。

## 文件操作如何变成指令执行

Jiritsu 在读写文件前先选择路径。在研究的固件上，含有 `/tmp/jiritsu/../..` 的选择路径会解析到预期 Jiritsu 目录之外。研究利用这一行为在临时 Work 区域放置共享对象，并影响动态加载器的预加载配置；随后通过服务文件接口，从 `/etc/ld.so.preload` 回读到了写入的对象路径。

触发点是一条 Jiritsu DEBUG 请求。它启动运行固定 shell 命令的 root 子进程。子进程的动态加载器读取 `/etc/ld.so.preload`，装入 Work 区域中的共享对象，并在固定命令前执行其初始化函数。工作对象写出带版本的结果记录并移除预加载条目。结果记录与预加载路径的单独回读共同证明执行和清理；仅有 USB 请求被接收并不足以证明代码已运行。

已观察到的是路径选择被接收、`/etc/ld.so.preload` 的回读内容、工作对象结果以及子进程上下文。所选 `/tmp` 路径与回读 `/etc` 路径之间的机内映射尚未唯一定位。确定的信任边界是：服务接口控制的文件内容进入了特权进程的动态加载决策。
