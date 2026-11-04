# Argus（阿尔戈斯）— 分布式工业视觉与设备数据采集平台

> 一个 Windows 客户端 + Linux 服务端的 C++ 全栈练手项目
> 目标：把 **多线程 / 线程池 / 内存池 / Qt / SQLite / OpenCV+FFmpeg / OpenGL / Modbus TCP** 拧成一条真实业务链，而不是八个孤立 Demo。

---

## 0. 一句话概述

**一条完全由软件仿真的"虚拟产线"**：虚拟 PLC（Modbus TCP 从站）驱动 4 台设备，虚拟摄像头输出 4 路视频流；Linux 服务端负责采集、分析、存储、分发；Windows 客户端用 Qt + OpenGL 做 3D 数字孪生、视频墙、实时曲线和告警控制。

最有意思的一条闭环：

```
虚拟摄像头出现"缺陷工件"
   → 服务端 OpenCV 检测到
   → 写 Modbus 线圈 00004（急停）
   → 虚拟 PLC 状态字变"故障"，产线停转
   → 客户端 3D 模型停下 + 弹红色告警 + 落库 + 存快照
```

这条链路走通，说明视频、协议、控制、3D、存储、UI 六件事真的被你自己串起来了。

---

## 1. 为什么选这个题目

| 你的诉求 | 本项目怎么满足 |
|---|---|
| C++ 多线程 | 服务端 8+ 个常驻线程：采集调度、Modbus 会话、解码、CV 分析、落库、网关推送、定时器 |
| 线程池 | 采集任务 + CV 任务 + 存储任务全部投递到自研线程池（带优先级/定时/优雅停机） |
| 内存池 | 视频帧、Modbus 报文、协议包三类高频对象走自研分级内存池（含 tcache + 引用计数帧） |
| Qt 界面 | 客户端 7 个功能页，全部 Qt Widgets + Model/View + 自定义控件 |
| SQLite | 服务端存点位/样本/告警/快照索引；客户端存离线缓存与 UI 配置（两端都用） |
| OpenCV + FFmpeg | FFmpeg 拉流解封装解码 → OpenCV 背景差分/连通域/计数/缺陷分类 → OSD 叠加 → 快照 |
| OpenGL 3D | QOpenGLWidget 手写渲染管线：VBO/VAO、MVP、光照、实例化、YUV→RGB shader、拾取 |
| Modbus TCP | **自研**从站模拟器（虚拟 PLC）+ 自研主站，不用 libmodbus 打底，只用它交叉验证 |
| 没有硬件 | 设备 = 自研 Modbus 从站 + 物理仿真内核；摄像头 = OpenCV 合成画面 / MediaMTX 推 RTSP |

**关键点**：因为设备和摄像头都是你自己仿真的，所以你拥有 **ground truth**——CV 计数准不准、采集延迟多大、3D 动画和数据是否同步，全部可以自动化量化验收。这是买不到硬件的人反而独有的优势。

---

## 2. 系统总体架构

```
┌──────────────────── Windows 11 (开发机) ────────────────────┐
│                                                             │
│  Argus Client (Qt 5.14 / MSVC)                              │
│  ├─ UI: 总览看板 / 3D 孪生 / 视频墙 / 点表 / 告警 / 回放     │
│  ├─ NetClient: 帧编解码、心跳、重连、订阅                    │
│  ├─ DecodeThread: FFmpeg 解码 → FrameQueue(丢旧帧)          │
│  ├─ GL Renderer: QOpenGLWidget, YUV shader, 3D scene        │
│  └─ LocalCache: SQLite 离线缓存 / 用户配置                   │
│                          │  自定义 TCP 协议 (帧: MAGIC+LEN+CRC) │
└──────────────────────────┼──────────────────────────────────┘
                           │  局域网 / WSL 端口转发
┌──────────────────────────┼──────────────────────────────────┐
│  Argus Server (Linux / WSL2 Ubuntu)                         │
│                          ▼                                   │
│  Gateway   : epoll TcpServer + Session + 主题订阅 + 推送      │
│  Acquisition: ModbusMaster + PollScheduler + TagTable(点表)   │
│  Media     : RtspIngest(FFmpeg) + CvPipeline(OpenCV) + FrameBus│
│  Storage   : SQLite(WAL/批量事务) + 保留策略 + CSV 导出        │
│  Core      : ThreadPool / MemoryPool / Logger / Config        │
│                                                             │
│  ── 仿真域（可独立进程，也可内嵌）──                          │
│  Simulator : ModbusSlaveSim ×4 (虚拟 PLC, :5020~:5023)       │
│              SynthCam ×4 (OpenCV 合成画面 / 推 RTSP)          │
│              FaultInjector (过温/轴承磨损/断链/网络抖动)       │
└─────────────────────────────────────────────────────────────┘
```

**进程建议**：M2 阶段把 Simulator 做成独立可执行文件 `argus-sim`。好处：
1. 服务端崩溃不影响仿真，反之亦然；
2. 可以用 Modbus Poll / QModMaster / pyModbus 等第三方工具连上去交叉验证你的从站实现是否正确；
3. 练习进程间协作、多实例部署、端口规划。

---

## 3. 技术选型

| 领域 | 选型 | 理由 |
|---|---|---|
| 语言标准 | C++17 | VS2017 v141 / GCC 9+ 都完整支持，别一上来 C++20 |
| 构建 | CMake + CMakePresets + vcpkg manifest | 一套 CMakeLists 双端构建，逼你写跨平台代码 |
| Win 编译器 | MSVC v143（VS2022 Build Tools）| ⚠️ 你现有 VS2017 v141 可用，但**新版 vcpkg 只支持 VS2019/2022**，见环境文档 |
| Linux 编译器 | GCC 11/13（WSL2 Ubuntu 24.04）| |
| 公共库 | `common/` 静态库（协议、线程池、内存池、日志、工具）| Windows/Linux 双端编译，是跨平台能力最好的训练场 |
| GUI | Qt 5.14.2 Widgets + QtCharts | 你已装好；MSVC 2017/2019/2022 ABI 互通 |
| 3D | 手写 OpenGL 3.3 Core（QOpenGLWidget）| 比 Qt3D 学得扎实得多；Qt3D 可做对照实现 |
| 视频 | FFmpeg（libavformat/libavcodec/libswscale）| 自己写 demux→decode→convert，不用 ffmpeg.exe 管道 |
| 图像 | OpenCV 4.x | 只当"算子库"用，不用它的 VideoCapture 兜底 |
| 数据库 | SQLite3（直接用 C API，不用 ORM）| 手写 SQL + 预编译语句 + WAL 调优，收获最大 |
| Modbus | **自研** Master + Slave；libmodbus 仅用于对拍 | 协议不复杂，自己写才能吃透 MBAP/功能码/异常码/字节序 |
| 日志 | spdlog（异步 + 滚动）| |
| 序列化 | 控制面 nlohmann/json 起步 → M7 换 Protobuf | 体验一次"协议演进式重构" |
| 测试 | Catch2 + Google Benchmark | 每个模块先写测试 |
| 静态检查 | clang-tidy + ASAN/TSAN/UBSAN | 多线程项目不上 TSAN 等于裸奔 |
| 抓包 | Wireshark（内置 Modbus 解析器）| 调协议必备 |
| 流媒体 | MediaMTX（原 rtsp-simple-server）| 把 mp4 循环推成 RTSP，模拟真实 IP 摄像头 |

---

## 4. 服务端模块设计

### 4.1 Core — 线程池

```cpp
class ThreadPool {
public:
    struct Options { size_t workers; size_t queueCapacity; bool steal; };
    explicit ThreadPool(Options);
    ~ThreadPool();                                  // 自动 shutdown

    template<class F> auto submit(F&& f) -> std::future<std::invoke_result_t<F>>;
    template<class F> auto submit(Priority p, F&& f) -> std::future<...>;
    void submitAfter(std::chrono::milliseconds d, std::function<void()>);   // 定时
    void shutdown(bool drainPending);               // 优雅停机
    Stats stats() const;                            // 队列深度/执行时长直方图/拒绝数
};
```

**必做（基础版）**：mutex + condition_variable + `std::deque<std::function<void()>>` + `std::packaged_task` 返回 future。
**进阶（拿分版）**：
- 多优先级队列（HIGH=控制指令, NORMAL=采集, LOW=落库/归档）
- 定时任务：最小堆 + 单一 timer 线程投递到池
- 无锁 MPMC 队列（环形数组 + 原子 head/tail），对比 mutex 版吞吐
- work-stealing：每 worker 一个本地双端队列
- 队列满时的背压策略（阻塞 / 丢弃最旧 / 拒绝并计数）
- 线程亲和性绑定 + 命名（`pthread_setname_np` / `SetThreadDescription`），方便 top/Process Explorer 里看

**验收**：`argus-bench` 对比 ①裸 std::thread ②std::async ③你的线程池，在 100 万个 1μs 空任务下的耗时与 CPU 占用；TSAN 干净。

### 4.2 Core — 内存池

三类对象特征完全不同，**分别设计**才有针对性：

| 对象 | 大小 | 频率 | 策略 |
|---|---|---|---|
| Modbus 报文/协议帧 | 16B~2KB | 极高 | 分级 size-class free-list（64/128/256/512/1K/2K） |
| 视频帧 1080p YUV420 | ~3MB | 25fps×4路 = 100/s | 大块 arena + 引用计数 `FrameRef`，环形复用固定 N 个槽 |
| 采样点结构体 | 24B | 800/s | slab 定长池 + 批量释放 |

```cpp
template<size_t Size, size_t Align = 64>
class SlabPool {                     // 定长块，侵入式 free-list
public:
    void* allocate();
    void  deallocate(void* p);
    // 每线程 tcache：thread_local std::vector<void*>，满 32 个批量归还中央池
};

class FrameRef {                     // 零拷贝帧句柄
public:
    uint8_t* data();  size_t size();
    int64_t  ptsUs(); int width(), height(); AVPixelFormat fmt();
    FrameRef(FrameRef&&) noexcept;   // 移动语义 + 原子引用计数
};
```

**必须踩的坑（也就是必须学到的东西）**：
- 对齐：`alignas(64)` 缓存行对齐避免 false sharing；帧数据 32/64B 对齐利于 SIMD 与 `glTexImage2D` 的 `GL_UNPACK_ALIGNMENT`
- 侵入式链表：块的前 8 字节存 next 指针，零额外开销
- 内存增长：chunk 按需向系统申请（`::operator new` / `VirtualAlloc` / `mmap`），一次 1~64MB
- 调试模式：red-zone 越界检测、`0xDEADBEEF` 填充、未释放块泄漏报告、水位/峰值统计
- RAII：`PoolPtr<T>` 自定义 deleter；严禁裸 new/delete 出现在业务代码里

**验收**：1000 万次 alloc/free 对比系统 malloc 的耗时（目标 ≥3 倍）；ASAN 无越界；长跑 24h RSS 不增长。

### 4.3 Acquisition — Modbus 主站 + 点表

自研 Modbus TCP 客户端要处理的东西（每一条都是真实工程问题）：
- MBAP 头（TransactionId / ProtocolId=0 / Length / UnitId）+ PDU
- 功能码 0x01/0x02/0x03/0x04 读，0x05/0x06/0x0F/0x10 写
- 异常响应（0x80|FC）与异常码 01~0B 的语义
- 大端字节序、32 位量跨寄存器拼接（ABCD/CDAB 字序可配）
- **请求合并优化**：把相邻地址的点位合并成一次批量读（这是工业采集性能的关键，一定要做）
- 事务号管理 + 响应超时 + 重传 + 连接断开重建
- 连接池：N 台设备共享 M 条 TCP 连接 or 每设备一条
- 通信质量戳（Good/Bad/Uncertain），断链时点位标 Bad 而不是保留旧值

**点表（TagTable）**——工业软件的灵魂，别偷懒：
```
tag: { device, name, fc, addr, count, dtype(u16/i16/u32/f32/bit),
       scale, offset, unit, deadband, alarm_lo, alarm_hi, poll_ms }
```
- `scale/offset`：原始寄存器值 → 工程量（如 40006 = 温度×10）
- `deadband`：死区上报，变化不足阈值不推送 → 大幅降低网络与存储压力
- 支持从 CSV/Excel 导入点表（做一个 `argus-tag-import` 小工具）

### 4.4 Media — FFmpeg + OpenCV

```
RtspIngest ──avformat_open_input──► demux ──► AVPacket 队列
                                                │
                              avcodec_send/receive ──► AVFrame(YUV420P)
                                                │  从 MemoryPool 取槽，包成 FrameRef
                                                ▼
                                        CvPipeline (线程池)
                             背景差分 → 形态学 → 连通域 → 计数/分类
                             → OSD 叠加(画框/文字/帧号) → 快照 JPEG
                                                ▼
                                        FrameBus (环形 3 槽, 满丢旧帧)
                                                ▼
                                   Gateway 推给客户端 / Storage 存快照
```

要点：
- **时间戳同步**：视频 PTS 与 Modbus 采样时间戳统一到服务端单调时钟，回放时才能对齐"当时转速多少、画面里几个工件"
- **丢帧策略**：分析跟不上时丢旧帧保新帧（实时性 > 完整性），并统计丢帧率
- **零拷贝**：解码输出的 AVFrame 用 `av_buffer_ref` 挂到你的内存池，避免 memcpy 3MB/帧
- **算子插件化**：`ICvOperator` 接口 + 配置驱动流水线，为 V2 的动态库插件铺路
- **合成摄像头（无硬件的关键）**：
  - SynthCam 用 OpenCV 画俯视传送带：底色 + 匀速运动的工件（颜色/形状随机），按概率生成"缺陷件"
  - **传送带速度取自 Modbus 转速寄存器** → 视频与设备数据同源，可验证时序一致性
  - 输出三选一：写 mp4 文件 / 推给 MediaMTX 变 RTSP / 直接走共享内存给 Media 模块（零拷贝，用于测试）
  - SynthCam 同时输出 ground-truth JSON（每个工件的位置/是否缺陷），CV 模块可自动算准确率

### 4.5 Storage — SQLite

```sql
PRAGMA journal_mode=WAL;  PRAGMA synchronous=NORMAL;  PRAGMA cache_size=-64000;

CREATE TABLE device(id INTEGER PRIMARY KEY, code TEXT UNIQUE, name TEXT, type TEXT,
                    host TEXT, port INT, unit_id INT, enabled INT);
CREATE TABLE tag  (id INTEGER PRIMARY KEY, device_id INT, name TEXT, fc INT, addr INT,
                   count INT, dtype TEXT, scale REAL, offset REAL, unit TEXT,
                   deadband REAL, alarm_hi REAL, alarm_lo REAL, poll_ms INT);
CREATE TABLE sample(ts_ms INT NOT NULL, tag_id INT NOT NULL, value REAL, quality INT,
                    PRIMARY KEY(ts_ms, tag_id)) WITHOUT ROWID;   -- 按天分表 sample_20261103
CREATE TABLE alarm (id INTEGER PRIMARY KEY AUTOINCREMENT, ts_ms INT, src TEXT, level INT,
                    code TEXT, text TEXT, ack_ts_ms INT, ack_user TEXT);
CREATE TABLE snapshot(id INTEGER PRIMARY KEY AUTOINCREMENT, ts_ms INT, cam_id TEXT,
                    path TEXT, kind TEXT, meta TEXT);
CREATE TABLE meta(k TEXT PRIMARY KEY, v TEXT);
```

必须练到的点：
- 预编译语句 + `sqlite3_bind_*`，**一次事务批量提交 1000~5000 行**（单行 autocommit 会慢 100 倍，亲手测出来印象最深）
- WAL 模式下的读写并发；一个写线程 + 多个读连接
- 索引设计与查询计划（`EXPLAIN QUERY PLAN`）
- 按天分表 + 保留策略滚动删除 + VACUUM 时机
- schema 版本迁移（`meta` 表存 version，启动时增量迁移）
- 导出 CSV / Parquet

**验收**：写入 1000 万条 sample，查询任意 1 小时区间聚合 < 200ms；插入吞吐 ≥ 5 万行/秒。

### 4.6 Gateway — 网络协议

自定义二进制帧（客户端也复用同一份 `common` 代码）：

```
 0        2    3     4      5         9         13              13+LEN
 +--------+----+-----+------+---------+---------+----------------+-------+
 | MAGIC  |VER |TYPE |FLAGS |  SEQ    |  LEN    |    PAYLOAD     | CRC32 |
 | 0xA5 0x5A |1B| 1B | 1B  |   4B    |   4B    |   LEN bytes    |  4B   |
 +--------+----+-----+------+---------+---------+----------------+-------+
FLAGS: bit0=payload压缩(lz4) bit1=分片 bit2=需要ACK
```

- 消息类型：HELLO / AUTH / HEARTBEAT / SUBSCRIBE / UNSUBSCRIBE / TELEMETRY / ALARM / VIDEO_FRAME / SNAPSHOT / COMMAND / COMMAND_ACK / HISTORY_REQ / HISTORY_RSP / ERROR
- 服务端用 **epoll ET 模式 + 每连接读写缓冲**，手写粘包/半包处理（这是面试高频考点，务必亲手写）
- 会话与订阅：客户端订阅主题（`dev/1/tag/temp`、`alarm/#`、`cam/0/frame`），服务端按订阅推送
- 背压：客户端消费慢时，视频帧丢弃、遥测数据聚合降频，绝不能把发送缓冲撑爆
- 心跳 5s，3 次未响应断开；指数退避重连
- Windows 侧 `common` 用 `#ifdef` 抽象出 epoll/IOCP 两套实现 → 顺便学 IOCP

---

## 5. 客户端模块设计（Qt）

### 5.1 界面布局

```
┌─ 顶栏: 连接状态 / 服务器 / 用户 / 告警计数 / 全局急停 ─────────────┐
├─ 左侧导航 ─┬─ ① 总览看板: 设备状态卡片 + KPI + 实时曲线(QtCharts)   │
│  总览       │─ ② 3D 数字孪生: 产线三维场景 + 相机漫游 + 点选设备      │
│  3D 孪生    │─ ③ 视频墙: 1/4/9 分屏, OSD, 抓图, 单路放大             │
│  视频墙     │─ ④ 点表浏览器: 树形设备→点位, 实时值, 手动写值/强制     │
│  点表       │─ ⑤ 告警中心: 实时告警滚动 + 历史查询 + 确认/派工       │
│  告警       │─ ⑥ 历史回放: 时间轴, 数据+视频+告警三轨同步            │
│  回放       │─ ⑦ 系统配置: 服务器/订阅/UI 主题/日志级别              │
└─ 底部状态栏: 延迟 / 丢帧率 / 内存池水位 / 线程池队列深度 ────────────┘
```

### 5.2 线程模型（Qt 最容易踩坑的地方）

```
UI 线程 ─── Qt event loop, 只做绘制与交互, 任何阻塞都不可接受
   ▲  queued signal/slot
NetThread ─ QTcpSocket 收帧 → 解码 → 发射信号（严禁在此碰 QWidget）
DecodeThread×4 ─ FFmpeg 解码 → FrameQueue(容量3, 满丢旧) → 发 FrameReady
TimerThread ─ 曲线重采样、状态栏刷新（20Hz 限流，别用 60Hz 刷 UI）
```

规矩：
- 跨线程只传**值类型或引用计数智能指针**，绝不传裸指针
- `connect(..., Qt::QueuedConnection)` 显式声明，别依赖 auto
- 高频数据（100Hz 遥测）在 NetThread 侧**先聚合再发**，UI 20Hz 取最新值，否则界面卡死
- 自定义类型要 `qRegisterMetaType`

### 5.3 OpenGL 3D 数字孪生

技术路线：**QOpenGLWidget + QOpenGLFunctions_3_3_Core + 手写 shader**（不用 Qt3D，学得深）

场景内容：
- 地面网格 + 坐标轴 + 天空盒（可选）
- 传送带：滚动 UV 偏移，偏移速度 = 转速寄存器 → 视觉上"真的在动"
- 3~4 台设备模型：底座 / 机身 / 旋转主轴 / 状态指示灯（emissive 颜色随状态字变化）
- 工件：Instancing 批量渲染几百个，位置由服务端 CV 结果或仿真真值驱动
- 摄像头视锥线框 + 告警光柱（半透明圆柱 + additive blending）
- 故障时：设备变红 + bloom 后处理闪烁

渲染管线要点：
- VBO/VAO/EBO、MVP 矩阵、法线矩阵、Phong 或简化 PBR
- 阴影（可选 CSM）、后处理 bloom、MSAA
- **拾取**：离屏 FBO 用"ID 编码成颜色"渲染一遍（GPU picking），或 CPU 射线-包围盒求交
- **轨道相机**：鼠标左键旋转 / 右键平移 / 滚轮缩放，四元数避免万向锁
- 模型加载：先手写几何体，再上 tinygltf 导入 glTF/OBJ
- 插值：网络来的状态用时间戳做 lerp/slerp，避免动画跳变（很关键，不做会一顿一顿）

视频渲染（性能关键点）：
```
YUV420P → 上传 3 张纹理 (Y: GL_RED, U/V: GL_RED 各 1/2 分辨率)
       → fragment shader 做 BT.601/709 矩阵转 RGB
```
对比"CPU 用 sws_scale 转 RGB 再上传"，GPU 方案能省掉 80% 以上的 CPU。**这个优化前后对比数据可以直接写进简历。**

### 5.4 客户端 SQLite
- 离线缓存：断线期间的告警/曲线写入本地库，重连后比对
- UI 配置、点表缓存、窗口布局持久化

---

## 6. 仓库结构

```
argus/
├─ CMakeLists.txt              # 顶层：options、依赖、子目录
├─ CMakePresets.json           # win-msvc / linux-gcc / asan / tsan / release
├─ vcpkg.json                  # manifest 模式依赖清单
├─ .clang-format / .clang-tidy / .editorconfig
├─ cmake/
│   ├─ FindFFmpeg.cmake
│   └─ CompilerWarnings.cmake
├─ docs/
│   ├─ architecture.md  protocol.md  db-schema.md  adr/
├─ common/                     # ★ Windows/Linux 双端编译的共享库
│   ├─ include/argus/common/
│   │   ├─ ThreadPool.h  MemoryPool.h  FrameRef.h
│   │   ├─ Protocol.h    Codec.h       RingBuffer.h
│   │   ├─ Logger.h      Config.h      TimeUtil.h  Result.h
│   └─ src/
├─ server/                     # Linux 服务端可执行文件
│   ├─ src/
│   │   ├─ core/       # 生命周期、指标、信号处理
│   │   ├─ modbus/     # Master、连接池、报文编解码、请求合并
│   │   ├─ acquire/    # PollScheduler、TagTable、死区、质量戳
│   │   ├─ media/      # RtspIngest、CvPipeline、算子、FrameBus
│   │   ├─ storage/    # SqliteDb、批量写、保留策略、迁移
│   │   ├─ gateway/    # EpollServer、Session、订阅、推送、命令路由
│   │   └─ main.cpp
│   └─ tests/
├─ simulator/                  # ★ 虚拟硬件层（独立可执行 argus-sim）
│   ├─ modbus_slave/           # 自研 Modbus TCP 从站（多实例、多连接）
│   ├─ plant_model/            # 温度/转速/振动/产量的物理仿真 + PID
│   ├─ synth_cam/              # OpenCV 合成摄像头画面 + ground truth
│   ├─ fault_inject/           # 故障注入脚本/命令
│   └─ main.cpp
├─ client/                     # Windows Qt 客户端
│   ├─ src/
│   │   ├─ app/        # main、Application、单实例、崩溃捕获
│   │   ├─ net/        # NetClient、帧编解码、重连、订阅管理
│   │   ├─ model/      # TagModel、AlarmModel、DeviceModel (QAbstractTableModel)
│   │   ├─ media/      # DecodeThread、FrameQueue、AudioOut
│   │   ├─ gl/         # GlWidget、Scene、Camera、Mesh、Shader、Picker
│   │   ├─ ui/         # 7 个页面 + 自定义控件
│   │   └─ cache/      # 本地 SQLite
│   ├─ resources/      # qrc: 图标、shader、模型、qss 样式
│   └─ tests/
├─ tools/
│   ├─ argus-bench/            # 压测与基准（线程池/内存池/网络/存储）
│   ├─ argus-tag-import/       # CSV → SQLite 点表
│   └─ modbus-cli/             # 命令行读写 Modbus，调试用
├─ third_party/                # 不便 vcpkg 化的源码（如 tinygltf、单头文件库）
├─ scripts/                    # build.ps1 / build.sh / deploy / gen-testdata
└─ deploy/
    ├─ argus-server.service    # systemd
    ├─ Dockerfile
    └─ windeployqt.ps1
```

**工程纪律**（比功能更能体现水平）：
- `common/` 里禁止出现 `windows.h` / POSIX 头文件，平台相关代码全部收到 `common/src/platform/` 下的两个实现文件
- 全项目 `/W4 -Wall -Wextra -Werror` 起步
- 每个模块配一个 `tests/` 与一个 `README.md`
- 关键决策写 ADR（Architecture Decision Record），例如"为什么自研 Modbus 而不用 libmodbus"
- Git：`main` 保持可编译可测试，功能走 feature 分支，PR 自审 diff
- Commit 规范：`feat(media): 实现背景差分算子` / `fix(modbus): 修正 32 位量字序`

---

## 7. 里程碑路线图

> 按业余时间每周 10~15 小时估算。全职做约 6~8 周，业余约 3~5 个月。
> **每个里程碑都必须"可运行 + 可演示 + 有测试"**，不要攒到最后集成。

### M0 · 环境与骨架（3~5 天）
- 装 VS2022 Build Tools、WSL2 + Ubuntu 24.04、配置 vcpkg binary caching
- 建仓库、CMakePresets、vcpkg.json、`common` 空库 + 一个 Catch2 冒烟测试
- Windows 与 WSL 双端都能 `cmake --preset xxx && ctest` 通过
- ✅ 验收：CI（或本地脚本）双端绿灯；`argus-server --version` 与 `argus-client` 空窗口能启动

### M1 · Core 基础库（1.5~2 周）★ 打底最关键
- ThreadPool（基础版 → 优先级 → 定时 → 无锁队列/work-stealing 二选一进阶）
- MemoryPool（size-class slab + 大块 arena + FrameRef + tcache）
- RingBuffer（SPSC 无锁 + MPMC）、Logger（spdlog 封装）、Config（toml/json 热加载）、TimeUtil、Result/Expected
- 单元测试覆盖率 > 70%，TSAN/ASAN 全跑一遍
- ✅ 验收：`argus-bench` 输出对比报告（线程池 vs std::async；内存池 vs malloc），数据记进 docs

### M2 · Modbus 与虚拟设备（2 周）★ 项目灵魂之一
- `argus-sim`：Modbus TCP 从站（支持 FC 01/02/03/04/05/06/0F/10、异常码、多客户端并发、多实例端口）
- 物理仿真内核：温度一阶惯性 + PID 控温、转速、振动、产量计数、报警位域
- 故障注入 CLI：`inject overtemp` / `inject bearing_wear` / `inject comm_loss` / `inject net_delay 200`
- 服务端 ModbusMaster：连接管理、超时重传、断线重连、请求合并、字序配置
- TagTable 点表 + 死区 + 质量戳 + CSV 导入工具
- ✅ 验收：
  - 用第三方工具（Modbus Poll / QModMaster / pyModbus）连你的从站，读写全部正确
  - Wireshark 抓包，Modbus 解析结果与日志一致
  - 4 设备 × 50 点 @ 1s 采集，周期抖动 < 50ms；拔线后 3s 内恢复且期间质量戳为 Bad
  - 注入过温 → 服务端收到告警

### M3 · 存储与网关协议（1.5 周）
- SQLite 封装：连接管理、迁移、批量事务写、按天分表、保留策略、导出
- 自定义协议编解码（common 双端共用）+ epoll 服务器 + 会话/订阅/推送 + 命令下发
- 粘包半包、超大帧、恶意帧（长度字段伪造）的健壮性测试
- ✅ 验收：
  - 写入 1000 万样本，1 小时区间聚合查询 < 200ms，插入 ≥ 5 万行/s
  - 100 个模拟客户端并发订阅，服务端 CPU/内存曲线平稳，无 fd 泄漏
  - 模糊测试（随机字节流灌入）不崩溃

### M4 · 客户端骨架与实时数据（1.5 周）
- 主窗体 + 导航 + QSS 深色主题；NetClient（连接/心跳/重连/订阅）
- 总览看板：设备状态卡片 + QtCharts 实时曲线（20Hz 限流、环形数据窗）
- 点表浏览器：QTreeView + QAbstractTableModel，双击写值
- 告警中心：实时滚动 + 历史分页查询 + 确认
- ✅ 验收：连接后 3s 内出曲线；断网 30s 再恢复，界面自动重连不崩、不重复订阅；写值能真的改变仿真器状态

### M5 · OpenGL 3D 数字孪生（2 周）★ 最有成就感
- GlWidget + Shader/Mesh/Camera 基础设施；模型加载（手写几何 → tinygltf）
- 产线场景：地面、传送带滚动、设备模型、主轴旋转、状态指示灯、告警光柱
- 轨道相机、GPU 拾取、悬浮信息卡、视角预设与保存
- 数据驱动动画：状态插值、时间戳对齐、断线时模型进入"灰化"状态
- ✅ 验收：1080p 下稳定 60fps（含几百个 instancing 工件）；点选设备弹出正确信息；转速变化 → 传送带速度平滑变化无跳变

### M6 · 媒体链路（2~2.5 周）★ 最难，预留缓冲
- FFmpeg 拉流（RTSP/文件）→ 解码 → FrameRef（内存池）→ CvPipeline
- OpenCV 算子：背景差分运动检测、连通域计数、颜色/面积缺陷分类、OSD 叠加、JPEG 快照
- SynthCam 合成画面 + ground truth → 自动化准确率评测脚本
- 客户端视频墙：DecodeThread + FrameQueue + **YUV shader 渲染**，1/4/9 分屏、抓图、单路放大
- 音频：FFmpeg 解出音轨 → Qt QAudioOutput 播放 + 波形显示（练一下音频重采样）
- ✅ 验收：
  - 4 路 1080p25 解码 + 分析 + 转发，服务端 CPU < 60%，端到端延迟 < 300ms
  - 合成流上工件计数准确率 > 95%，缺陷分类 > 90%（有脚本自动跑）
  - 客户端 GPU 渲染 YUV 相比 CPU sws_scale 转 RGB，CPU 占用下降 ≥ 60%（记下具体数字）

### M7 · 闭环、压测与交付（1.5 周）
- **视觉 → Modbus 反控**：检测到连续 N 个缺陷件 → 写线圈急停 → 全链路 < 1s，3D 与视频墙同步体现
- 历史回放：时间轴三轨同步（曲线 + 视频 + 告警）
- 协议换 Protobuf（体验一次演进式重构，保持向后兼容）
- 压测：`argus-bench` 全链路 24h 稳定性，ASAN/TSAN/Valgrind 报告
- 部署：systemd unit + Dockerfile（服务端）、windeployqt + Inno Setup 安装包（客户端）
- 文档：README（含架构图、GIF 演示）、快速上手、开发指南、API 文档（Doxygen）
- ✅ 验收：一键脚本从零起环境 → 起仿真 → 起服务端 → 起客户端 → 看到完整闭环演示

---

## 8. 知识点 ↔ 模块 映射（你在练什么）

| 技术点 | 具体练到什么 | 在哪个模块 |
|---|---|---|
| C++17 | RAII、移动语义、`std::optional/variant/string_view`、模板、SFINAE/ concepts-lite、`std::filesystem` | common |
| 多线程 | 条件变量、原子与内存序、生产者消费者、双缓冲、生命周期与悬垂引用、死锁排查 | common / server |
| 线程池 | 任务队列、future、优先级、定时器、优雅停机、无锁队列、work-stealing、背压 | common::ThreadPool |
| 内存池 | 对齐、slab/free-list、arena、tcache、引用计数、红区检测、泄漏统计 | common::MemoryPool |
| 网络编程 | epoll(ET/LT)、IOCP、Reactor、粘包半包、心跳、背压、fd 泄漏、优雅关闭 | server::gateway / client::net |
| 协议设计 | 二进制帧、版本兼容、CRC、压缩、分片、TLV、Protobuf 迁移 | common::Protocol |
| Qt | 信号槽跨线程、事件循环、Model/View、QSS、自定义控件绘制、qrc、windeployqt | client |
| OpenGL | VBO/VAO/EBO、shader、MVP、光照、纹理、instancing、FBO、拾取、后处理 | client::gl |
| SQLite | WAL、事务批量、预编译、索引、查询计划、分表、迁移、并发 | server::storage / client::cache |
| FFmpeg | demux、解码器、PTS/DTS、swscale、avbuffer 引用计数、RTSP、音频重采样 | server::media / client::media |
| OpenCV | 背景差分、形态学、连通域、特征匹配、颜色空间、性能优化（UMat/多线程） | server::media |
| Modbus | MBAP、功能码、异常码、字节序/字序、超时重传、点表工程 | simulator / server::modbus |
| 工业软件概念 | 点表、质量戳、死区、报警分级、历史库、数字孪生、OEE | 全项目 |
| 工程能力 | CMake、vcpkg、CI、ASAN/TSAN、perf 剖析、benchmark、Docker、systemd、文档 | 全项目 |

---

## 9. 可以量化的成果指标（写简历用）

做完之后你应该能拿出这些数字（示例，实测后替换）：

- 单机采集 **4 台设备 × 200 点位 @ 1Hz**，采集周期抖动 **P99 < 30ms**
- 自研线程池处理 100 万个短任务耗时比 `std::async` **降低 XX%**
- 自研内存池相比系统 malloc，1000 万次分配/释放 **提速 X.X 倍**，长跑 24h **RSS 零增长**
- **4 路 1080p25** 视频拉流 + 解码 + CV 分析，服务端 CPU **< 60%**，端到端延迟 **< 300ms**
- YUV 直传 GPU shader 渲染，客户端 CPU 占用相比 sws_scale 方案 **下降 XX%**
- 合成流上工件计数准确率 **> 95%**（带自动化评测脚本与 ground truth）
- SQLite 批量写入 **> 5 万行/秒**，1000 万行下 1 小时区间聚合查询 **< 200ms**
- **100 并发客户端**长连接订阅，24h 压测无崩溃无泄漏（ASAN/TSAN/Valgrind 报告）
- 视觉缺陷 → Modbus 急停全链路响应 **< 1s**

配套交付：架构图、协议文档、Doxygen API 文档、benchmark 报告、演示视频/GIF、Docker 一键部署。

---

## 10. 风险与对策

| 风险 | 对策 |
|---|---|
| FFmpeg/OpenCV 在 Windows 下用 vcpkg 编译极慢（可能 1~3 小时） | 开启 vcpkg binary caching；或客户端直接用预编译包（BtbN builds / gyan.dev），服务端在 WSL 用 apt 装 |
| 新版 vcpkg 不支持 VS2017 | 装 VS2022 Build Tools（Qt 5.14 msvc2017_64 库与 v143 ABI 兼容，可继续用） |
| CMake 4.x 拒绝 Qt 5.14 的老 `cmake_minimum_required` | 传 `-DCMAKE_POLICY_VERSION_MINIMUM=3.5`，或装 CMake 3.28 LTS |
| WSL2 未安装 / 虚拟化未开 | 见 `argus-env-setup.md`；若公司电脑禁止虚拟化，退路是 VirtualBox（需 BIOS 开 VT-x）或一台云主机 |
| OpenGL 在 WSL/虚拟机里跑不动 3D | 3D 只在 Windows 客户端做，服务端不需要 GPU |
| 范围失控（想一次做完） | 严格按 M0→M7，每个里程碑先跑通"最小可用"，再回来加进阶特性 |
| 视频链路卡壳 | M6 拆成"文件解码通" → "RTSP 通" → "CV 通" → "客户端渲染通"四小步，每步独立可验 |
| 一个人开发容易烂尾 | 每周固定输出：一条 commit 记录 + 一张截图/GIF 存进 `docs/progress/`，形成正反馈 |

---

## 11. 后续版本扩展路线（V2+）

**V2 · 工程化与可扩展**
- CV 算子做成动态库插件（`dlopen`/`LoadLibrary` + ABI 稳定的 C 接口），配置化流水线编排
- 协议换 gRPC/WebSocket，支持 Web 客户端；Protobuf 全量 schema 化
- Prometheus exporter + Grafana 面板；结构化日志 + Loki
- Docker Compose 一键起（sim + server + mediamtx + grafana）

**V3 · 数据与规模**
- 时序数据库替换/并存 SQLite（TDengine / InfluxDB / TimescaleDB），做数据迁移与对比测试
- MQTT / Kafka 接入，支持多服务端节点与消息总线
- 分布式：一致性哈希分片设备、主备切换（练习 raft 或 etcd）

**V4 · 智能化**
- ONNX Runtime / NCNN 推理替换传统 CV（YOLO 目标检测），练模型部署与 GPU 加速
- 预测性维护：振动/温度时序特征 + 简单模型做故障预测
- 报警根因分析、OEE 自动计算

**V5 · 表现力**
- 3D 升级：glTF 2.0 PBR、IBL 环境光、SSAO、级联阴影、物理引擎（bullet/jolt）做机械臂运动学
- Qt WebAssembly 或 Vue + WebSocket 做浏览器端看板
- AR 巡检（手机扫码看设备孪生）

**V6 · 真机对接（等你有硬件时）**
- 接真实 PLC（西门子 S7-1200 / 汇川 / 台达）与真实 IP 摄像头（ONVIF/GB28181）
- 加 S7 协议、OPC UA、EtherNet/IP —— 你自研的协议栈此时就是护城河
- 边缘计算盒子部署（ARM64 + Jetson），交叉编译

---

## 12. 现在就可以做的三件事

1. 按 `argus-env-setup.md` 把 VS2022 Build Tools + WSL2 Ubuntu + vcpkg 依赖装好（约半天，主要是等编译）
2. 建 Git 仓库，把 `docs/` 里这两份文档提交进去，写下你自己的 M0 验收清单
3. **先写 M1 的 ThreadPool**：这是整个项目的地基，也是最容易独立完成、最有面试价值的一块

> 需要我把 M0 的仓库骨架（CMakeLists + CMakePresets + vcpkg.json + common 目录 + Catch2 冒烟测试 + CI）直接生成出来吗？说一声我就动手。
