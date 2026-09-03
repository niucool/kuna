# 阶段 2（重现计划）：Rust 全量移植——"为什么是移植而不是改进 C++"（决策篇）

> 对应原版真实历史：2026-06-10（`rust/Cargo.toml` 首提交 `6a095f8a`）→ 2026-06-19 M3 675/675 平价（`bca4ae8d`）→ 2026-06-20 SLEIGH 编译器完成 + C++ 树删除（`b3838e1b` / `9346a1a5`）。本篇是决策论证；wave 协议与 porter/verifier 操作细节见后续阶段 2 执行篇。证据源：`git show 9346a1a5:docs/RUST_PORT.md`（删除 C++ 树时写的总结，含"Why"一节）、`git log` 06-10→06-20、docs/history.md。

## 一、原版自己的说法（RUST_PORT.md "Why" 原文）

> "The C++ engine is excellent but hard to study, instrument, and extend stage-by-stage inside the Ghidra build. A faithful Rust port gives a memory-safe, modular, independently-buildable engine with a first-class stage model and an LLM/human control surface."

四条：**难研究（study）、难插桩（instrument）、难以按阶段扩展（extend stage-by-stage）、被 Ghidra 构建绑架（inside the Ghidra build）**。注意评价是 "excellent"——移植不是嫌弃 C++ 质量，而是 C++ 树作为 **agent-first 产品的载体**不合格。

## 二、为什么这四条对"agent-first"是致命的（核心论证）

kuna 的产品是"LLM 控制面 + 可翻转决策点"，这需要四样 C++ 给不了的东西：

### 1. Agent 能独立构建

C++ 引擎在 Ghidra 的 Java/gradle 构建系统里，agent 会话想构建/测试/迭代它几乎不可能。Rust 的 `cargo build/test` 是单命令、可缓存、跨平台——后来 `tools/pipeline/run.sh` 维持 7 个并发 `claude -p` 会话**各在自己 worktree 里构建测试**，这 methodologies 的物理前提就是构建轻量。**这在 C++/gradle 树上不可能运转。**

### 2. Agent 能安全插桩

内存安全不只是工程洁癖。196k LOC 的 C++ 树有真实 UB（移植中发现了上游 6 个潜在 bug，UB-1..UB-5：`opcode_name[]` 在 `CPUI_MAX` 越界读、`INT64_MIN / -1` 的 SIGFPE、`convertCharRef` 有符号溢出、`rangemap::erase` 悬垂读、`MemoryBank` 页拷贝越界、pcode 关键词表排序违规使 `||`/`abs` 无法词法化）。agent 要做的是**大胆插桩、加断言、跑变体**（`pipeline <variant>` 子查询：克隆函数换一套 action 集跑掉）——在 C++ 里这把 agent 带进 UB 荒野，agent 无法自我判断崩的是它的实验还是引擎本身。Rust 里 panic 可捕获、可定位、可恢复。**agent 信任引擎、引擎不坑 agent——控制面能工作的前提。**

### 3. 确定性可以被语言级强制

"反编译输出是规则应用顺序的函数"——所以需要 `BTreeMap` only、`HashMap` 全 workspace 被 clippy `disallowed_types` 禁止（ADR-0002）、稳定排序、显式宽度 + 强制包装 helper（ADR-0003）。C++ 里这些是**约定**，agent 违反了编译器不拦；Rust 里是编译器错误。给 agent 定的规约若能变成编译器强制，就不需要 review 兜底——这对 re-pipeline 的**自合并车道是存在性的**：builder 自主合入 main 的置信度，一部分来自"这类错误编译器已经替我挡了"。

### 4. 可分阶段切片的代码布局

P1–P9 目录物理命名、`kuna_<slug>.rs` 一文件一决策点、`phases.toml` 声明式注册表（ADR-0006）——"症状→阶段→文件"可导航的物理结构。196k LOC 平铺 C++ + 跨文件继承 + 宏 + 指针别名，LLM 上下文既装不下也读不准；Rust 模块边界让"一个 PR 一个决策点一个文件"成立。

## 三、"改进 C++"不成立的另外三条（旁证）

### 5. 上游是活的，分叉是负担

改进 C++ = 维护 Ghidra fork。Ghidra 持续演进（kuna vendor 12.1.1/`cef869af`，保留 `tools/sync_upstream.py` + `GHIDRA_REV`）。fork 追着上游 rebase 196k LOC 的冲突面，比一次性移植 + 冻结 oracle 的成本曲线更差。移植后上游降级为"参考"，不再是"必须跟的活对象"。

### 6. 差分方法论需要 oracle 是"不动的"

"所有正确性声明都对着一个字节不动的 C++ 参考"——如果同时在改 C++ 本身，oracle 就动了，675/675 差分失去意义（改 C++ 时你对它的信任从哪里来？）。原版解法：**C++ 冻结成纯 oracle（整个移植期间字节不动），Rust 做全部实验，平价达成后 oracle 退役**（`baseline.json` 记录快照）。"改进 C++"让 oracle 和实验对象混为一体，**方法论本身解体**。

### 7. SLEIGH 编译器是隐藏的依赖闭环

`.sla` 只有 C++ `sleigh_opt` 能产出——这是"C++ 树必须存在才能跑管线"的最后理由。所以移植分两段：先反编译器（675/675 平价，06-19），再 SLEIGH 编译器（148/148 内容等价 + 用 Rust 产 `.sla` 重跑全套 675/675 的端到端 backstop，06-20）。第二段完成那一刻，**C++ 树存在的理由归零**——删除（`9346a1a5`）是逻辑必然，不是风格选择。

## 四、成本边的诚实核算

不是免费决定：**两周连续 LLM 时间、约 $8k、91 移植项 + 91 配对验证项 + 18 infra 项（182,926 LOC 范围）**。

关键时间线细节：决策发生在 06-10，而**阶段模型已验证（GH-558 原型成功）、流水线已开动（06-09 PR #1 合入）**——顺序是先证明方法论能工作，才投这笔钱换载体。载体换掉后方法论不变（wave 协议、verifier 协议、675 差分门全从 C++ 时代继承——阶段 1 的 `tests/stages/` 语料、控制面五件套概念全部平移）。移植顺带发现上游 6 个 latent bug——这是"改进 C++"路线永远不会给的附带产出：没人会为一个 fork 义务审计 196k LOC。

## 五、移植范围与"刻意不移植"

- **移植了**：反编译器全套（`kuna-base/num/sleigh/decomp/console/harness`）+ SLEIGH 编译器（`kuna-slacomp`，含 bison/flex → 手写递归下降解析器的既有模式）。
- **刻意不移植**（删除而非重实现）：Ghidra Java 客户端（`ghidra_*`）、`printjava`、`rulecompile`/`unify`、graph/callgraph dump 命令——与差分 oracle 无关或超出范围。
- **唯一化妆性残留**（LOSS-010）：`.sla` 是 4 字节 magic + zlib 流；Rust deflate（flate2/miniz_oxide）选了与 C zlib 不同但同样合法的 DEFLATE 编码——**解压后的元素流逐字节一致**，这是 zlib→flate2 依赖替换的已知代价，故门槛是内容而非原始字节（zlib-rs 可达整文件逐字节一致，但正确性不需要）。

## 六、决策点：重现者的判断清单

在开始 wave 划分之前（阶段 2 执行篇），先回答这个二分：

- **目标是"agent-first 反编译器"** → Rust 移植是必经路（七条论证全部适用）；
- **目标只是"改 Ghidra"** → fork 上游就好，这个项目（控制面、流水线、自合并）全都不用做。

中间态（在 C++ 上加控制面）被原版证据否决：控制面五件套在 C++ 侧**已经建过一遍**（阶段 1 全在 C++ 侧），它们能工作；但"agent 能自主构建 + 安全插桩 + 分片读"这三样卡死了后续的流水线——**所以移植发生在流水线启动之后一天（06-09 → 06-10），不是之前**。这是最重要的证据：原版先用一天证明了 C++ 侧的流水线可行（PR #1 合入），紧接着就换了载体——不是冲动，是流水线试跑暴露了载体的天花板。
