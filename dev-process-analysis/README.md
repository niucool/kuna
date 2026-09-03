# kuna 开发过程分析：人工参与 vs Agent 生成，及重现计划

> 分析日期：2026-09-02。数据来源：`docs/`（agents.md / history.md / phases.md / improvement-pipeline.md / re-pipeline.md / decbench-loop.md / devcontainer.md）+ git 历史（共 1754 个 commit，2026-06-05 起）。

## 一、项目是什么

kuna 是一个 **agent-first 的 Rust 反编译器**：反编译引擎 + SLEIGH 编译器，围绕显式阶段模型组织，把阶段内决策点暴露为可翻转的 option（`--option NAME VALUE`）——这个 LLM 控制面本身就是产品。它源于 Ghidra 反编译器的 Rust 移植，之后独立演化出自己的默认值和特性。

阶段模型：P0（知识/配置平面，正交）+ P1–P9 有序阶段（分区 → 提升流图 → SSA 数据流 → 调用/原型 → 类型 → 变量 → 区域 → 结构化 AST → C 输出），其中 P3–P6 是 Band B 互不动点带。源码目录 `decompiler/crates/kuna-decomp/src/p0_knowledge/ … p9_emit/` 按阶段物理命名。规范算法在 `docs/spec/`，一屏图在 `docs/phases.md`。

## 二、实现时间线（git 历史证据）

| 时期 | 事件 |
|---|---|
| 2026-06-05（首日 10 个 commit） | vendor Ghidra C++ 反编译器（~196k LOC）+ 39 个 SLEIGH 处理器模块；搭构建驱动、sync_upstream 工具、写 CLAUDE.md |
| 2026-06-06 → 06-08（C++ 改写期） | 从 Ghidra/angr/Reko 推导出阶段模型；修复 37 个公开 Ghidra issue；从 angr 移植第一个特性 |
| 2026-06-09（1cf56a68） | **自主 angr 特性流水线启动**，首个自主 PR 当天合入（PR #1: `stackguard`） |
| 2026-06-10 → 06-19（Rust 移植，两周，~$8k） | 6-crate workspace；91 移植项 + 91 配对验证项；wave gate W0–W11；675/675 达成平价 |
| 2026-06-20 | SLEIGH 编译器移植完成（148/148 spec 内容等价）；**C++ oracle 树删除**；Python CLI → 单一 Rust `kuna` 二进制 |
| 2026-06-22 → 约 5 周 | 分析层移植：Ghidra Java 分析器套件重建为 `kuna-analysis`（62 个增量） |
| 2026-07-04 起 | decbench 基准战役、Ghidra 集成、WASM 浏览器前端（kuna.noelo.org） |
| 2026-07 月末起 | **re-pipeline 全自主自合并车道**（#359/#366）：agent 用 kuna 逆向工程、记录摩擦、自动修复合入 |

Git 月度 commit 分布：2026-06：1428 · 2026-07：195 · 2026-08：126 · 2026-09：5。

## 三、Rust 移植的验证方法论（项目核心）

- **差分 oracle**：C++ 树全程字节不动作为 oracle，所有正确性声明都是差分验证（`--engine {cpp,rust}` 开关跑同一套 675 断言 XML datatests），绝自评。
- **porter/verifier 分离**：每个移植项配结构独立的对抗验证者 agent——只拿到 C++ 源（pinned blob sha）+ Rust diff + 测试输出，**拿不到移植者的推理**；必须完成猎杀清单（符号数宽度、整数提升、comparator 全序、每次循环的迭代顺序 provenance、do-while/lower_bound 边界、erase-while-iterating 等价、异常→Result 部分状态平价）并写 ≥3 个对抗测试。verdict：ACCEPT / ACCEPT-WITH-LOSSES（每条 divergence 入 losses ledger）/ REJECT（3 个 REJECT 阻塞人工决策）。
- **分层差分证据**（细→粗）：opbehavior/float/comparators/XML 金标向量；每指令 SLEIGH lift-diff；阶段边界快照 B0–B5 与 C++ 逐字节相等；207 个上游单元测试 1:1 转写；675 datatest 断言 + **单调性强制**（每 wave 做通过集 diff，已通过断言永不回退）。
- **确定性结构强制**：BTreeMap/BTreeSet only（HashMap 被 clippy disallowed_types 全 workspace 禁用）、稳定排序、强制包装 helper、显式宽度。
- **附带产出**：发现上游 Ghidra 6 个潜在 bug（UB-1..UB-5 系列）。

## 四、人工参与 vs Agent 生成的实际分工

### 作者署名统计

- **mahaloz**：1447 commits（项目发起人，人类）——全部立项期、C++ 改写期、Rust 移植期的 wave 集成提交
- **Zion Leonahenahe Basque**：299 commits，其中 **117 个带 `[AUTOMATED]`**（re-pipeline builder agent 在自合并车道落的特性/修复 PR，如 DIV-81…DIV-99）
- **kuna-verifier**：3 commits（移植期对抗验证者 agent 的测试提交）
- **Claude Fable 5 / phix33 / Shreethaar / Jordan**：6 commits（零星 agent/协作者修复）

### 分工矩阵

| 阶段 | 人工（mahaloz 的角色） | Agent（生成的内容） |
|---|---|---|
| 1. 立项/第一周（06-05→06-08） | 选定 Ghidra 起点与 vendor 决策；写 CLAUDE.md/agents.md 规约；审计 18 个 PORT_PROBLEMS；批准 GH issue 修复方向 | vendor 提交本体、构建驱动、sync_upstream 工具、阶段模型研究文档、37 个 GH issue 修复代码 |
| 2. Rust 移植（06-10→06-20） | **定义方法论**：wave 划分（W0–W11）、91+91 项清单、每项 pin 到 C++ blob sha、7 个 ADR（slotmap vs Rc、BTreeMap 强制、u64 强制包装、Result 镜像、declarative 调度表、phases.toml 代码生成、typed P0 store）、验收标准 | **每个移植项的代码本体**（line-faithful 转写）、每个验证项的 ≥3 个对抗测试、losses ledger ~250 条、W10 parity grind 的根因分析 |
| 3. 特性流水线（06-09 起） | 写 improvement-pipeline runbook、设计 state claim/lease 机制、**审每个 PR**（人审车道：nothing lands on main without human-reviewed PR） | 从 PR #1 起几乎全部特性 PR：`claude -p` worker 会话在隔离 worktree 完成"找机会→实现→验证→开 PR"全循环 |
| 4. re-pipeline 自合并车道（07 月末起） | 设计双臂谓词门、captain 状态机、bwrap 防泄漏沙箱、merge lease 协议、三层碰撞避免；跑对抗审查（4 个独立 agent 审查实现，32 确认/17 驳回，全部修复） | **117 个 `[AUTOMATED]` PR 的全部代码**、tester 的 crackme 摩擦观察、captain 的状态转移与 proposal 审批 |

**一句话总结**：人工定义"做什么、怎么算对、边界在哪"（规约、ADR、验收标准、PR 审查、谓词门设计）；agent 生成"所有代码和测试"。人工不写算法实现；agent 不做方向性决策——方向要么被 runbook 预先写死，要么走人审 PR，要么由 captain 状态机 + 可执行谓词门机械执行。

## 五、三条流水线（agent 持续开发机制）

### 1. improvement-pipeline（人审 PR 车道）

问题："kuna 的 C 输出是否比参考（angr）差？" 循环：Find（三条路：angr 测试集 sweep / decbench 真实语料 GED 挖掘 / 直接对比）→ Investigate（复现→确认参考真的更好→归因到一个阶段决策）→ Implement（一个 PR 一个 option 门控特性 `kuna_<slug>.rs`，默认关）→ Verify & Ship（消融定默认：0/675 断言变化且速度 ≤5% 才能默认开并记 DIV 行）。
无头集群：`PIPELINE_WORKERS=N tools/pipeline/run.sh`，每个 worker 是隔离 git worktree 里的 `claude -p` 会话，原子认领（`state claim`），四个门 + `open_pr.sh` 开真实 before/after PR。

### 2. decbench 车道

同一特性工厂，信号源换成真实二进制语料（94,575 函数 / 803 二进制），GED（CFG 图编辑距离 vs Joern 解析的原始源码）。现状：kuna GED 完美率 35.93%，与 IDA（36.06%）持平，byte_match 第一。文档强调约一半 GED 差距是评分伪影，triage 清单强制。

### 3. re-pipeline（全自主自合并车道）

问题："agent 能否用 kuna 逆向工程二进制？" codex tester 在 bwrap 沙箱里用 kuna 解 250 个 crackme，记录每处摩擦。关键设计：
- **LLM 叙述永不能直达 builder**：每条观察带两个可执行谓词——probe（断言当前坏行为）PASS + acceptance（断言期望行为）FAIL 才算真实需求；acceptance 翻 PASS 后原样升级为 `tests/cli/` 永久回归测试。
- **三种 builder 轨道**：tooling（不改输出，自由并行）/ quality（改输出，全仪式，全局同时最多一个 option-adding builder 在飞）/ perf。
- **captain**（Claude 会话）执行受限守卫状态转移（每次 tick 一个，非法转移 exit 2），并审批大特性 `[PROPOSAL]`。
- **自合并**：merge lease 串行 → rebase → `counters --fix` 重新推导共享计数器 → 四门全绿 → acceptance 翻 PASS → squash-merge with `full-ci` label → 重启 tester 攻击新表面；旧需求回退标记 regressed 最高优先。

### 横切规约（用真实事故换来的）

- 永不重钉 `docs/baseline.json` 吸收回归（mergecheck 硬拒绝）。
- 永不在 worktree `git stash`（共享 stash 栈丢过工作）、永不 `make specs` 于 worktree、`CARGO_INCREMENTAL=0` 等（每 worktree 20–30 GB）。
- 共享计数器 rebase 后从新鲜采集重推导，永不做算术（静默合并会自动产出错数字）。
- Standing requirement 7：sweep 整个语料而非只看见证用例；requirement 8：refuter 必须靠构建读 diff，不能靠论证。
- 四个提交门：`make test`（675/675）/ `make test-stages` / `make rust-test` / `make check-spec` + `kuna catalog --check`。

## 六、从头重现的计划（人工 + agent 结合）

### 前提认知

重现的不是"mahaloz 的 kuna"，而是"用同样方法论构建自己的 kuna"。C++ oracle 树在 06-20 已删，但 git history 里都在（`9346a1a5` 之前的提交，或按 docs/history.md 的 `GHIDRA_REV` 从上游重新 vendor）。

### 阶段 0 — 人工：环境与 oracle（对应 06-05）

1. 建 repo；检出 pre-removal 的 `decompiler/cpp` 树作为 oracle。
2. 搷 83 文件/675 断言 datatest 语料 + 39 个 SLEIGH spec 模块。
3. 写自己的 AGENTS.md/CLAUDE.md（人工最关键的投资：**规约即方法论**）。
4. 环境：官方 devcontainer（ubuntu:22.04 + Rust 1.90 + 全套交叉编译器）：`docker build -t kuna-dev -f .devcontainer/Dockerfile .devcontainer`。

### 阶段 1 — 人工主导：阶段模型 + 差分基线（对应 06-06）

1. 从 Ghidra/angr/Reko **推导**自己的阶段模型（全项目最重要的人工决策）。
2. 人工写 `--engine {cpp,rust}` 差分 harness 验收标准，agent 实现 harness 本体。
3. 确立三条铁律：675/675 平价、永不重钉 baseline、单调性（通过集不许回退）——这是 agent 后续能安全自主的根。

### 阶段 2 — 人工设计 + agent 执行：Rust 移植（对应 06-10→06-20）

人工：wave 划分（91+91 项，每项 pin blob sha）；写 7 个 ADR；定义验证者协议（只看 C++ 源 + Rust diff + 测试输出；猎杀清单；≥3 对抗测试；三档 verdict）。
Agent：每 wave 内并行 fan-out（每 agent 一个隔离 worktree），串行集成在 wave gate；W10 parity grind 根因（每个未过断言一条 LOSS 或 UB 记录）。
人工逐 wave 验收。成本预期：原版约两周连续 LLM 时间、$8k；重现方法论已知、坑已知，但验证开销是结构性的，别指望数量级下降。

### 阶段 3 — 半自动：特性流水线（对应 06-09 起）

1. 人工写 runbook（或直接复用本仓库 `docs/improvement-pipeline.md`）。
2. 搭 `scripts/pipeline/`（state claim/lease、sweep/select/timeit/prdemo/open_pr）——agent 可写，但**人必须审协议设计**，它是防 agent 出错的根。
3. 两档跑法：交互式（人 + agent 会话手动走全循环，人审 PR；原版 PR #1 的模式）→ 无头集群（`PIPELINE_WORKERS=N tools/pipeline/run.sh`，人看 `status --watch` 和 PR 队列）。

### 阶段 4 — 全自主：re-pipeline 式自合并（可选，最高阶）

**不重造，直接复用本仓库 `scripts/repipe/` + `tools/repipe/`**——它浓缩了 32 个已确认事故的修复（含 8 个 critical：惯性循环、租约死锁、probe 允许表绕过、沙箱漏目录等）。
人工持续角色：`smoke.sh`（L1 免费）/ dashboard / `[PROPOSAL]` 审批（或交 captain）/ `regressed` 处理。
分钱路径：L1 冒烟（0）→ L2 一个 tester（~$2）→ L3 一个 builder（~$20）→ L4 一轮（~$150）。

### 实操建议

1. **顺序别颠倒**：先跑通本仓库现状（`make && make test` 四门全绿）→ 玩它的流水线 → 才谈从头重现。
2. **人工投资集中四处**：AGENTS.md 规约、ADR/验收标准、PR 审查、谓词门设计——这四样决定 agent 产出的质量上限。
3. **从 git 历史抄作业**：`git log --reverse` 按时间序读 commit message（此 repo message 质量极高，几乎每个带证据），比读 docs 更接近真实过程；docs 是事后压缩，commit 是过程本身。

## 附：本机（Windows）注意事项

- 流水线驱动脚本默认 Linux 宿主机（`~/.virtualenvs/kuna` 等）；Windows 上要么用 devcontainer/WSL2 跑流水线，要么手动导出覆盖（`KUNA_PY`、`KUNA_PIPELINE_ANGR_PYTHON`、`KUNA_PIPELINE_ANGR_REPO`、`KUNA_PIPELINE_BIN_ROOT`，见 docs/improvement-pipeline.md §0）。
- re-pipeline 还需要：`bwrap`、已认证的 `gh`、`codex`、`claude`、crackme 数据集 `~/github/kuna-re-dataset`。
