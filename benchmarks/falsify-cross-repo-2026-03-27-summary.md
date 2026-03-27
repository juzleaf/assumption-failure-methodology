# Falsify 跨仓库验证 + CR→Falsify 串行 Benchmark 总报告

> 日期：2026-03-27
> 方法：AFM Falsify v1.1 盲测 + CR→Falsify 串行管线
> 条件：空 prompt 盲测（3 仓库），CR findings 作为种子（1 仓库）
> 执行：Opus subagent，独立 session
>
> **⚠️ 本报告数据已被逐条交叉验证补充。详见 `falsify-cross-validation-2026-03-27.md`。**
> 关键修正：21 PROVEN 中仅 3 个（14%）是四方法独家发现。Falsify 核心价值是证明质量和噪音过滤，不是独家检测能力。

---

## 一、实验矩阵

| # | 仓库 | 语言 | 实验类型 | 报告文件 |
|---|------|------|----------|----------|
| 1 | 2fauth-worker | TS | Falsify 盲测 | 2fauth-2026-03-27-falsify.md |
| 2 | picolm | C | Falsify 盲测 | picolm-2026-03-27-falsify.md |
| 3 | OpenBSE | Rust | Falsify 盲测 | openbse-2026-03-27-falsify.md |
| 4 | picolm | C | CR→Falsify 串行 | picolm-2026-03-27-cr-falsify.md |

加上此前 zigpty（Zig）的三轮 Falsify 实验，共覆盖 **4 仓库 × 4 语言**。

---

## 二、Falsify 盲测跨仓库数据

### 量化汇总

| 仓库 | 语言 | Seeds | PROVEN | SUSPECTED | HARDENED | UNCLEAR | 证伪率 |
|------|------|-------|--------|-----------|----------|---------|--------|
| 2fauth | TS | 25 | 8 | 11 | 5 | 1 | 32% |
| picolm | C | 23 | 6 | 4 | 10 | 3 | 26% |
| OpenBSE | Rust | 16 | 2 | 9 | 4 | 1 | 13% |
| zigpty (v3) | Zig | 18 | 5 | 6 | 5 | 3 | 28% |
| **合计** | | **82** | **21** | **30** | **24** | **8** | **26%** |
| **平均** | | **20.5** | **5.3** | **7.5** | **6.0** | **2.0** | **25%** |

### 与 Round 3 其他方法对比

| 仓库 | CR findings | v3.1 V | v3.1 H | Radiate V | Falsify PROVEN | Falsify HARDENED |
|------|-------------|--------|--------|-----------|----------------|------------------|
| 2fauth | 24 | 14 | 4 | 10 | 8 | 5 |
| picolm | 21 | 8 | 8 | 12 | 6 | 10 |
| OpenBSE | 21 | 7 | 9 | 11 | 2 | 4 |
| zigpty | 25 | 13 | 44 | 11 | 5 | 5 |
| **合计** | **91** | **42** | **65** | **44** | **21** | **24** |

---

## 三、Falsify 跨仓库核心发现

### 发现 1：证伪率与项目类型相关

| 仓库 | 证伪率 | 项目特征 |
|------|--------|----------|
| 2fauth | 32% | Web 应用，多路径认证，配置驱动 |
| zigpty | 28% | 系统库，跨平台，并发 |
| picolm | 26% | 嵌入式 C，输入解析，内存管理 |
| OpenBSE | 13% | 领域模拟，物理公式，架构问题 |

**Web 应用和系统库更容易构造具体证明**（有明确的输入→输出路径）。物理模拟引擎的问题更多是架构层面（缺失迭代求解器），难以构造单一 failing input → 更多 SUSPECTED。

### 发现 2：PROVEN 发现类型分布

| 类型 | 数量 | 比例 | 典型例子 |
|------|------|------|----------|
| 输入验证缺失 | 7 | 33% | GGUF n_dims>4 溢出、reader_t 无边界检查 |
| 认证/授权绕过 | 5 | 24% | WebAuthn 跳过白名单、CF-Connecting-IP 伪造 |
| 逻辑不一致 | 4 | 19% | 速率限制路径不匹配、export 密码长度覆盖 |
| 并发/状态问题 | 3 | 14% | flow control ===、WriteQueue 丢数据 |
| 架构缺口 | 2 | 10% | 收敛迭代死代码、锅炉流量守恒违反 |

**Falsify 在"输入→崩溃"链条清晰的问题上最强。** 认证绕过类也强（因为可以构造具体请求）。架构类问题偏向 SUSPECTED。

### 发现 3：HARDENED 有独立价值

24 个 HARDENED 横跨 4 仓库。典型：
- picolm fp16 subnormal 循环实际被 10-bit mantissa 天然限制
- OpenBSE fan heat 公式与 EnergyPlus 一致
- 2fauth CSRF double-submit 正确实现
- zigpty arena allocator fork 后由 exec 替换

**HARDENED = "经受攻击的确认"**，比 v3.1 的 HOLDS 更强（HOLDS 只是推理判定，HARDENED 是主动攻击后幸存）。

### 发现 4：自我纠错行为跨仓库稳定

| 仓库 | 自我纠错实例 |
|------|-------------|
| zigpty v3 | arena allocator：写到一半推翻 → HARDENED |
| picolm | fp16 loop：初判 VULNERABLE → 攻击后承认"bounded by 10-bit" → HARDENED |
| OpenBSE | 图算法：初判 VULNERABLE → 发现"代码正确注释错误" → HARDENED |

**PROVEN 标准的严格性是自我纠错的触发器。** 当 agent 被要求"给出具体证明或降级"时，它会诚实降级而非编造证明。这是 Falsify 的核心设计成功。

---

## 四、CR→Falsify 串行管线分析

### picolm 三路对比

| 维度 | CR 独立 | Falsify 盲测 | CR→Falsify 串行 |
|------|---------|-------------|-----------------|
| 总 findings/seeds | 21 | 23 | 24 (21+3 新) |
| 最高确信发现 | 1C + 4H | 6 PROVEN | **10 PROVEN** |
| 噪音过滤 | 无 | 10 HARDENED | **10 HARDENED** |
| 新发现 | — | — | 3 个 CR 漏掉的 |
| Tokens | 83K | ~98K | ~86K |

### 串行管线的三个价值

**价值 1：精度提升（最大价值）**
- CR 21 findings 中 10 个被 PROVEN（48%），10 个被 HARDENED（48%），只有 4 个 SUSPECTED
- 从"21 个可能有问题"→"10 个确认有问题 + 10 个确认安全"
- **噪音减半，信号翻倍**

**价值 2：深度增量**
- CR→Falsify 的 10 PROVEN > Falsify 盲测的 6 PROVEN（+67%）
- CR 的广度种子让 Falsify 探索到盲测没覆盖的区域（如 grammar.c 的 in_string masking bug）
- 3 个升级（CR 说 Medium/Low → Falsify 证明更严重）
- 5 个降级（CR 说 Critical/High → Falsify 无法证明）

**价值 3：邻接发现**
- 串行过程中发现 3 个新 seed（CR 和盲测 Falsify 都没找到）
- 深度分析一个区域时，自然发现相邻的问题

### 管线效率

| 管线 | PROVEN | Tokens | PROVEN/100K tokens |
|------|--------|--------|--------------------|
| CR 独立 | ~1 Critical | 83K | ~1.2 |
| Falsify 盲测 | 6 | ~98K | ~6.1 |
| CR→Falsify 串行 | 10 | ~86K+83K=169K | **~5.9** |
| v3.1 独立 | 8 V (无证明) | 110K | — |

串行总 token 更高（两步加起来 ~169K），但 PROVEN 数量最多。如果只看 Falsify 步骤本身（~86K），效率和盲测相当但产出更多。

---

## 五、方法论角色定位（更新）

| 方法 | 角色 | 输出 | 最佳场景 |
|------|------|------|----------|
| CR+debug | 广度扫描器 | ~22 findings，无证明 | 快速了解全貌 |
| AFM v3.1 | 深度引擎 | V/H 判定 + 根因 + WHY | 理解为什么会出问题 |
| AFM Radiate | Pattern 探测器 | 辐射链 + 系统 Pattern | 找同类问题 |
| **AFM Falsify** | **证明引擎** | PROVEN/HARDENED + 具体证据 | 需要可验证的 bug report |
| **CR→Falsify** | **精度管线** | 噪音过滤 + 深度证明 | 生产级审计（最全面） |

### 推荐工作流

1. **快速扫描**：CR（~83K tokens，~22 findings）
2. **深度理解**：CR → v3.1（CR 做广度，v3.1 做假设链）
3. **需要证据**：CR → Falsify（CR 做广度，Falsify 做证明）
4. **最全面**：CR + v3.1 + Falsify 三路并行 → 交叉对比

---

## 六、关键洞察（给开源文档用）

### 一句话
> Falsify 把"可能有问题"变成"这是证明"或"这经受住了攻击"。CR→Falsify 串行管线是噪音杀手：48% 噪音被过滤，同时信号被加固为具体证据。

### 数据支撑
- 4 仓库 × 4 语言盲测，82 seeds → 21 PROVEN（26%）+ 24 HARDENED
- CR→Falsify 串行比 Falsify 盲测多 67% PROVEN（10 vs 6）
- 自我纠错行为在 3/4 仓库出现，由 PROVEN 标准严格性触发
- PROVEN 发现类型：输入验证（33%）> 认证绕过（24%）> 逻辑不一致（19%）
- 证伪率与项目类型相关：Web 应用 32% > 系统库 28% > 嵌入式 26% > 物理模拟 13%

### Falsify 的独特贡献
1. **四级确信体系** — 不是所有发现都一样可信，PROVEN ≠ SUSPECTED
2. **攻防验证** — 每个判定都经过主动攻击
3. **噪音过滤** — HARDENED 把"看起来像 bug"变成"确认不是 bug"
4. **自我纠错** — 严格标准触发 agent 自己推翻错误判定
5. **CR 种子加速** — 用 CR 的广度为 Falsify 的深度导航

---

## 七、局限性

1. **N=1**：每个仓库只跑了一次 Falsify，有随机性（zigpty 三轮实验证明各轮有独家发现）
2. **Subagent 执行**：Falsify 设计为 adversary subagent，但本次由单 agent 执行（简化版）
3. **PROVEN 标准主观性**：什么算"可运行证明"在静态分析中有灰度
4. **物理模拟短板**：OpenBSE 只有 2 PROVEN，领域特定问题难以构造 failing input
5. **串行管线只在 picolm 测试**：需要更多仓库验证 CR→Falsify 增量的稳定性
