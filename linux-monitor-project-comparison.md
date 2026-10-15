# 两个 Linux 性能监控项目对比分析

## 1. 结论先行

综合看，`monitor_system` 的项目含金量更高，尤其适合投递 Linux C/C++、系统开发、后端基础设施、性能工程等岗位。它不仅采集本机指标，还覆盖了内核态采集、eBPF、跨主机 gRPC、Protobuf、MySQL 持久化、趋势/异常查询和多机评分，技术链路明显更长。

`qt-linux-monitor` 的优势是界面完整、容易现场演示，进程列表、进程控制、CPU/内存/网络曲线和文件系统表格都能直接被用户看到。它更适合体现 Qt 桌面开发和 GUI 数据展示能力，但从系统架构深度、分布式能力和内核技术来看，弱于 `monitor_system`。

这不是“功能多就一定更好”：`monitor_system` 的工程质量还没有达到生产级，存在安全、并发、数据库和测试方面的明显短板；`qt-linux-monitor` 也存在老式线程模型、跨线程访问 UI、缺少测试等问题。因此更准确的判断是：

- 技术上限与简历亮点：`monitor_system` 更高。
- 可视化完成度与现场演示：`qt-linux-monitor` 更好。
- 当前生产可用性：两者都还需要较多加固。

## 2. 项目定位

| 维度 | `monitor_system` | `qt-linux-monitor` |
|---|---|---|
| 产品形态 | 多服务器分布式监控后端 | 单机 Qt 桌面监控器 |
| 部署单元 | 每台被监控机部署 Worker，中心部署 Manager + MySQL | 被监控 Linux 主机直接运行 GUI |
| 核心语言 | C++17 + Linux 内核 C + eBPF C | C++17 + Qt5 |
| 主要依赖 | gRPC、Protobuf、MySQL、libbpf、libelf | Qt Widgets、procps、QCustomPlot、pthread |
| 数据来源 | 内核模块、eBPF、`/proc`、`/sys` | procps、`/proc`、`/sys`、`/etc/mtab`、`statvfs` |
| 指标范围 | CPU、负载、软中断、内存、网络、磁盘、主机身份 | CPU、负载、内存、Swap、网络、进程、文件系统/IO |
| 数据传输 | Worker 定时 Push 到 Manager | 无网络传输，进程内传递 |
| 持久化 | MySQL 5 张表，保存汇总和明细 | 无历史持久化，只有 QSettings 保存偏好 |
| 查询能力 | 9 个 gRPC 查询接口，含趋势、异常、排名和明细 | 无外部查询 API |
| 展示/操作 | 控制台输出，无正式前端 | Qt GUI、实时曲线、进程终止/强杀、属性查看 |
| 扩展性 | 可横向增加 Worker 和指标采集器 | 主要面向一台机器，扩展依赖修改 GUI |
| 自动化测试 | 4 个独立测试/验证程序，无系统化测试框架 | 未发现测试代码 |

代码规模只用于辅助理解，不能直接代表质量：排除生成的 `vmlinux.h` 后，`monitor_system` 的核心源码约 6819 行；排除内置 QCustomPlot 后，`qt-linux-monitor` 约 3397 行。

## 3. 共同点

1. 都是 Linux 专用的 C++17/CMake 项目，直接依赖 Linux 的 procfs、sysfs 或系统接口。
2. 都采用“周期采样 + 前后两次计数器做差”的思路计算 CPU、网络或磁盘速率。
3. 都覆盖 CPU、内存、网络和磁盘/文件系统这四类核心资源。
4. 都把耗时采集放到后台线程，避免主控制流一直阻塞。
5. 都对采集模块做了一定拆分，而不是把全部逻辑写在入口文件里。
6. 都缺少成体系的单元测试、集成测试、持续集成和性能基准。
7. 都存在配置、错误处理、线程安全和可运维性方面的工程化欠账。

## 4. `monitor_system` 架构与逻辑

架构图见 [`monitor-system-architecture.drawio`](./monitor-system-architecture.drawio)。

### 4.1 Worker 侧

1. `worker/main.cpp` 读取 Manager 地址和推送周期，创建 `MonitorPusher`。
2. `MonitorPusher` 建立不加密的 gRPC Channel，并启动独立推送线程。
3. 每次推送前，`MetricCollector` 顺序调用多个 `MonitorInter::UpdateOnce()`：
   - CPU 使用率：从 `/dev/cpu_stat_monitor` 映射内核模块共享数据，再对两次 jiffies 做差。
   - 软中断：从 `/dev/cpu_softirq_monitor` 映射每核计数器。
   - CPU Load：优先读取 `/dev/cpu_load_monitor`，失败时回退到 `/proc/loadavg`。
   - 内存：读取 `/proc/meminfo`。
   - 磁盘：读取 `/proc/diskstats`，计算吞吐、IOPS、时延和利用率。
   - 网络：依赖满足时使用挂载在 TC ingress/egress 的 eBPF 程序；否则改用 `/proc/net/dev`。
   - 主机身份：采集 hostname 和 IP。
4. 采集结果组装成 `MonitorInfo` Protobuf，通过 `GrpcManager.SetMonitorInfo` 每 N 秒推送到 Manager。

### 4.2 Manager 侧

1. Manager 在同一个 gRPC 端口注册数据接收服务和查询服务。
2. `GrpcServerImpl` 校验主机名、保存一份最新数据，并同步调用 `HostManager::OnDataReceived()`。
3. `HostManager` 生成 `hostname_ip` 标识，计算 CPU/内存/负载/磁盘/网络加权评分以及指标变化率。
4. 最新主机状态进入内存 Map；超过 60 秒未更新的主机会被后台线程移除。
5. 每次上报同步连接 MySQL，将汇总、网络、软中断、内存和磁盘明细写入 5 张表。
6. `QueryManager` 从 MySQL 查询历史数据，`QueryServiceImpl` 将其映射成 Protobuf 响应，对外提供性能、趋势、异常、评分排名、最新评分以及 4 类明细查询。

### 4.3 技术亮点

- 用户态、内核模块和 eBPF 三层技术都有实际代码，不只是概念描述。
- Worker/Manager/查询方职责明确，Protobuf 是稳定的跨进程数据契约。
- 既有实时最新状态，也有历史存储、趋势、异常和评分，链路比较完整。
- 网络监控支持 eBPF 与 procfs 降级，体现了依赖探测和兼容思路。
- 内核模块使用字符设备和 `mmap` 向用户态暴露采样结果，是较强的系统编程面试点。

### 4.4 主要短板

- Manager 和 Worker 使用明文、无认证的 gRPC；数据库账号密码写死在代码中。
- SQL 由字符串直接拼接，`server_name` 等外部字段没有转义，存在 SQL 注入和数据写入失败风险。
- 每次采样新建 MySQL 连接并逐条执行多条 INSERT，且发生在 gRPC 接收回调链路中；高并发时延迟和吞吐都会变差。
- 内存中的历史采样 Map 有全局静态状态，其中部分访问没有统一互斥保护；多路 gRPC 并发上报时存在数据竞争风险。
- `/proc/net/dev` 降级实现算出的是 KB/s，而 eBPF 路径和 Manager 按 B/s 理解，单位契约不一致。
- CPU 使用率和评分通常取 `cpu_stat(0)`，依赖第一个元素恰好是聚合 CPU；接口契约没有显式保证。
- CPU 内核模块缺失时没有 procfs 回退，监控会静默缺项；README 对回退能力的描述偏乐观。
- 缺少批量写入、连接池、重试/退避、消息缓冲、数据保留策略、监控告警和系统级自动化测试。

## 5. `qt-linux-monitor` 架构与逻辑

架构图见 [`qt-linux-monitor-architecture.drawio`](./qt-linux-monitor-architecture.drawio)。

### 5.1 启动与线程

1. `main.cpp` 创建 `QApplication` 和 `MainWindow`，进入 Qt 事件循环。
2. `MainWindow` 加载 `.ui`，创建 `QSettings`，再创建进程、资源、文件系统三个 Worker。
3. 三个 Worker 同时继承 `QObject` 和自定义 `workerThread`；后者为每个 Worker 启动一个 detached `std::thread`。
4. 切换 Tab 时，主窗口暂停不相关的 Worker，减少无意义刷新；资源线程可按设置保持运行。

### 5.2 三条数据链路

1. 进程链路：`libprocps -> processInformationWorker -> QTableWidget`。它枚举进程、计算进程 CPU/内存、支持搜索和“只看当前用户”，还能发送 `SIGTERM`/`SIGKILL`、打开进程属性窗口。
2. 资源链路：`/proc/stat + procps meminfo + /sys/class/net -> resourcesWorker -> Qt signals -> QCustomPlot/标签/进度条`。CPU 和网络保留最近 60 个采样点。
3. 文件系统链路：`/etc/mtab + statvfs + /sys/block/*/stat -> fileSystemWorker -> 文件系统表格`，展示容量、可用空间、使用率和 IO 利用率。
4. 偏好链路：`PreferencesDialogue -> QSettings`，保存刷新周期、颜色、内存口径、单位制和窗口位置等设置。

### 5.3 技术亮点

- 是一个用户可以直接使用和演示的桌面产品，展示闭环优于纯控制台项目。
- Qt Designer UI、信号槽、QSettings、QCustomPlot、排序表格、对话框等 GUI 能力覆盖较全。
- 不仅“看指标”，还支持进程搜索、属性查看、停止和强杀，交互链路更完整。
- CPU、网络和磁盘 IO 都采用差分采样，基本监控逻辑合理。

### 5.4 主要短板

- 自定义线程使用 detached `std::thread`、普通 `bool` 作为跨线程控制标志，并由线程自身删除线程对象，存在数据竞争和生命周期风险。
- Worker 的 `QObject` 线程归属没有移动到后台线程，但后台线程会访问 `QSettings`，部分路径还直接操作 `QPushButton`，违反 Qt GUI 只能在主线程访问的约束。
- 线程暂停采用 1ms 轮询，空转浪费 CPU；更适合使用 `QThread`、条件变量或定时器。
- 文件系统数据结构大量手工 `new/delete`，可用值类型或智能指针降低泄漏和异常安全风险。
- 只适合本机实时查看，没有历史数据、远程节点、外部 API、告警或持久化。
- 未发现自动化测试。源码头部统一标注 2017 年原作者版权，如果用于个人简历，需要明确本人完成或重构的部分，避免项目归属被质疑。

## 6. 关键区别

### 系统边界

`monitor_system` 解决“集中管理多台服务器”的问题，边界从 Linux 内核一直延伸到远程 RPC、数据库和查询客户端。`qt-linux-monitor` 解决“人在本机查看和管理进程”的问题，边界集中在单个桌面进程内。

### 数据流方向

`monitor_system` 是定时 Push：Worker 主动采集并上报，Manager 被动接收。`qt-linux-monitor` 是 GUI 驱动的本地 Poll：线程周期读取本机数据，再通过信号槽刷新界面。

### 状态与历史

`monitor_system` 同时维护内存最新值和 MySQL 历史值，可做趋势、异常与排名。`qt-linux-monitor` 只保存最多 60 个曲线点，程序退出后监控历史消失。

### 操作对象

`monitor_system` 面向机器/集群，支持评分和调度决策；`qt-linux-monitor` 面向本机用户和进程，支持查看、停止和强杀进程。

### 技术难点

`monitor_system` 的难点是内核/用户态边界、eBPF、序列化、分布式通信、时序数据和数据库查询。`qt-linux-monitor` 的难点是 GUI 响应性、线程与事件循环、实时绘图、表格交互和进程控制。

## 7. 含金量判断

按简历项目常见评价维度做一个非绝对的参考评分：

| 维度 | 权重 | `monitor_system` | `qt-linux-monitor` | 判断依据 |
|---|---:|---:|---:|---|
| 技术深度 | 30% | 9.0 | 6.5 | 内核模块/eBPF/gRPC/MySQL 对比 Qt/系统接口 |
| 架构复杂度 | 25% | 8.5 | 5.5 | 多进程多节点、协议和存储层对比单进程三线程 |
| 功能闭环 | 20% | 7.5 | 7.5 | 前者有采集到查询，后者有采集到 GUI 与进程操作 |
| 工程质量 | 15% | 5.5 | 5.0 | 两者都有线程、安全、测试与错误处理欠账 |
| 演示效果 | 10% | 5.5 | 8.5 | Qt GUI 更直观，分布式项目目前主要是控制台 |
| 加权参考分 | 100% | **7.8** | **6.3** | 只反映当前代码快照 |

最终建议：将 `monitor_system` 作为主项目，`qt-linux-monitor` 作为补充项目。前者重点讲“为什么采用 Push、内核态与 procfs/eBPF 的取舍、指标差分、数据协议、评分和查询”；后者重点讲“Qt 信号槽、实时绘图、进程管理和线程模型”。如果只能保留一个，保留 `monitor_system`。

## 8. 提升项目可信度的优先级

### `monitor_system`

1. 用 TLS/mTLS 和环境变量/配置文件替代明文 gRPC 与硬编码凭据。
2. 用预处理语句、连接池、批量异步写入替代动态 SQL 和每次新建连接。
3. 明确所有指标单位和聚合 CPU 契约，修复 procfs 网络 KB/s 与 B/s 不一致。
4. 增加 Worker/Manager Docker Compose 或一键演示脚本，并补一个简单 Web/Grafana 展示层。
5. 增加采集器单测、Proto 兼容测试、Manager 集成测试和多 Worker 压测。

### `qt-linux-monitor`

1. 用标准 `QThread` Worker Object 模式或主线程 `QTimer` 重构 detached 线程。
2. 保证所有 QWidget 更新都回到 GUI 线程，跨线程状态使用原子量或条件变量。
3. 用 RAII/value type 重构文件系统对象的手工内存管理。
4. 增加采集解析单测，并用可注入文本代替测试时直接读取真实 `/proc`。
5. 清楚标注原项目来源和本人新增/重构范围。

## 9. 分析范围说明

本报告基于当前目录中的源码、CMake、Proto、SQL 和 README 做静态审阅。当前机器是 macOS，两个项目都依赖 Linux 专用接口，因此没有做真实 Linux 内核模块加载、eBPF attach、MySQL/gRPC 联调或 Qt 运行验证；架构图和结论描述的是代码当前实现，而不是生产环境实测结果。
