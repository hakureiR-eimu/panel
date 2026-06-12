# AOSP InputFlinger C++ 案例研究报告

> 数据库：`tracker_aosp_cpp_46534e27` · 仓库：`platform_frameworks_base` · 研究维度：dim1（阶段）、dim2（职责）、dim3（接口稳定性）

## 1. 研究背景

InputFlinger 是 Android 系统的输入事件处理核心组件，负责从内核设备节点读取原始输入事件（EventHub），经过处理和转换（InputReader），最终分发给目标窗口（InputDispatcher）。整个流程通过
JNI 桥接层（InputManagerService.cpp）与 Java 层的 InputManagerService 连接。

本研究追踪 InputFlinger 四个核心文件在 `platform_frameworks_base` 仓库中的演化历史，涵盖 2011 至 2024 年。

## 2. 研究范围

| 文件                      | 数据库路径                         | 标识符数      | 活跃年份           |
|-------------------------|-------------------------------|-----------|----------------|
| InputManagerService.cpp | services/core/jni/            | 489       | 2012-2024      |
| InputReader.cpp         | services/input/ + libs/input/ | 418 + 294 | 2011-2014      |
| InputDispatcher.cpp     | services/input/ + libs/input/ | 296 + 188 | 2011-2014/2017 |
| EventHub.cpp            | services/input/ + libs/input/ | 104 + 71  | 2011-2014      |

**注意**：InputReader.cpp、InputDispatcher.cpp、EventHub.cpp 于 2014 年从 `frameworks/base` 迁出到 `frameworks/native`
，因此它们在本数据库中只有 2011-2014 年的数据。InputManagerService.cpp 留在 `frameworks/base` 作为 JNI 桥接层，持续活跃至今。

## 3. dim1: 阶段与里程碑

### 3.1 总体活跃度

14 年间，四文件共产生 5,394 次变更事件（不含 STYLE_CHANGE_ONLY / UNCHANGE），涉及 492 个 commit。

### 3.2 阶段划分

基于年度活跃度直方图和 FILE_DIR_CHANGE / 大规模 ADD+DELETE 事件，划分为 5 个阶段：

#### 阶段一：初创期（2011）

- **事件**：1,871 次，131 个 commit
- InputReader.cpp 贡献最多（999 次事件），大量 ADD（250 次）标识初次实现
- EventHub.cpp 从 InputDispatcher.cpp 中抽取 `InputWindow` 相关方法（4 个 CROSS_FILE_MOVE）
- InputReader.cpp 有 38 次重命名（IDENTIFIER_NAME_CHANGE），接口快速迭代

**解读**：Android 输入系统从零构建，InputReader 作为核心处理逻辑，承载了最密集的开发活动。

#### 阶段二：功能完善期（2012）

- **事件**：692 次，64 个 commit
- 新增 94 个标识符，接口逐步稳定（仅 10 次重命名）
- InputManagerService.cpp 开始活跃（174 次事件），JNI 桥接层成型

**解读**：输入系统基本架构稳定，开始完善 JNI 桥接和输入策略。

#### 阶段三：模块化迁移期（2013）

- **事件**：746 次，36 个 commit
- **关键事件**：622 次 FILE_DIR_CHANGE — InputReader/InputDispatcher/EventHub 从 `services/input/` 迁移到 `libs/input/`
- InputManagerService.cpp 从 `services/jni/` 迁移到 `services/core/jni/`
- 这是一次**仓库内目录重组**，将输入系统从 `services` 层独立到 `libs` 层

**解读**：Android 4.2/4.3 时期的模块化重构，将 InputFlinger 从系统服务依赖中解耦出来，为后续独立仓库做准备。

#### 阶段四：迁出与重写期（2014）

- **事件**：1,779 次，27 个 commit
- **关键事件**：2014-02-11 三次大规模 commit，各涉及 549 个标识符
    - 555 个 ADD、1,101 个 DELETE — InputFlinger 从 `frameworks/base` 迁出到 `frameworks/native`
- InputReader.cpp 产生 891 次事件（294 ADD / 588 DELETE）— 近乎全部标识符被删除后重建
- InputDispatcher.cpp 产生 578 次事件（185 ADD / 370 DELETE）

**解读**：InputFlinger 从 `frameworks/base` 整体迁出到独立的 `frameworks/native` 仓库，这是 Android 输入系统最重要的架构变更之一。留在
`frameworks/base` 的仅有 JNI 桥接层 InputManagerService.cpp。

#### 阶段五：JNI 桥接持续演化期（2015-2024）

- 仅 InputManagerService.cpp 持续活跃
- 2020 年最活跃（273 次事件，50 个 commit），2024 年仍保持 375 次事件
- 标识符净增长趋于平衡（ADD ≈ DELETE），说明 JNI 层跟随 Android 框架持续适配

**解读**：InputFlinger 核心逻辑已在 `frameworks/native` 独立演化，`frameworks/base` 中的 JNI 层作为桥梁持续跟随 Android 框架
API 变化。

## 4. dim2: 职责变化

### 4.1 四类职责变化统计

| 变化类型                  | 次数    | 说明         |
|-----------------------|-------|------------|
| ADD（诞生）               | 1,134 | 新职责诞生      |
| DELETE（清算）            | 1,495 | 职责被彻底删除    |
| FILE_DIR_CHANGE（整体迁移） | 986   | 文件随目录改名    |
| CROSS_FILE_MOVE（职责让渡） | 4     | 方法从本文件搬到别处 |

### 4.2 关键职责让渡事件

**2011 年：从 InputDispatcher 抽取 InputWindow**

4 个方法从 `InputDispatcher.cpp` 迁移到 `InputWindow.cpp`：

- `touchableAreaContainsPoint`
- `frameContainsPoint`
- `isTrustedOverlay`
- `supportsSplitTouch`

这体现了"窗口几何判断"职责从输入分发逻辑中独立出来的早期设计决策。

### 4.3 2013 年目录迁移详情

InputReader.cpp 有 ~90 个函数通过 FILE_DIR_CHANGE 从 `services/input/` 迁移到 `libs/input/`，包括核心方法：

- `process`（15 次）、`reset`（14 次）、`configure`（9 次）、`dump`（8 次）
- `getScanCodeState`（6 次）、`populateDeviceInfo`（6 次）、`getSources`（6 次）
- 各种 Accumulator 和状态查询方法

这是**整体文件搬迁**，不涉及职责拆分，所有方法作为一个整体迁移。

### 4.4 2014 年迁出详情

2014-02-11 的三次 commit 实现了 InputFlinger 从 `frameworks/base` 到 `frameworks/native` 的迁出：

- 第一次 commit：在 `libs/input/` 新增 549 个标识符（可能是新路径的初始版本）
- 第二、三次 commit：删除旧路径下的 549 + 549 个标识符

InputReader.cpp 有 294 个 ADD + 588 个 DELETE，接近 1:2 的比例，暗示迁出过程中可能发生了代码重构（部分函数被合并或重新组织）。

## 5. dim3: 接口稳定性

### 5.1 变更类型分布

| 接口相关变更                 | 次数    | 占比    |
|------------------------|-------|-------|
| COMMON_BODY_CHANGE     | 2,524 | 46.8% |
| COMMON_PARAM_CHANGE    | 416   | 7.7%  |
| IDENTIFIER_NAME_CHANGE | 83    | 1.5%  |
| RETURN_TYPE_CHANGE     | 43    | 0.8%  |
| COMMON_MODIFIER_CHANGE | 5     | 0.1%  |

### 5.2 Top 变更函数

| 函数                                   | 文件                      | 变更次数    | 特征               |
|--------------------------------------|-------------------------|---------|------------------|
| register_android_server_InputManager | InputManagerService.cpp | 81      | JNI 注册入口，持续适配    |
| process                              | InputReader.cpp         | 73 + 60 | 核心处理循环，频繁参数和逻辑变更 |
| sync                                 | InputReader.cpp         | 71 + 56 | 同步逻辑             |
| configure                            | InputReader.cpp         | 69 + 38 | 配置方法，参数频繁变化      |
| reset                                | InputReader.cpp         | 50 + 56 | 状态重置             |
| findTouchedWindowTargetsLocked       | InputDispatcher.cpp     | 41      | 触摸目标查找，最复杂的分发逻辑  |

### 5.3 重命名分析

共 83 次重命名事件，集中在两个时期：

**2011 年（38 次）**：初创期接口快速迭代

- `prepareTouches` → 多次重命名（触摸准备逻辑的命名探索）
- `processEventsForDevice` / `processEventsForDeviceLocked` → 命名规范化（加 Locked 后缀表示线程安全）
- 大量 `Locked` 后缀的重命名，体现多线程安全设计的引入

**2011 年 EventHub 大规模重命名**（commit 93FA5B30）：

- `loadVirtualKeyMapLocked`、`readNotifyLocked`、`setKeyboardPropertiesLocked` 等 11 个方法重命名
- 统一添加 `Locked` 后缀，标记需要持有锁的方法

**2020 年（19 次）**：InputManagerService 的大规模适配

- `nativeGetInputDeviceIds` 等方法重命名，跟随 Android 输入 API 的大版本变更

### 5.4 参数变化趋势

416 次 COMMON_PARAM_CHANGE 中：

- InputReader.cpp 的 `process`、`sync`、`configure` 方法是参数变化最密集的函数
- InputManagerService.cpp 的 `getReaderConfiguration`（43 次变更）和 `nativeInjectInputEvent`（25 次）也频繁修改参数

这反映了 Android 输入系统在 2011-2012 年间参数接口尚未稳定，到 2013 年后参数变化显著减少。

## 6. 总结

### 6.1 演化叙事

InputFlinger 在 `frameworks/base` 中经历了完整的生命周期：2011 年从零构建 → 2012 年功能完善 → 2013 年模块化迁移 → 2014
年整体迁出到独立仓库。留在 `frameworks/base` 的 JNI 桥接层（InputManagerService.cpp）作为唯一持续活跃的文件，从 2012 到 2024
年持续演化，是连接 Java 框架层和 Native 输入系统的关键纽带。

### 6.2 关键发现

1. **两次迁移，两种性质**：2013 年是仓库内目录重组（FILE_DIR_CHANGE），2014 年是跨仓库迁出（ADD+DELETE）。前者不改变代码，后者涉及标识符重建。
2. **InputReader 是最活跃的文件**：无论是 ADD、DELETE 还是参数变更，InputReader 始终排在首位，反映了输入处理逻辑的复杂性和持续演进。
3. **接口稳定化**：重命名事件从 2011 年的 38 次降到 2012 年的 10 次，体现了接口快速收敛。2020 年的 19 次重命名则对应
   Android 大版本 API 变更。
4. **JNI 层的生命力**：InputManagerService.cpp 持续活跃 13 年，2024 年仍有 56 个 commit，说明 Android 框架与 Native
   输入系统的接口仍在持续演进。
