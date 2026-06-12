# AOSP frameworks/native 扫描记录

> 扫描日期：2026-06-09 · 数据库：`tracker_aosp_native_20260609_8c2c6144` · 仓库：`platform_frameworks_native`

## 1. 基本统计

| 表                       | 行数        |
|-------------------------|-----------|
| identifier              | 83,637    |
| identifier_change       | 291,221   |
| identifier_relation     | 59,698    |
| commit                  | 126,370   |
| physical_co_change_pair | 0（未执行预计算） |

## 2. 语言分布

| 语言   | 标识符数   |
|------|--------|
| CPP  | 70,808 |
| JAVA | 8,155  |
| C    | 4,674  |

以 C++ 为主（84.7%），符合 `frameworks/native` 作为 Android Native 层的定位。

## 3. 标识符类型分布

| 类型                     | 数量     |
|------------------------|--------|
| METHOD（成员方法）           | 43,447 |
| TOP_LEVEL_FUNCTION（函数） | 24,588 |
| CONSTRUCTOR（构造方法）      | 4,733  |
| VALUE_MACRO（值宏）        | 4,165  |
| CLASS（类）               | 2,481  |
| STRUCT（结构体）            | 1,382  |
| FUNCTION_MACRO（函数宏）    | 1,316  |
| DESTRUCTOR（析构方法）       | 1,281  |
| INTERFACE（接口）          | 123    |
| ENUM（枚举）               | 121    |

## 4. 扫描阶段耗时

| 阶段            | 耗时            |
|---------------|---------------|
| 预处理           | 19s           |
| Commits 识别与存储 | 19s           |
| 初始提交标识符识别与存储  | 187ms         |
| Commits 扫描持久化 | 13m 58s       |
| 大提交补扫         | 4ms           |
| TrackChain 合成 | 0ms           |
| **总计**        | **约 14m 17s** |

## 5. 与 frameworks/base 扫描对比

| 指标                | frameworks/base (`aosp_cpp`) | frameworks/native (`aosp_native`) |
|-------------------|------------------------------|-----------------------------------|
| identifier        | 1,065,412                    | 83,637                            |
| identifier_change | 3,478,675                    | 291,221                           |
| commit            | 1,064,617                    | 126,370                           |
| CPP 标识符           | 85,207                       | 70,808                            |
| JAVA 标识符          | 965,999                      | 8,155                             |
| 扫描耗时              | ~1h 29m                      | ~14m 17s                          |

native 仓库规模约为 base 的 1/13（标识符数），扫描耗时约 1/6，符合预期——native 仓库更小但 commit 密度相对更高。