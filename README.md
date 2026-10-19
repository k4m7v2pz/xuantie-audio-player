# Yumo Audio

> 本仓库 fork 自 [keyBonk-org/audio-player](https://github.com/keyBonk-org/audio-player)。
> 已用玄铁语言重写（见下方「玄铁版」），上游 C++ 实现保留在 `src/`，见文末「上游 C++ 版（历史）」。

## 玄铁版（xuantie-audio-player）

用 [玄铁语言](https://gitcode.com/MARKJY/XuanTie-Lang)（全中文关键字、编译到原生码）重写的 Yumo Audio，播放后端为玄铁渲染库（raylib）的音效/音乐流槽位池。

### 文件结构

```
xuantie-audio-player/
├── 音频库.xt            # 库模块：预加载池/播放池/实例控制/全局信号
├── 演示.xt              # 项目入口：合成音效 + WAV 播放演示
├── tests/音频库.xt      # 逻辑测试（xtl t 运行）
├── xtl.json / xtl.lock  # xtl 项目配置（依赖：渲染，内置:核心库）
├── tiepm_modules/       # 渲染库依赖副本（跟踪，clone 即可构建）
├── src/ test/           # 上游 C++ 实现（历史，Windows 独占）
└── AGENTS.md            # 开发约定（自动提交/防泄漏/小尾巴）
```

### 构建与测试

```bash
xtl b                 # 构建 → dist/演示
xtl t                 # 运行测试（tests/ 下）
xtl r 演示.xt         # 编译并运行演示
```

### API（与 C++ / Rust 版语义对齐）

| 函数 | 对应上游 | 说明 |
|------|----------|------|
| `预加载(路径)` | preloadAudio | 加载文件：.mp3 系走音乐流池，其余走音效池 |
| `预加载合成(频率, 毫秒)` | （玄铁版新增） | 生成正弦波音效，零资产可用 |
| `播放(预加载ID)` | addAudio | 创建播放实例并立即播放（音效可叠加混音） |
| `播放文件(路径)` | addAudio(filename) | 便利接口：加载并播放，用完自动释放 |
| `重取(实例ID)` | regain | 按 ID 取回播放实例 |
| `播放中?(实例ID)` | isPlaying | 查询实例激活状态 |
| `移除(实例ID)` | remove | 移除实例（音乐流同时停止） |
| `移除预加载(预加载ID)` | removePreloadedAudio | 被引用则标记待删，播放结束后释放 |
| `取预加载数()` / `取播放数()` | getPreloadedCount/getPlayingCount | 数量查询 |
| `重置全部()` | resetAll | 音乐流停掉重播；音效重新触发 |
| `更新音乐流(句柄)` / `更新全部音乐流()` | （玄铁版新增） | 音乐流需周期性推进解码 |
| `设全局停止/静音/音量` | yumo::global | 全局信号（见能力边界） |
| `关闭音频()` | （玄铁版新增） | 释放全部资源并关闭设备 |

### 能力边界（如实声明）

玄铁渲染桥（raylib）未提供以下能力，玄铁版**不提供假 API**，调用方需自行规避：

- **实例级音量/暂停/位置**、**播放完成回调**：渲染桥无对应控制
- **音效实例单独强停**：raylib Sound 无实例句柄（音乐流可停）
- **全局静音/音量**：为逻辑标记/占位，实际不生效；全局停止可真正停音乐流

### 已知玄铁工具链限制（本项目踩坑记录）

- 模块文件内 `造 自定义类型` 解析报「未定义的型」→ 用字典字面量 + 动态成员访问代替（本项目数据结构全为字典）
- 函数形参类型注解为静态检查，字典字段访问（动态类型）无法传入带注解形参 → 形参不写类型注解
- 泛型类型注解（`结果<整>`、`数组<X?>`）在全局变量/函数签名处不可用 → 省略返回注解
- `xtl t` 把测试程序放到 /tmp 运行，macOS 限制下音频设备初始化失败 → 测试做环境容错（设备可用测成功路径，否则测失败路径）

---

## 上游 C++ 版（历史）

Yumo Audio是利用WIN32 API实现的C++音频播放库，主要聚焦于实现音频混合等基础API无法满足的功能，属于自用型库，完全服务于[keyBonk](https://github.com/xiaoditx/keyBonk)所以有一些通用功能将不予考虑

由于上文提到的自用特性，本库结构异常混乱，不支持CMake构建，部分文件缺失需要手动补齐，因此暂不建议直接使用。如果必须使用，请自行改造本库。

本库初期版本基本为纯vibe产物，后期由人工进行，目前所有vibe代码已经经过了人工初审，但仍请谨慎使用。

目前开发进度：基本实现，接口已经基本实现，但是格式支持于兼容性还有待提升

## 一. 获取库文件

可以从[release](https://github.com/keyBonk-org/audio-player/releases)中获取库文件，也可以通过手动构建来实现

手动构建流程如下：

```powershell
git clone https://github.io/keybonk-org/audio-player/
cd audio-player
make          # 获取本地 g++ 编译的版本
make 32       # 获取 32 位版本（需 i686-w64-mingw32-g++）
```

参考文档“[为make管理的项目引入Yumo audio](https://keybonk-org.github.io/docs/audio-player/make_get/)”可以为通过make管理的项目引入Yumo audio并实现自动下载

## 二. 示例

播放音频：

```cpp
#include <yumo/audio.hpp>
#include <chrono>
#include <thread>

int main() {
    yumo::readySign ready(false);
    auto preloadId = yumo::preloadAudio(L"bgm.wav", &ready);

    while (!ready) {
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }

    auto inst = yumo::addAudio(preloadId, 0.8f);
}
```

控制播放状态：

```cpp
auto inst = yumo::addAudio(preloadId, 1.0f);

inst.volume = 0.5f;          // 音量 50%
inst.stopped = true;         // 暂停
inst.stopped = false;        // 继续播放
inst.muted = true;           // 静音（位置继续推进）
inst.muted = false;          // 取消静音
inst.position = 44100 * 10;  // 跳到第 10 秒（按采样点计）
```

## 三. 兼容性

### 1.平台
Windows独占

### 2.文件格式

（格式按扩展名记）

| 格式 | 是否支持 | 支持程度 |
|:------:|:---------:|:----------:|
|WAV   |支持     |不支持非PCM格式以及超过2声道的WAV文件|
|PCM   |即将支持  | -       |
|MP3   |支持     | 支持大多MP3文件 |
|MP4   |不支持    | -       |
|M4A   |不支持    | -       |
|OGG   |不支持    | -       |
|FLAC  |不支持    | -       |
|WMA   |不支持    | -       |

### 3.是否兼容C语言

Yumo Audio大量采用C++特性编写，不兼容C语言，虽然可以通过函数包装链接到C，但库内不提供这些包装

## 四. 技术栈

- C++
- WIN32 API

## 五. 文档

- [开始使用 Yumo audio](https://keybonk-org.github.io/docs/audio-player/usage/)
- [音频库选择指南](https://keybonk-org.github.io/docs/audio-player/choice/)
