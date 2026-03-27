# AFM Falsify 实验报告：zigpty 三轮对比

> 日期：2026-03-27 | 目标：pithings/zigpty（Zig+TS PTY 库，~3700 行）
> 方法：AFM Falsify v1.0（seed → interrogate → adversarial falsification）
> 实验设计：三轮递进测试——有上下文 → 盲测旧标准 → 盲测新标准

---

## 一、实验目的

Round 3 benchmark（12 runs）完成了 CR / v3.1 / Radiate 三路对比。AFM Falsify 是新 skill——在 v3.1 的 REVERSE/FORWARD 基础上增加 **FALSIFY** 操作：对每个 verdict 进行攻击性验证，用四级判定（PROVEN/SUSPECTED/HARDENED/UNCLEAR）替代三级（V/H/U）。

核心假设：增加 adversarial falsification 步骤能提高 verdict 可信度。

---

## 二、三轮数据

| 指标 | v1（有上下文） | v2（盲测，旧标准） | v3（盲测，新标准） |
|------|--------------|-------------------|-------------------|
| Seeds | 24 | 28 | 18 |
| PROVEN | 8 | 3 | **5** |
| SUSPECTED | 5 | 7 | 6 |
| HARDENED | 10 | 15 | 5 |
| UNCLEAR | 0 | 3 | 3 |
| Tokens | 104K | 135K | 106K |
| Tool calls | 43 | 88 | 53 |
| 耗时 | 524s | 655s | 351s |

### 关键变量

| 轮次 | 给 agent 的额外信息 | PROVEN 标准 | 改动意图 |
|------|---------------------|------------|----------|
| v1 | "Zig+TS, N-API, 跨平台, 多线程" | 原版（concrete evidence of failure） | 基线 |
| v2 | 无（纯空 prompt） | 同 v1 | 消除上下文污染 |
| v3 | 无 | 加硬："你必须能写出可粘贴运行的测试。推理≠PROVEN" | 提高 PROVEN 含金量 |

---

## 三、核心发现

### 发现 1：上下文污染让 PROVEN 注水

v1 的 8 个 PROVEN 里至少 3 个被 v2 降级为 SUSPECTED：

| Finding | v1 | v2 | 降级原因 |
|---------|-----|-----|---------|
| Windows UAF | PROVEN | SUSPECTED | v2 发现 `_closed` JS guard 阻止正常调用，UAF 需绕过 class API |
| Privilege drop | PROVEN | SUSPECTED | "POSIX 文档说会这样"不等于"我构造出了证明" |
| napi_external | PROVEN | SUSPECTED | `_closed` guard 在正常使用下阻止 UAF |

**结论：给 agent 上下文信息会让它倾向于把已知问题判定为 PROVEN，降低诚实度。**

### 发现 2：盲测找到有上下文找不到的东西

| 发现 | 出自 | 未出自 | 类型 |
|------|------|--------|------|
| 零维度穿透 | v2 | v1、v3 | API 契约违反 |
| WindowsPty.process 静态 | v2 | v1、v3 | 接口承诺未兑现 |
| Flow control `===` 永不命中 | v3 | v1、v2 | 逻辑 bug |
| WriteQueue 静默丢数据 | v3 | v1、v2 | 错误处理缺失 |
| macOS environ 并发 forkpty 竞态 | v3 | v1、v2 | 线程安全 |

**v2 和 v3 各自找到了 2-3 个三轮中独家发现，且与 v1 的独家不重叠。**

### 发现 3：新 PROVEN 标准触发自我纠错

v3 报告中出现了自我纠错行为——分析 arena allocator 时写到一半推翻自己：

> "After thorough analysis, this is actually HARDENED — fork creates CoW copies... Moving to HARDENED."

然后用 WriteQueue 数据丢失（一个更扎实的 bug）替补为 PROVEN-5。

v1 和 v2 都没有这种行为。新标准的关键句子：

> "Reasoning alone is never PROVEN... 'POSIX says it would fail' is SUSPECTED. 'Here is the input, here is the code path, here is the wrong output' is PROVEN."

### 发现 4：新标准让 agent 深度优先于广度

v3 用最少 seed（18）产出最多独家 PROVEN（3），用最少时间（351s）和最少 tokens（106K）。更严格的 PROVEN 要求迫使 agent 把精力花在深挖而不是铺面。

### 发现 5：零维度从 PROVEN 降到 SUSPECTED 是正确的

v2 把零维度判为 PROVEN，v3 降为 SUSPECTED。v3 的理由：

> "除法错误发生在 PTY 消费端（xterm.js），不在 zigpty 自身"

这是正确的边界划分。zigpty 传了 0×0 给 kernel，kernel 接受了。崩溃发生在下游。

---

## 四、Falsify vs Round 3 三路方法交叉对比

### 与 CR 对比

| 维度 | CR（25 findings） | Falsify v3（18 seeds） |
|------|-------------------|----------------------|
| 广度 | 25 独立 findings | 18 seeds（更少但更深） |
| 证据层 | severity 分级（C/H/M/L） | PROVEN/SUSPECTED/HARDENED/UNCLEAR |
| 独家发现 | SIGUSR 硬编码、Windows resize 竞态 | Flow control ===、WriteQueue 丢数据、macOS environ 竞态 |
| 验证深度 | 描述性（"可能导致…"） | 攻防性（"我试了X、Y、Z，结果是…"） |

### 与 v3.1 对比

| 维度 | v3.1（58 seeds） | Falsify v3（18 seeds） |
|------|-----------------|----------------------|
| 覆盖面 | 58 seeds（每 64 行一个） | 18 seeds（更选择性） |
| HOLDS/HARDENED | 44 HOLDS（"防御存在"） | 5 HARDENED（"防御经受住了攻击"） |
| 含金量 | V 是推理判定 | PROVEN 是构造验证 |
| 盲点 | UAF、double-close | UAF、double-close |
| Token | 103K | 106K |

**关键区别**：v3.1 的 HOLDS 说"检查了有防御"，Falsify 的 HARDENED 说"我攻击了这个防御，它没被破"。HARDENED 的可信度更高但覆盖面窄很多（5 vs 44）。

### 与 Radiate 对比

| 维度 | Radiate（23 seeds） | Falsify v3（18 seeds） |
|------|-------------------|----------------------|
| 视角 | Pattern → 辐射同类 | 假设 → 攻击验证 |
| 独家 | 生命周期同步缺陷辐射链 | Flow control、WriteQueue、environ 竞态 |
| Token | 97K | 106K |

---

## 五、四级 Verdict 的价值分析

| Verdict | 数量 | 与 V/H/U 的关系 | 新增信息 |
|---------|------|-----------------|----------|
| PROVEN | 5 | 比 V 更硬——有可运行的证明 | 可直接用于 bug report |
| SUSPECTED | 6 | V 但缺具体证明 | 附带"如何在运行时验证"的指引 |
| HARDENED | 5 | H 但经过攻击测试 | 附带"我试了什么攻击，都失败了" |
| UNCLEAR | 3 | U | 无变化 |

**PROVEN 的直接商业价值**：v3 的 5 个 PROVEN 每个都附带了可粘贴的代码片段。这意味着：
- 可以直接写成 GitHub issue
- 可以直接变成测试用例
- 维护者无需复现——proof 已经给了

**HARDENED 的防御价值**：告诉维护者"不要动这段代码，有人攻击过了确认是安全的"。

---

## 六、三轮实验的方法论收获

### 盲测 > 有上下文

上下文让 agent 确认已知问题（confirmation bias），盲测让 agent 发现新问题。以后所有 benchmark 必须盲测。

### PROVEN 标准必须硬

"推理上必然失效"≠"证明失效"。加一句话的标准改动就让 agent 行为从"找到就标 PROVEN"变成"构造不出就降级"。

### 单次 Falsify 不够

三轮各自找到了不同的 PROVEN。说明单次 run 有随机性。如果要最全面的覆盖，需要多次 Falsify 或 Falsify + 其他方法搭配。

### 自我纠错是涌现行为

v3 的 arena → HARDENED 自我纠错不是 skill 明确要求的。是 PROVEN 标准的严格性自然导致 agent 在写到一半时发现"等等，我构造不出证明"然后回头修改。

---

## 七、推荐使用方式

| 场景 | 推荐方法 |
|------|---------|
| 快速扫描，列出问题 | CR+debug |
| 理解假设结构，知道什么不该改 | AFM v3.1 |
| 找系统性 Pattern | AFM Radiate |
| 需要可运行的 proof 给维护者 | **AFM Falsify** |
| 全面覆盖 | CR → Falsify（或三路并行） |

---

## 八、Falsify Skill 更新日志

| 版本 | 改动 | 效果 |
|------|------|------|
| v1.0 原版 | 初始设计 | v1 测试基线 |
| v1.1（当前） | PROVEN 标准加硬：必须能写出可粘贴运行的测试 | v3 测试确认：自我纠错出现、边界案例正确降级、独家发现率提升 |

---

## 九、原始数据位置

| 文件 | 路径 |
|------|------|
| v1 报告 | `/tmp/afm-benchmark-round3/zigpty-falsify/afm-falsify-report.md` |
| v2 报告 | `/tmp/afm-benchmark-round3/zigpty-falsify-v2/afm-falsify-report.md` |
| v3 报告 | `/tmp/afm-benchmark-round3/zigpty-falsify-v3/afm-falsify-report.md` |
| Skill | `SKILL.md (afm-falsify)` |
| 持久化 | `docs/benchmarks/zigpty-2026-03-27-falsify-experiment.md`（本文件） |
