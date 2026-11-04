# Argus 环境搭建指南（基于你当前这台机器的实测结果）

> 检测时间：2026-11-03 · 系统：Windows 11 25H2 (build 26200) · E 盘剩余 169.7 GB

## 0. 现状体检

| 组件 | 状态 | 路径 / 版本 |
|---|---|---|
| Visual Studio | ✅ 已装 | VS2017 Community 15.9.82，MSVC **14.16 (v141)** @ `C:\Program Files (x86)\Microsoft Visual Studio\2017\Community` |
| Qt | ✅ 已装 | 5.14.2，含 **msvc2017_64** 与 **mingw73_64** 两套 kit，模块齐全（Qt3D/Charts/Multimedia 都有） |
| Qt Creator | ✅ 已装 | `D:\Qt\Qt5.14.2\Tools\QtCreator` |
| MinGW | ✅ 已装 | GCC 7.3.0 @ `D:\Qt\Qt5.14.2\Tools\mingw730_64\bin\g++.exe`（太旧，只做备胎） |
| CMake | ✅ 已装 | **4.4.3**（⚠️ 见下方"CMake 4.x 坑"） |
| vcpkg | ✅ 已装 | `D:\vcpkg`，2026-07-27 版，triplet `x64-windows`，现有 mongo-cxx-driver / zlib |
| Git | ✅ 已装 | `D:\Program Files\Git` |
| **WSL** | ❌ **未安装** | 需安装 —— Linux 服务端的开发/运行环境 |
| **Docker** | ❌ 未安装 | 可选，做部署验证时用 |
| VirtualBox / VMware | ❌ 未安装 | WSL 的备胎方案 |

**结论**：Windows 侧基础很好，只差两件事 —— ①补一个 VS2022 编译器（vcpkg 需要），②装 WSL2 作为 Linux 侧。

---

## 1. 装 VS2022 Build Tools（必须）

**为什么**：我查了你 vcpkg 的源码，`D:\vcpkg\scripts\cmake\vcpkg_configure_cmake.cmake` 里只认
`Visual Studio 16 2019` 和 `Visual Studio 17 2022` 两个 generator —— **VS2017 已被新版 vcpkg 弃用**。
硬用 VS2017 会在装 opencv/ffmpeg 时报错。

```powershell
# 下载 Build Tools（免费，无需装完整 IDE）
# https://visualstudio.microsoft.com/zh-hans/downloads/ → "Visual Studio 2022 生成工具"
# 勾选工作负载：「使用 C++ 的桌面开发」+ 右侧「C++ CMake 工具」「MSVC v143」「Windows 11 SDK」
```

**好消息**：MSVC 2015/2017/2019/2022 二进制 ABI 兼容（微软官方保证），所以你现有的
**Qt 5.14.2 msvc2017_64 库可以直接和 v143 编译器链接**，不用重装 Qt。

> 如果你不想装 VS2022，退路是把 vcpkg 回退到 2022 年左右的 commit（支持 VS2017），
> 但那样很多新版端口用不了，不推荐。

---

## 2. 装 WSL2 + Ubuntu 24.04（必须）

你的 Linux 服务端需要一个真实的 Linux 环境。WSL2 是最省事的选择。

```powershell
# 管理员 PowerShell
wsl --install -d Ubuntu-24.04
# 需要重启一次；如果报"虚拟化未开启"，进 BIOS 打开 Intel VT-x / AMD-V
# 任务管理器 → 性能 → CPU → "虚拟化: 已启用" 可以先确认
```

重启后设置用户名密码，然后：

```bash
sudo apt update && sudo apt -y upgrade
sudo apt -y install build-essential gdb git cmake ninja-build pkg-config \
    libgl1-mesa-dev libxkbcommon-dev \
    libavcodec-dev libavformat-dev libavutil-dev libswscale-dev libswresample-dev \
    libopencv-dev libsqlite3-dev libspdlog-dev libfmt-dev \
    libboost-system-dev libboost-thread-dev \
    valgrind clang-tidy clang-format doxygen graphviz \
    python3-pip net-tools iproute2 lsof htop tmux
pip3 install pymodbus matplotlib   # 用来交叉验证你的 Modbus 从站
```

限制 WSL 内存（防止吃光宿主机），在 `%UserProfile%\.wslconfig`：

```ini
[wsl2]
memory=8GB
processors=6
swap=4GB
localhostForwarding=true
```

**网络注意**：WSL2 的 `localhost` 转发让 Windows 客户端能直接连 `127.0.0.1:5020` 访问 WSL 里的服务。
如果转发不生效，用 `hostname -I` 拿 WSL IP，或 `netsh interface portproxy` 手工映射。
反向（WSL 访问 Windows 上的 MediaMTX）需要用宿主机 IP，不是 localhost。

> **公司电脑禁虚拟化的备胎**：① VirtualBox + Ubuntu Server（仍需 BIOS 开 VT-x）
> ② 一台最便宜的云主机（2C4G 足够）+ VS Code Remote-SSH，体验其实最好
> ③ 极端情况：先在 Windows 上把服务端编出来跑通逻辑，只把 epoll 换成 IOCP（`common` 已经抽象好），后期再上 Linux

---

## 3. CMake 4.x 的坑（一定会踩，先记下来）

你装的是 **CMake 4.4.3**。CMake 4.0 起**移除了对 `cmake_minimum_required(VERSION < 3.5)` 的兼容**，
而 **Qt 5.14.2 自带的 CMake 配置文件正好是老版本声明**，于是会报：

```
Compatibility with CMake < 3.5 has been removed from CMake.
```

三种解法，任选其一（推荐第 1 种）：

```powershell
# ① 加一个变量兜底（最省事）
cmake -B build -DCMAKE_POLICY_VERSION_MINIMUM=3.5 ...

# ② 环境变量方式，写进 CMakePresets.json 的 cacheVariables
#    "CMAKE_POLICY_VERSION_MINIMUM": "3.5"

# ③ 装 CMake 3.28 LTS 与 4.x 共存
```

同理，从 GitHub 拉的第三方老项目（tinygltf、某些 vcpkg port）也可能报同样的错，都用 ① 解决。

---

## 4. vcpkg 依赖安装（Windows 侧）

**第一件事：开启 binary caching**，否则 opencv/ffmpeg 每次重编要几小时。

```powershell
# 用本地目录做缓存（简单）
[Environment]::SetEnvironmentVariable('VCPKG_BINARY_SOURCES','clear;files,D:\vcpkg-cache,readwrite','User')
# 或者用 NuGet 缓存：clear;nuget,C:\vcpkg-nuget,readwrite
```

项目用 **manifest 模式**（推荐），在仓库根放 `vcpkg.json`：

```json
{
  "name": "argus",
  "version-string": "0.1.0",
  "dependencies": [
    "spdlog", "fmt", "nlohmann-json", "tomlplusplus",
    "sqlite3", "catch2", "benchmark",
    { "name": "opencv4", "features": ["ffmpeg"] },
    { "name": "ffmpeg", "features": ["avcodec","avformat","swscale","swresample"] },
    "protobuf", "lz4", "tinygltf", "stb"
  ]
}
```

配合 `CMakePresets.json` 的工具链：

```json
{
  "version": 6,
  "configurePresets": [
    {
      "name": "base", "hidden": true,
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/${presetName}",
      "cacheVariables": {
        "CMAKE_TOOLCHAIN_FILE": "D:/vcpkg/scripts/buildsystems/vcpkg.cmake",
        "VCPKG_TARGET_TRIPLET": "x64-windows",
        "CMAKE_POLICY_VERSION_MINIMUM": "3.5",
        "CMAKE_CXX_STANDARD": "17"
      }
    },
    {
      "name": "win-msvc", "displayName": "Windows MSVC x64 Debug", "inherits": "base",
      "architecture": { "value": "x64", "strategy": "set" },
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Debug", "VCPKG_INSTALLED_DIR": "${sourceDir}/vcpkg_installed" }
    },
    {
      "name": "win-release", "inherits": "win-msvc",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "RelWithDebInfo" }
    },
    {
      "name": "linux-gcc", "inherits": "base",
      "condition": { "type": "equals", "lhs": "${hostSystemName}", "rhs": "Linux" },
      "cacheVariables": { "CMAKE_TOOLCHAIN_FILE": "", "CMAKE_BUILD_TYPE": "Debug" }
    },
    { "name": "win-tsan", "inherits": "win-msvc",
      "cacheVariables": { "ARGUS_ENABLE_TSAN": "ON" } },
    { "name": "linux-asan", "inherits": "linux-gcc",
      "cacheVariables": { "ARGUS_ENABLE_ASAN": "ON" } }
  ],
  "buildPresets": [
    { "name": "win-msvc", "configurePreset": "win-msvc" },
    { "name": "linux-gcc", "configurePreset": "linux-gcc" }
  ],
  "testPresets": [
    { "name": "win-msvc", "configurePreset": "win-msvc", "output": { "outputOnFailure": true } },
    { "name": "linux-gcc", "configurePreset": "linux-gcc", "output": { "outputOnFailure": true } }
  ]
}
```

**注意**：
- Ninja 你机器上没有，装 VS2022 Build Tools 时会带上（勾 CMake 工具即可），或者 `winget install Ninja-build.Ninja`
- `opencv4` + `ffmpeg` 首次编译 **1~3 小时**，建议睡前挂机，或者：
  - **客户端**用预编译包更快：FFmpeg 用 [gyan.dev](https://www.gyan.dev/ffmpeg/builds/) 的 shared 版，OpenCV 用官网 Windows 自解压包
  - **服务端**在 WSL 里直接 `apt install libopencv-dev libavcodec-dev`，秒装，不用 vcpkg
  - 这也是个真实的工程权衡：**Windows 用预编译包 + Linux 用系统包**，CMake 里用 `find_package` 统一抽象
- Qt 不走 vcpkg，用 `CMAKE_PREFIX_PATH=D:/Qt/Qt5.14.2/5.14.2/msvc2017_64`

---

## 5. 开发工具建议

| 用途 | 推荐 |
|---|---|
| 客户端 IDE | **Qt Creator**（你已装，对 Qt 项目体验最好）或 VS2022 + Qt VS Tools |
| 服务端 IDE | **VS Code + WSL Remote**（在 WSL 里直接开发，编译快、路径干净）或 CLion |
| Modbus 调试 | Modbus Poll（收费/试用）、**QModMaster**（免费开源）、pyModbus CLI、Wireshark（自带 Modbus 解析） |
| RTSP 服务 | **MediaMTX**（原 rtsp-simple-server，单文件绿色，`mediamtx.yml` 里配 `path: source: runOnDemand ...` 循环推 mp4） |
| 抓包/网络 | Wireshark、`tcpdump`、Postman（如果后面加 HTTP） |
| 性能剖析 | Linux: `perf`、`htop`、Valgrind；Windows: VS Profiler、Process Explorer、Intel VTune（免费版够用） |
| Sanitizer | `-fsanitize=address,undefined`（GCC/Clang）、`/fsanitize=address`（MSVC）；TSAN 建议只在 Linux 侧用 GCC 跑 |
| 文档 | Doxygen + Graphviz（生成调用图）、Mermaid（画架构图） |

### MediaMTX 最小配置（把 mp4 变成"IP 摄像头"）

```yaml
# mediamtx.yml
paths:
  cam0:
    runOnDemand: ffmpeg -re -stream_loop -1 -i /path/to/factory0.mp4 -c copy -f rtsp rtsp://localhost:8554/cam0
    runOnDemandRestart: yes
  cam1:
    runOnDemand: ffmpeg -re -stream_loop -1 -i /path/to/factory1.mp4 -c copy -f rtsp rtsp://localhost:8554/cam1
    runOnDemandRestart: yes
```

启动后你的服务端就能用 `rtsp://<host>:8554/cam0` 拉流，和真摄像头完全一样的代码路径。
没有素材视频？用你的 **SynthCam** 先生成，或者：

```bash
# 用 ffmpeg 造一段 60 秒测试视频（纯软件，不需要任何素材）
ffmpeg -f lavfi -i testsrc2=size=1920x1080:rate=25 -t 60 -c:v libx264 -pix_fmt yuv420p test1080p.mp4
```

---

## 6. 环境验收清单

装完逐条打勾：

- [ ] `cl.exe` 可用（在 "x64 Native Tools Command Prompt for VS 2022" 里 `cl` 有输出，版本 ≥ 19.3x）
- [ ] `cmake --version` ≥ 3.25，`ninja --version` 有输出
- [ ] `wsl -l -v` 显示 Ubuntu-24.04 且 VERSION=2
- [ ] WSL 内 `g++ --version` ≥ 11，`pkg-config --modversion opencv4` 有输出，`pkg-config --modversion libavcodec` 有输出
- [ ] `vcpkg install spdlog --triplet x64-windows` 能在 10 分钟内成功（验证 binary cache 与 VS2022 生效）
- [ ] Qt：`D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\qmake.exe -v` 正常，且能被 CMake `find_package(Qt5 COMPONENTS Widgets)` 找到
- [ ] MediaMTX 能起，`ffplay rtsp://localhost:8554/cam0` 能看到画面
- [ ] QModMaster 能连上你后面写的 `argus-sim`（M2 验收项）
- [ ] 建一个空仓库，Windows 与 WSL 双端 `cmake --preset xxx && cmake --build --preset xxx && ctest` 全绿（M0 验收项）

---

## 7. 一份可以直接跑的 M0 起手命令

```powershell
# Windows PowerShell（管理员装完 VS2022/WSL 之后）
mkdir E:\Dev\argus; cd E:\Dev\argus
git init
mkdir common,server,simulator,client,tools,docs,scripts,deploy,third_party,cmake
# 把 docs/argus-project-plan.md 和 docs/argus-env-setup.md 拷进 docs/
git add .; git commit -m "chore: 初始化 Argus 项目骨架与方案文档"
```

```bash
# WSL Ubuntu
sudo mkdir -p /mnt/e/Dev/argus          # 或者把仓库放在 WSL 原生分区（编译更快，推荐）
git clone /mnt/e/Dev/argus ~/argus && cd ~/argus
cmake --preset linux-gcc && cmake --build --preset linux-gcc && ctest --preset linux-gcc
```

> 性能提示：**WSL 里跨 `/mnt/e` 访问 Windows 分区的 IO 很慢**，编译大项目建议把仓库 clone 到 WSL 原生目录（`~/argus`），用 VS Code Remote-WSL 编辑，两边用 git 同步。
