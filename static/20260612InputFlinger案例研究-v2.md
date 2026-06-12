# InputFlinger 跨仓库演化案例研究报告 v2

> **数据源**：`tracker_aosp_cpp_46534e27`（frameworks/base）+ `tracker_aosp_native_20260609_8c2c6144`（frameworks/native）
>
> **本地源码**：`D:\work\linux-project\repos\AOSP`（base）+ `D:\work\linux-project\repos\AOSP-native`（native）
>
> **研究范围**：EventHub / InputReader / InputDispatcher / InputManagerService（C++ + Java）

## 1. 研究背景

InputFlinger 是 Android 输入系统的核心，负责从内核设备节点读取事件（EventHub）、处理转换（InputReader）、分发给目标窗口（InputDispatcher），并通过
JNI 桥接层（InputManagerService）连接 Java 框架层。

本研究覆盖 InputFlinger **跨两个仓库**的 14 年演化历史：2011-2014 年在 `frameworks/base`，2014-2025 年在
`frameworks/native`，同时追踪留在 base 中的 JNI 桥接层和 Java 层。

## 2. 跨仓库数据概览

### 2.1 两库规模对比

| 指标       | frameworks/base | frameworks/native |
|----------|-----------------|-------------------|
| 标识符总数    | 1,065,412       | 83,637            |
| 变更事件数    | 3,478,675       | 291,221           |
| Commit 数 | 1,064,617       | 126,370           |
| CPP 标识符  | 85,207          | 70,808            |
| JAVA 标识符 | 965,999         | 8,155             |

### 2.2 InputFlinger 核心文件标识符分布

**frameworks/base（历史数据，2011-2014）**：

| 文件                      | 路径                            | 标识符数      |
|-------------------------|-------------------------------|-----------|
| InputManagerService.cpp | services/core/jni/            | 489       |
| InputReader.cpp         | services/input/ + libs/input/ | 418 + 294 |
| InputDispatcher.cpp     | services/input/ + libs/input/ | 296 + 188 |
| EventHub.cpp            | services/input/ + libs/input/ | 104 + 71  |

**frameworks/native（2014-2025）**：

| 文件                   | 路径                                   | 标识符数         |
|----------------------|--------------------------------------|--------------|
| InputDispatcher.cpp  | services/inputflinger/dispatcher/    | 982          |
| InputReader.cpp      | services/inputflinger/reader/        | 439          |
| InputReader.cpp      | services/inputflinger/               | 412（旧路径，迁移前） |
| InputDispatcher.cpp  | services/inputflinger/               | 285（旧路径，迁移前） |
| EventHub.cpp         | services/inputflinger/reader/        | 189          |
| TouchInputMapper.cpp | services/inputflinger/reader/mapper/ | 156          |
| EventHub.cpp         | services/inputflinger/               | 100（旧路径）     |
| InputListener.cpp    | services/inputflinger/               | 100          |
| InputManager.cpp     | services/inputflinger/               | 53           |

**frameworks/base Java 层**：

| 文件                                    | 标识符数 |
|---------------------------------------|------|
| InputManagerService.java（core）        | 534  |
| InputManagerService.cpp（JNI，core/jni） | 489  |
| InputManagerService.java（旧路径）         | 156  |

## 3. 阶段划分（跨仓库视角）

### 阶段一：初创期（2011，base 仓库）

- base 仓库内 1,871 次变更事件，131 个 commit
- InputReader.cpp 贡献最多（999 次），大量 ADD 标识初次实现
- EventHub.cpp 从 InputDispatcher.cpp 中抽取 `InputWindow` 相关方法（4 个 CROSS_FILE_MOVE）
- InputReader.cpp 有 38 次重命名，接口快速迭代

### 阶段二：功能完善期（2012，base 仓库）

- base 仓库内 692 次事件，64 个 commit
- InputManagerService.cpp 开始活跃（174 次），JNI 桥接层成型
- Java 层 InputManagerService.java 开始活跃（482 次），API 体系建立

### 阶段三：模块化迁移期（2013，base 仓库）

- 746 次事件，36 个 commit
- 622 次 FILE_DIR_CHANGE — InputReader/InputDispatcher/EventHub 从 `services/input/` 迁移到 `libs/input/`
- 为后续独立仓库做准备

### 阶段四：跨仓库迁出（2014，base → native）

- base 仓库内 1,779 次事件（含大量 DELETE）
- native 仓库开始有数据：2,314 次事件，33 个 commit
- InputFlinger 从 `frameworks/base` 整体迁出到 `frameworks/native`
- 留在 base 的仅有 JNI 桥接层 InputManagerService.cpp 和 Java 层 InputManagerService.java

**关键数据**：2014 年 base 仓库 Java 层事件暴增至 957 次（108 个 commit），说明 JNI 和 Java 层在迁出过程中进行了大量适配。

### 阶段五：独立演化与初步扩展（2015-2017，native 仓库）

- native 仓库年度事件：380（2015）→ 60（2016）→ 307（2017）
- 2015 年：119 个 ADD，18 个 DELETE — 新能力引入
- 2017 年：84 个 ADD，169 个 BODY_CHANGE — 功能扩展和接口调整

**base 仓库 JNI/Java 层同步演化**：持续活跃，2016 年 Java 层 245 次（41 commit），说明 native 层变更持续驱动 JNI 层适配。

### 阶段六：第一次大重构（2018，native 仓库）

- native 仓库 815 次事件，66 个 commit
- **关键职责转移**：
    - `InputReader.cpp` → `InputReaderBase.cpp`：6 个方法抽出，形成基类（`applyTo`, `dump`, `dumpViewport`,
      `getDisplayViewport`, `setDisplayViewports`, `threadLoop`）
    - `InputWindow.cpp` → `libs/input/InputWindow.cpp`：9 个方法移到共享库（`addTouchableRegion`, `frameContainsPoint`,
      `isTrustedOverlay`, `overlaps`, `releaseInfo`, `supportsSplitTouch`, `touchableRegionContainsPoint`）

**解读**：这次重构的核心是"提取共享基础设施"——InputReaderBase 成为读取器的通用基类，InputWindow 的几何计算逻辑下沉到
`libs/input/` 供多个模块共用。

### 阶段七：大拆分重构（2019，native 仓库）

- **全年 4,977 次事件，139 个 commit** — native 仓库历史上最活跃的一年
- 2,530 次 FILE_DIR_CHANGE + 875 次 CROSS_FILE_MOVE — 规模空前的代码重组

**InputReader.cpp 大拆分**（459 个标识符迁出）：

| 目标文件                                       | 迁出职责                                         |
|--------------------------------------------|----------------------------------------------|
| reader/mapper/TouchInputMapper.cpp         | 触摸输入映射（abortTouches, process, sync 等核心触摸逻辑）  |
| reader/mapper/CursorInputMapper.cpp        | 光标/鼠标输入映射                                    |
| reader/mapper/KeyboardInputMapper.cpp      | 键盘输入映射                                       |
| reader/mapper/JoystickInputMapper.cpp      | 游戏手柄映射                                       |
| reader/mapper/RotaryEncoderInputMapper.cpp | 旋钮编码器映射                                      |
| reader/InputDevice.cpp                     | 设备管理（addMapper, bumpGeneration, cancelTouch） |
| reader/CursorButtonAccumulator.cpp         | 按键累积器                                        |
| reader/SingleTouchMotionAccumulator.cpp    | 单点触摸累积器                                      |

**InputDispatcher.cpp 大拆分**（95 个标识符迁出）：

| 目标文件                          | 迁出职责                                        |
|-------------------------------|---------------------------------------------|
| dispatcher/Entry.cpp          | 事件条目（DispatchEntry, MotionEntry, KeyEntry）  |
| dispatcher/TouchState.cpp     | 触摸状态（addGestureMonitors, addOrUpdateWindow） |
| dispatcher/InputState.cpp     | 输入状态（addKeyMemento, addMotionMemento）       |
| dispatcher/FocusResolver.cpp  | 焦点解析                                        |
| dispatcher/LatencyTracker.cpp | 延迟追踪                                        |

**解读**：这是 InputFlinger 从"巨石文件"到"分层模块"的转折点。之前 InputReader.cpp 和 InputDispatcher.cpp
各自是几千行的单一文件，承载了所有逻辑。2019 年的重构将它们拆分为 reader/ 和 dispatcher/ 两个子目录，每个 mapper
类型独立成文件，dispatcher 的状态管理、事件条目、延迟追踪也各自独立。

### 阶段八：持续演化与能力增强（2020-2025，native + base）

**native 仓库年度活跃度**：

| 年    | Commits | 事件    | ADD | DELETE | BODY_CHANGE | 重命名 |
|------|---------|-------|-----|--------|-------------|-----|
| 2020 | 219     | 3,667 | 759 | 154    | 2,151       | 33  |
| 2021 | 177     | 2,792 | 604 | 169    | 1,359       | 24  |
| 2022 | 283     | 4,272 | 749 | 430    | 1,895       | 19  |
| 2023 | 421     | 6,179 | 946 | 392    | 3,547       | 42  |
| 2024 | 319     | 4,667 | 714 | 495    | 2,622       | 45  |

**新增能力**（通过大量 ADD 事件识别）：

- **InputClassifier**（输入分类）：2020 年新增，用于将输入事件路由到 ML 分类器
- **PointerChoreographer**（指针编排）：2021 年前后，统一管理指针光标显示
- **MotionPredictor**（运动预测）：2022 年引入 `libs/input/MotionPredictor.cpp`，降低输入延迟感知
- **InputProcessor**（输入处理器）：2023 年新增 `InputProcessor.cpp`（19,631 行），整合分类和过滤流水线
- **GestureConverter / TouchpadInputMapper**（触控板手势）：2022-2023 年，支持触控板复杂手势识别
- **InputDeviceMetricsCollector**（设备指标采集）：2024 年新增，用于设备使用统计

**base 仓库 Java 层同步演化**（InputManagerService.java）：

| 年    | Commits | 事件  | ADD | DELETE |
|------|---------|-----|-----|--------|
| 2020 | 64      | 409 | 80  | 10     |
| 2022 | 74      | 596 | 101 | 36     |
| 2024 | 110     | 567 | 91  | 38     |

Java 层持续活跃，2024 年仍有 110 个 commit，说明 Android 框架与 Native 输入系统的接口仍在持续演进。

## 4. 职责变化深度分析

### 4.1 InputReader.cpp：从"万能读取器"到"薄协调层"

**2011 年**（base）：InputReader.cpp 是输入读取的**唯一**实现文件，999 次事件，承载所有设备类型的处理逻辑（触摸、键盘、轨迹球等）。

**2014 年**（迁入 native）：439 个标识符进入 native 仓库。

**2018 年**（第一次拆分）：基类方法提取到 `InputReaderBase.cpp`。

**2019 年**（大拆分）：412 个标识符中的大部分被拆分到 15+ 个独立 mapper 文件：

- `TouchInputMapper.cpp` 接收了最密集的迁移（abortTouches, process, sync, assignPointerIds 等 30+ 个方法）
- `CursorInputMapper.cpp`, `KeyboardInputMapper.cpp`, `JoystickInputMapper.cpp` 各自独立
- `InputDevice.cpp` 接收设备管理职责

**当前状态**（源码核实）：`reader/InputReader.cpp` 仅剩 1,096 行，是一个**薄协调层**——管理设备生命周期和事件路由，具体处理逻辑全部委托给各个
mapper。

### 4.2 InputDispatcher.cpp：从"单体分发"到"分层分发引擎"

**2011 年**（base）：682 次事件，集事件调度、状态管理、焦点解析、ANR 处理于一身。

**2019 年**（大拆分）：982 个标识符中的 95 个通过 CROSS_FILE_MOVE 迁出：

- `Entry.cpp`：事件条目抽象（DispatchEntry, MotionEntry, KeyEntry, FocusEventEntry）
- `TouchState.cpp`：触摸窗口状态（addGestureMonitors, addOrUpdateWindow, addPortalWindow）
- `InputState.cpp`：输入消费状态（addKeyMemento, addMotionMemento）
- `FocusResolver.cpp`：焦点解析逻辑
- `LatencyTracker.cpp` / `LatencyAggregator.cpp`：延迟监控

**当前状态**（源码核实）：`dispatcher/InputDispatcher.cpp` 仍有 7,335 行，是 InputFlinger 中最大的文件。核心分发逻辑（
`findTouchedWindowTargetsLocked` 158 次变更、`injectInputEvent` 72 次、`enqueueDispatchEntryLocked` 60 次）仍然集中在此文件。

### 4.3 EventHub.cpp：稳定的设备监听层

**2011-2014**（base）：104 + 71 个标识符，负责监听 `/dev/input/` 设备节点。

**2014-2025**（native）：从 inputflinger/ 迁移到 reader/ 子目录，当前 189 个标识符（2,980 行）。

**职责特征**：EventHub 是 InputFlinger 中**最稳定**的核心文件，职责从未发生本质变化——始终是 INotify + epoll
的设备事件监听器。变更主要是跟随 Linux 内核接口变化和性能优化。

### 4.4 InputManagerService（JNI + Java）：永不停歇的桥接层

**C++ JNI 层**（base 仓库）：`InputManagerService.cpp` 从 2012 年活跃至今，2024 年仍有 56 个 commit。
`register_android_server_InputManager` 函数累计变更 81 次，每次 Android 大版本更新都需要适配新的 native 方法注册。

**Java 层**（base 仓库）：`InputManagerService.java`（534 个标识符）从 2012 年活跃至今。

关键职责变化：

- **2014 年**（957 事件）：大版本适配，跟随 native 层迁出后的 API 变化
- **2020 年**（409 事件）：引入 InputFilter、权限管理增强
- **2022-2024 年**：持续增加设备配置、输入捕获（capture）、手势导航等新 API

### 4.5 新增能力追踪

| 能力                              | 首次出现年份    | 关键文件                                           | 说明                             |
|---------------------------------|-----------|------------------------------------------------|--------------------------------|
| InputClassifier                 | 2020      | InputClassifier.cpp                            | 将输入事件路由到 ML 分类器，实现输入事件的智能分类    |
| PointerChoreographer            | 2021      | PointerChoreographer.cpp（49,874 行）             | 统一管理指针光标显示和动画                  |
| MotionPredictor                 | 2022      | libs/input/MotionPredictor.cpp                 | 基于历史轨迹预测触摸位置，降低延迟感知            |
| InputProcessor                  | 2023      | InputProcessor.cpp（19,631 行）                   | 整合分类、过滤、监控的流水线架构               |
| TouchpadInputMapper             | 2022-2023 | reader/mapper/TouchpadInputMapper.cpp          | 触控板复杂手势识别（配合 GestureConverter） |
| InputDeviceMetricsCollector     | 2024      | InputDeviceMetricsCollector.cpp                | 设备使用指标采集和上报                    |
| LatencyAggregatorWithHistograms | 2023      | dispatcher/LatencyAggregatorWithHistograms.cpp | 直方图化的延迟聚合，用于性能监控               |
| InputTracer (Perfetto)          | 2023      | dispatcher/trace/                              | 基于 Perfetto 的输入事件追踪            |

## 5. 变更类型分布

### 5.1 native 仓库 InputFlinger 整体变更类型

| 变更类型                  | 次数     | 占比    |
|-----------------------|--------|-------|
| COMMON_BODY_CHANGE    | 13,479 | 46.3% |
| ADD                   | 6,210  | 21.3% |
| DELETE                | 2,628  | 9.0%  |
| COMMON_PARAM_CHANGE   | 1,965  | 6.8%  |
| FILE_DIR_CHANGE       | 1,960  | 6.7%  |
| LOGICAL_MODULE_CHANGE | 1,866  | 6.4%  |
| CROSS_FILE_MOVE       | 875    | 3.0%  |

### 5.2 Top 变更函数（排除测试代码）

| 函数                             | 文件                             | 变更次数 | 特征              |
|--------------------------------|--------------------------------|------|-----------------|
| findTouchedWindowTargetsLocked | dispatcher/InputDispatcher.cpp | 158  | 触摸目标查找，最复杂的分发逻辑 |
| dumpDispatchStateLocked        | dispatcher/InputDispatcher.cpp | 81   | 状态 dump，持续扩展    |
| dump                           | InputReader.cpp                | 80   | 读取器状态 dump      |
| configure                      | InputReader.cpp                | 75   | 配置管理，参数频繁变化     |
| injectInputEvent               | dispatcher/InputDispatcher.cpp | 72   | 事件注入，安全敏感       |
| sync                           | InputReader.cpp                | 66   | 同步循环            |
| notifyMotion                   | dispatcher/InputDispatcher.cpp | 62   | 运动事件通知          |
| process                        | InputReader.cpp                | 61   | 核心处理循环          |
| enqueueDispatchEntryLocked     | dispatcher/InputDispatcher.cpp | 60   | 分发入队            |
| reset                          | InputReader.cpp                | 57   | 状态重置            |

## 6. 总结

### 6.1 演化叙事

InputFlinger 经历了三次架构转折：

1. **2014 年跨仓库迁出**：从 `frameworks/base` 独立到 `frameworks/native`，标志着 Android 输入系统从"框架附属"变为"
   独立子系统"。

2. **2019 年大拆分**：InputReader 和 InputDispatcher 从单体巨石文件拆分为 reader/ 和 dispatcher/ 子目录，每个 mapper
   类型独立成文件。这次重构将 InputReader.cpp 从万能读取器变为薄协调层。

3. **2020-2024 年能力扩展**：在模块化架构基础上，持续引入新能力——ML 分类、运动预测、触控板手势、Perfetto
   追踪等。InputProcessor.cpp（19,631 行）的出现标志着流水线架构的成熟。

### 6.2 关键发现

1. **InputDispatcher.cpp 始终是最复杂的文件**：即使经过 2019 年的大拆分，它仍有 7,335 行和 982 个标识符，核心分发逻辑仍然高度集中。

2. **跨仓库 JNI 层的生命力**：`InputManagerService.cpp`（C++ JNI）和 `InputManagerService.java` 持续活跃 13 年，2024 年分别有
   56 和 110 个 commit，是连接两个仓库的关键纽带。

3. **CROSS_FILE_MOVE 揭示的职责演化**：875 次跨文件移动中，2019 年占绝大多数，清楚地刻画了"从单体到分层"的架构转型路径。

4. **新能力引入节奏加速**：2020 年后每年新增标识符 600-950 个，远超 2015-2017 年的年均 50-80 个，说明 InputFlinger
   在模块化之后进入了能力快速扩展期。