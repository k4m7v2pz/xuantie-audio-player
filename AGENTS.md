# Yumo Audio 音频混合播放库

## 项目概述

轻量级音频混合播放库，解决原生 API 同一时间只能播放一个音频的问题。本仓库为 `keyBonk-org/audio-player` 的 fork（origin：`k4m7v2pz/xuantie-audio-player`）。

**重写状态**：已用玄铁语言重写（2026-10-19）。玄铁版为当前活跃代码（`音频库.xt`/`演示.xt`/`tests/`），上游 C++ 实现保留在 `src/`（历史，Windows 独占）。

### 当前基线（玄铁）

- 语言：玄铁（全中文关键字，编译到原生码；`xtl` 工具链，`xtc tie` 编译）
- 播放后端：玄铁渲染库（raylib）音效/音乐流槽位池；WAV/MP3 解码、同源多实例叠加混音由渲染库提供
- 构建：`xtl b`（→ `dist/演示`）、`xtl t`（→ 跑 tests/）
- 目标平台：macOS arm64（渲染库随发行包提供 darwin 静态库）
- 能力边界：实例级音量/暂停/位置、播放完成回调、音效实例强停、全局音量/静音实际生效——渲染桥均未提供，库不提供假 API（详见 `音频库.xt` 头注释与 README）

### 历史基线（C++，已冻结）

- 语言/标准：C++17，宽字符（Unicode）构建
- 音频：Win32 `waveOut` + Windows ACM 重采样（msacm32.lib）
- 构建：MSYS2/MinGW GNU Make（64/32 位）
- 目标平台：仅 Windows（x64/x86）

### 核心定位

- **不同时播放**：解决原生 API 同一时间只能播放一个音频的问题
- **灵活控制**：支持预加载、动态添加播放、静音控制等
- **混合播放**：多个音频可以同时播放并混合

### 设计目标

- `addAudio` 添加音频后立即播放
- 提供 `preloadAudio` 接口允许预先加载标准音频对象
- 支持 `stopAll` 静音功能
- 允许同一音频的重复播放（通过播放实例追踪独立位置）

## 玄铁版开发须知

### 构建与测试

```bash
xtl b            # 构建演示（dist/演示；中文产物名在本项目不建窗口，无崩溃）
xtl t            # 运行 tests/ 逻辑测试
xtl r 演示.xt    # 编译并运行
```

### 玄铁工具链踩坑记录（写玄铁代码前必读）

1. **模块文件内 `造 自定义类型` 解析报「未定义的型」**（独立文件正常，模块内失败）→ 数据一律用字典字面量 + 动态成员访问（本项目预加载项/播放实例均为字典）。
2. **函数形参类型注解是静态检查**，字典字段访问（动态类型）传不进带注解形参 → 库函数形参一律不写类型注解；返回注解仅用于静态确定的返回值（整/字/逻辑/小数）。
3. **泛型类型注解（`结果<整>`、`数组<X?>`）在全局变量与函数签名处不可用** → 省略返回类型注解（`结果` 容器通过 `成功()`/`失败()` 构造，`.成功`/`.值`/`.错误` 判定）。
4. **`xtl t` 把测试程序放 /tmp 运行**，macOS 限制下音频设备初始化失败（AUDIO: Failed to initialize playback device）→ 测试必须环境容错：设备可用测成功路径，否则自动降级为失败路径（无效 ID/缺失文件/全局信号），禁止在成功假设下直接访问 `.值`。
5. 字典方法名以 runtime 为准：`包含?`/`含?` 判断、`表.键()` 取键数组（带括号调用，无括号走属性路径会段错误）、`删` 删键、下标读写。`键集` 不存在。
6. 中文可执行文件名在 macOS 下渲染库 GUI 程序窗口初始化会崩溃（raylib 非 ASCII 路径缺陷）；本项目纯 CLI（不建窗口）不受影响。
7. `xtl add 渲染 内置:核心库` 无法 git 拉取 → `tiepm_modules/` 必须入库跟踪，保证 clone 后 `xtl b` 直接可用。

### 文件结构（玄铁版）

```
├── 音频库.xt            # 库模块（预加载池/播放池/全局信号，字典数据结构）
├── 演示.xt              # 入口（合成音效 + WAV 演示）
├── tests/音频库.xt      # 逻辑测试（环境容错）
├── xtl.json / xtl.lock  # 项目配置与依赖锁（渲染：内置:核心库）
└── tiepm_modules/       # 渲染库副本（入库跟踪）
```

## 依赖库

| 库 | 说明 |
|----|------|
| Windows ACM | 系统自带音频转换库（msacm32.lib） |
| winmm | Windows 多媒体 API |

### 编译命令

```powershell
make          # 构建 64 位静态库并打包（输出至 build/）
make 32       # 构建 32 位版本（需 i686-w64-mingw32-g++）
make release  # 同时构建 64/32 位
```

手动编译单个测试：

```powershell
g++ -std=c++17 -I"src" -o output.exe source.cpp src/audioPlayer.cpp -lmsacm32 -lwinmm -lcomdlg32
```

### 编译中间产物

编译过程中生成的中间文件应放置在以下被 `.gitignore` 忽略的路径：

| 文件类型 | 存放路径 | 说明 |
|----------|----------|------|
| `.exe` 测试可执行文件 | `test/compiled/` | 测试程序输出 |
| 静态库 / 打包产物 | `build/` | make 构建输出 |

**不要将中间产物放在项目根目录**，以免污染源码仓库。

### 测试文件编译

测试文件编译至：`./test/compiled/`

## API 接口

### 核心接口

| 接口 | 说明 |
|------|------|
| `AudioPool::getInstance()` | 获取单例实例 |
| `preloadAudio(filename, ready)` | 异步预加载音频文件 |
| `addAudio(preloadedId)` | 添加预加载音频并立即播放 |
| `addAudio(filename)` | 简化用法：异步加载并播放 |
| `stopAll()` | 停止所有播放（挂起） |
| `resume()` | 恢复播放 |

### 查询接口

| 接口 | 说明 |
|------|------|
| `getPreloadedCount()` | 获取预加载音频数量 |
| `getPlayingCount()` | 获取当前播放实例数量 |
| `isPlaying(instanceId)` | 检查播放实例是否正在播放 |

### 控制接口

| 接口 | 说明 |
|------|------|
| `setVolume(instanceId, volume)` | 设置播放实例音量 |
| `setMuted(muted)` | 设置全局静音状态 |
| `resetAll()` | 重置所有播放实例位置到开头 |

## 数据结构

### PreloadedAudio

存储共享的音频数据，同一个预加载音频可创建多个播放实例。

```cpp
struct PreloadedAudio {
    StandardWavInfo data;  // 重采样后的音频数据（44.1kHz, 双声道, 16位）
};
```

### PlayInstance

每次 `addAudio` 创建，追踪独立的播放位置。

```cpp
struct PlayInstance {
    PreloadedAudio* source;  // 指向共享的预加载音频数据
    size_t position;         // 当前播放位置（样本索引）
    float volume;            // 音量（0.0-1.0）
    bool active;             // 是否激活播放
};
```

### StandardWavInfo

标准音频格式：`std::vector<int16_t>` - 44.1kHz, 双声道, 16位 PCM

## 核心实现

### 音频格式

| 参数 | 值 |
|------|-----|
| 采样率 | 44.1kHz |
| 声道数 | 双声道 |
| 位深度 | 16位 |
| 缓冲区大小 | 8820 样本（100ms） |

### 双缓冲技术

使用两个音频缓冲区交替播放，避免音频断续：

1. `playAll()` 或首次 `addAudio` 时预先准备2个缓冲区
2. `waveOutCallback` 在缓冲区播放完成后准备下一个缓冲区
3. 保证音频设备始终有数据可播放

### 异步加载

`preloadAudio` 使用后台线程异步加载，不阻塞音频播放：

1. 主线程预留位置并返回 ID
2. 后台线程加载 WAV 文件并使用 Windows ACM 重采样
3. 通过 `std::atomic<bool>` 标记加载完成状态
4. `addAudio(preloadedId)` 检查数据就绪后创建播放实例

### Windows ACM 重采样

使用 Windows ACM API 进行音频格式转换：

1. `acmStreamOpen` - 打开转换流
2. `acmStreamSize` - 计算目标缓冲区大小
3. `acmStreamPrepareHeader` - 准备转换头
4. `acmStreamConvert` - 执行转换
5. `acmStreamClose` - 关闭转换流

### 线程安全

- 使用 `std::mutex` 保护共享数据
- `std::atomic<bool>` 用于跨线程状态同步
- 使用 `std::unique_ptr` 管理动态分配的预加载音频数据，避免 vector 扩容导致的悬空指针
- 所有公开接口都是线程安全的

## 使用示例

### 简化用法（快速播放）

```cpp
AudioPool& pool = AudioPool::getInstance();
pool.addAudio(L"test.wav");  // 异步加载，立即返回
pool.addAudio(L"test2.wav");  // 混合播放
```

### 标准用法（预加载 + 控制）

```cpp
AudioPool& pool = AudioPool::getInstance();
std::atomic<bool> ready(false);

// 预加载
size_t id1 = pool.preloadAudio(L"test.wav", ready);
while (!ready.load()) Sleep(10);

// 添加播放
size_t instanceId = pool.addAudio(id1);
pool.setVolume(instanceId, 0.5f);

// 停止
pool.stopAll();

// 恢复
pool.resume();
```

## 注意事项

1. **路径问题**：使用相对路径时注意工作目录，测试程序从 `test/compiled/` 运行
2. **音频播放完毕**：播放实例不会自动清理，需手动 `stopAll()`
3. **设备状态**：`stopAll()` 不会关闭音频设备，只是停止并重置位置
4. **内存管理**：音频数据由库管理，调用者无需释放
5. **停止与恢复**：`stopAll()` 后需要调用 `resume()` 才能重新播放

## 测试文件

| 文件 | 说明 |
|------|------|
| `test/1_readAudioInfo.cpp` | WAV 文件信息读取测试 |
| `test/2_resample_test.cpp` | 音频重采样测试 |
| `test/3_audio_pool_test.cpp` | addAudio 字符串版本测试 |
| `test/4_audio_mix_test.cpp` | 预加载 + 延迟添加测试 |
| `test/5_simple_test.cpp` | 单音频播放测试 |
| `test/6_play_control_test.cpp` | 播放控制测试 |
| `test/7_control_test.cpp` | 控制接口测试 |
| `test/8_mp3_test.cpp` | MP3 播放测试 |
| `test/9_Cirno.cpp` | 综合测试 |

## 文件结构

```
audio-player/
├── 音频库.xt              # 玄铁库模块（活跃代码，见「玄铁版开发须知」）
├── 演示.xt                # 玄铁入口
├── tests/                 # 玄铁测试（xtl t）
├── xtl.json / xtl.lock    # xtl 配置与依赖锁
├── tiepm_modules/         # 渲染库副本（入库跟踪）
├── src/                   # C++ 历史实现（冻结）
│   ├── audioPlayer.hpp    # 头文件
│   ├── audioPlayer.cpp    # 实现
│   └── yumo_except.hpp    # 异常定义
├── test/                  # C++ 历史测试
│   ├── audio/             # 测试音频文件
│   ├── compiled/          # 编译输出目录
│   ├── build_manager.py   # 编译管理脚本
│   └── *.cpp              # 测试源文件
├── makefile               # C++ GNU Make 构建（64/32 位）
├── README.md              # 项目说明（玄铁版为当前主说明）
└── AGENTS.md              # 本文档
```

## AI开发须知

- **玄铁版为当前活跃代码**：修改后必须 `xtl b` 构建通过 + `xtl t` 测试通过再提交（tests 已做环境容错，设备不可用时自动降级失败路径）。
- 玄铁代码遵守「玄铁工具链踩坑记录」七条（见上文），尤其是：数据用字典、形参不写类型注解、测试不假设设备可用。
- C++ 历史代码：不再主动改动；确需修改时，编译对应测试文件到 `test/compiled/` 路径下以便检测。
- 无 Windows 环境时，C++ 改动至少保证不破坏既有目录结构与接口，并做静态核对。
- 本仓库文档约定对齐 `xtl-toolchain/AGENTS.md` 的提交规范。

### 编译管理脚本

使用 `build_manager.py` 脚本编译测试文件：

```powershell
cd test
python build_manager.py
```

脚本支持：
- 按序号选择编译文件（如 `1 3 4-6`）
- `all` 选项编译全部测试文件
- 自动输出到 `test/compiled/` 目录

## 提交约定

### 自动提交授权

本仓库允许 Agent 在完成用户既定的改动后，**自动执行 `git add` / `commit` / `push` 到当前跟踪分支，无需逐次等待人类批准**。自动提交仍须遵守：

- 先过「提交前防泄漏守卫」核对清单与内容；
- 如实写 Co-Authored-By 与提交小尾巴；
- **禁止 force push / `--force-with-lease`**；确需改写历史必须先说明理由并给出备份方案；
- 只提交用户本次明确要求范围内的改动，不擅自夹带超出范围的内容。

### 提交尾部 Co-Authored-By

提交信息末尾用 trailer（空行隔开）`Co-Authored-By`，名字填写**实际调用工具、执行本次提交的模型/Agent**（不是写代码的模型，二者可能不同——如强力模型写代码、切小模型来提交）；不知道实际提交者则该名字不写。`noreply@` 域名无对应则不写域名只留名字。格式示例：`Co-Authored-By: <实际提交的Agent名> <noreply@<对应域名>>`。

### 提交前防泄漏守卫（硬约束）

进入 git 的内容默认互联网可见（公开仓库直接可见）。**禁止提交**：私钥/token/密码、个人邮箱、真实姓名、私人服务器地址、代理端口（HTTP/SOCKS 复用端口）、内部 IP、SSH 口令、VPS/VM 连接信息。

- 敏感凭据与连接信息一律不入 commit/issue/PR/公开文档；需本地保存时放入已被 `.gitignore` 的目录（如 `secrets/`）。
- 文档里举例远端/邮箱/端口一律用占位符：`<example@example.com>`、`<proxy-port>`、`<ssh-port>`、`<host>`。
- **提交前核对流程（不得跳过）**：
  1. `git add` 后先 `git diff --cached --name-only` 确认暂存区文件清单，无 `secrets/`、`*.pem`、`id_rsa`、`id_ed25519`、`*.key`、`credentials` 等；
  2. `git diff --cached` 扫内容，发现真实 IP / 端口 / 邮箱 / token → `git restore --staged <file>` 摘出，替换为占位符或删除后重新 `git add`；
  3. 确认无泄漏再 `git commit`。

### 提交小尾巴（如实标注，禁止编造）

issue/PR/追评小尾巴如实标注人的参与程度，**禁止把「批准」写成「审阅/查看」**：

- 仅当人类**实际查看过**内容（描述/代码）才写"已查看/已审阅"。
- 人类仅授权、未逐字审阅 → 只写"批准"，档位按「是否理解（能否接住）」划分：
  - 已理解 → `人类已在Agent对话中批准提交（已理解）`
  - 未能完全理解（如正文表格未接住）→ `人类已在Agent对话中批准提交（未能完全理解，可附具体）`
- 末尾如实标注提交方式与提交者：`提交方式：Personal access tokens (classic)｜提交者：豆包工作 Mac版`；模型/推理强度可选（AI 知道才写）。
- 已发内容的小尾巴不改。
