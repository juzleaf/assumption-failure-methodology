# Round 3 Benchmark 总报告：4 仓库 × 3 方法 = 12 Runs

> 日期：2026-03-26
> 方法：CR+debug vs AFM v3.1 vs AFM Radiate v3.2
> 条件：空 prompt 盲测，无引导，Opus subagent
> 所有原始报告在 `docs/benchmarks/` 目录

---

## 一、仓库矩阵

| # | 仓库 | 语言 | 规模 | 领域 | 选择理由 |
|---|------|------|------|------|----------|
| 1 | nap0o/2fauth-worker | TS+Vue | 18K 行 | 安全（2FA 服务） | Cloudflare Worker + OAuth + 加密 |
| 2 | RightNow-AI/picolm | C | 2.5K 行 | AI（嵌入式 LLM） | 纯 C、GGUF 解析、内存安全 |
| 3 | bbrannon4/OpenBSE | Rust | ~15K 行 | 工程（建筑能耗） | 领域密集、对标 EnergyPlus |
| 4 | pithings/zigpty | Zig+TS | 3.7K 行 | 系统（PTY 库） | 跨语言+跨平台+N-API |

覆盖：4 种语言（TS/C/Rust/Zig）、4 个领域、规模从 2.5K 到 18K。

---

## 二、全局数据汇总

### 量化结果

| 仓库 | CR findings | v3.1 Seeds | v3.1 V | v3.1 H | Radiate Seeds | Radiate V | Radiate H |
|------|-------------|------------|--------|--------|---------------|-----------|-----------|
| 2fauth | 24 | 22 | 14 | 4 | 21 | 10 | 8 |
| picolm | 21 | 23 | 8 | 8 | 25 | 12 | 8 |
| OpenBSE | 21 | 19 | 7 | 9 | 13 | 11 | 2 |
| zigpty | 25 | 58 | 13 | 44 | 23 | 11 | 10 |
| **合计** | **91** | **122** | **42** | **65** | **82** | **44** | **28** |
| **平均** | **22.8** | **30.5** | **10.5** | **16.3** | **20.5** | **11.0** | **7.0** |

### 效率数据

| 仓库 | CR tokens | v3.1 tokens | Radiate tokens | CR calls | v3.1 calls | Radiate calls |
|------|-----------|-------------|----------------|----------|------------|---------------|
| 2fauth | 138K | 136K | 140K | 49 | 49 | 66 |
| picolm | 83K | 110K | 104K | 26 | 42 | — |
| OpenBSE | 68K | 154K | 79K | 50 | 131 | 74 |
| zigpty | 83K | 103K | 97K | 25 | 27 | 39 |
| **合计** | **372K** | **503K** | **420K** | **150** | **249** | **179+** |

---

## 三、核心发现

### 发现 1：CR 广度稳定，不受语言/规模影响

CR findings 在 21-25 之间波动极小（σ=1.9）。无论目标是 2.5K 行 C 还是 18K 行 TS，CR 都能稳定产出 ~22 个 finding。这说明 **CR 的广度是方法本身的属性，不是目标决定的**。

### 发现 2：v3.1 VULNERABLE 数与"假设密度"正相关

| 仓库 | v3.1 V | 特征 |
|------|--------|------|
| 2fauth | 14 | 多层 auth+OAuth+加密，假设链长 |
| zigpty | 13 | N-API 跨线程+跨平台，假设密集 |
| picolm | 8 | 纯 C 单线程，假设较直接 |
| OpenBSE | 7 | E+ 惯例多，很多"可疑但正确" |

v3.1 在"假设密度高"的项目上产出更多 VULNERABLE——2fauth（14V）和 zigpty（13V）都是跨边界（语言/平台/线程）项目，每个边界都是假设来源。

### 发现 3：Radiate VULNERABLE 最稳定（10-12 range）

Radiate V 在 10-12 之间（σ=0.8），是三种方法中最稳定的。原因：RADIATE 操作的"从一个 pattern 搜索同类"天然产生一致的输出量——pattern 数量有限，每个 pattern 辐射出几个实例。

### 发现 4：v3.1 HOLDS 是独立价值

v3.1 的 65 个 HOLDS（平均 16.3/仓库）不是"没找到问题"，而是**验证了防御为什么有效**。

典型 HOLDS：
- OpenBSE fan heat 公式看起来反直觉但匹配 E+（避免误报）
- zigpty arena allocator 在 fork 后安全（exec 替换地址空间）
- 2fauth CSRF double-submit 正确实现（告诉开发者"别改这个"）
- picolm float time comparison 在受限 timestep 下精度足够

**对 AI coder 来说，"什么不该改"和"什么该修"同样重要。**

### 发现 5：三方独家发现各有不可替代的类型

| 类型 | 典型发现 | 来源 |
|------|----------|------|
| 平台/领域知识 | SIGUSR 信号常量硬编码、HFG 物理常量不一致、CP_WATER 差异 | CR 独家 |
| 设计 vs 实现不一致 | README 声称支持 Q5_K 但无实现、类型签名说谎 | v3.1 独家 |
| 假设链级联 | DX coil BF 懒初始化→w≈0→psychrometric 爆炸 | v3.1 独家 |
| 系统性 Pattern | CLI vs Library gap、Unix/Windows 不对称 | Radiate 独家 |
| 辐射链 | "trusted input" → 20+ 未保护 read 调用点 | Radiate 独家 |
| 根因归类 | 5 个 Root Cause 归组覆盖 19 个 seeds | v3.1 独家 |

**没有一种方法能覆盖所有类型。三方搭配是最优策略。**

### 发现 6：语言切换无障碍

方法论在 TS→C→Rust→Zig 四种语言间表现一致。没有任何一种语言让 AFM 失效。这验证了 AFM 的核心定位：**它是 understanding protocol，不依赖语言特性。**

---

## 四、方法论角色定位（Round 3 确认）

| 方法 | 角色 | 强项 | 弱项 |
|------|------|------|------|
| CR+debug | **广度扫描器** | 稳定覆盖 ~22 finding，领域知识丰富 | 每个 finding 独立，无根因归并 |
| AFM v3.1 | **深度引擎** | 假设链分析、HOLDS 验证、根因归类 | Token 消耗最高，播种受限 |
| AFM Radiate | **Pattern 探测器** | 稳定的 V 输出、辐射链、系统性 Pattern | HOLDS 较少，验证深度不如 v3.1 |

### 推荐使用策略

**独立使用**：
- 快速扫描 → CR+debug
- 深度理解 → AFM v3.1
- Pattern 发现 → AFM Radiate

**搭配使用（推荐）**：
- **CR → AFM v3.1**：CR 做广度播种，v3.1 对 CR 的 findings 做假设链深挖
- **AFM v3.1 → Strike**：v3.1 产出 V/HOLDS，Strike 做攻击验证
- **三路并行 → 交叉对比**：本次 benchmark 的方式，最全面但成本最高

---

## 五、与历史 Benchmark 对比

| 维度 | Round 2（6 仓库，2-way） | Round 3（4 仓库，3-way） |
|------|--------------------------|--------------------------|
| 方法 | CR vs AFM | CR vs AFM v3.1 vs AFM Radiate |
| Prompt | 有引导（已作废） | 空 prompt 盲测 |
| 语言 | Java/PHP/Ruby/Rust/Python/Go | TS/C/Rust/Zig |
| 核心结论 | CR 赢广度 AFM 赢深度 | **确认 + 细化**：三方互补各有不可替代的发现类型 |
| 数据可信度 | ❌ 被 prompt 引导污染 | ✅ 干净的空 prompt 实验 |

**Round 3 是第一轮完全干净的 benchmark 数据。** Round 2 的修复质量分析（CR=补丁 AFM=根治）仍有参考价值，但量化对比数据已作废。

---

## 六、关键洞察（给开源文档用）

### 一句话总结
> CR 告诉你"哪里有问题"，AFM 告诉你"为什么会有问题以及还有哪里会有同样的问题"。

### 数据支撑
- 12 runs × 4 仓库 × 4 语言，空 prompt 盲测
- CR 平均 22.8 findings（广度稳定）
- AFM v3.1 平均 10.5 VULNERABLE + 16.3 HOLDS（深度+验证）
- AFM Radiate 平均 11.0 VULNERABLE + 7.0 HOLDS（Pattern+辐射）
- 三方共识 findings 100% TRUE（历史抽检 15/15 确认）
- 每种方法都有不可替代的独家发现类型

### AFM 的独特贡献
1. **假设链分析**——不只报告 bug，追溯"为什么开发者会这样写"
2. **HOLDS 验证**——告诉你"什么不该改"
3. **根因归类**——把 N 个 finding 归并为 M 个 root cause（M << N）
4. **辐射搜索**——从一个 pattern 找到同类所有实例
5. **AI coder 友好**——输出格式直接可用于指导修复

---

## 七、AFM Falsify 实验（2026-03-27 追加）

Round 3 完成后，设计并实验了 **AFM Falsify**——在 v3.1 基础上增加 FALSIFY 操作（对每个 verdict 进行攻击性验证）。在 zigpty 上跑了三轮递进测试。

### 核心数据

| 轮次 | 条件 | Seeds | PROVEN | SUSPECTED | HARDENED | UNCLEAR | Tokens |
|------|------|-------|--------|-----------|----------|---------|--------|
| v1 | 有上下文 | 24 | 8 | 5 | 10 | 0 | 104K |
| v2 | 盲测，旧标准 | 28 | 3 | 7 | 15 | 3 | 135K |
| v3 | 盲测，新标准 | 18 | **5** | 6 | 5 | 3 | 106K |

### 三个关键发现

1. **上下文污染让 PROVEN 注水**：v1 的 8 PROVEN 中 3 个被 v2 降级为 SUSPECTED（UAF 有 `_closed` guard、privilege drop 是文档推理不是实际证明）
2. **新 PROVEN 标准触发自我纠错**：v3 分析 arena allocator 时写到一半推翻自己→重分类为 HARDENED。关键改动："Reasoning alone is never PROVEN"
3. **盲测的独家发现不重叠**：v2 找到零维度穿透、v3 找到 flow control `===` 和 WriteQueue 静默丢数据——三轮各有独家

### 四级 Verdict 与 V/H/U 的关系

| 新 Verdict | 对应旧系统 | 新增信息 |
|-----------|----------|----------|
| PROVEN | V + 可运行证明 | 可直接写 bug report / 测试用例 |
| SUSPECTED | V 但缺具体证明 | 附"如何在运行时验证"指引 |
| HARDENED | H + 经过攻击测试 | 附"我试了什么攻击，都失败了" |
| UNCLEAR | U | 无变化 |

### 方法论角色更新

| 方法 | 角色 | 强项 | 弱项 |
|------|------|------|------|
| CR+debug | 广度扫描器 | 稳定覆盖 ~22 finding | 无根因归并 |
| AFM v3.1 | 深度引擎 | 假设链 + HOLDS + 根因 | Token 高 |
| AFM Radiate | Pattern 探测器 | 辐射链 + 系统性 Pattern | HOLDS 较少 |
| **AFM Falsify** | **证明引擎** | **PROVEN 有可运行证据、HARDENED 有攻防记录** | **覆盖面最窄（18 seeds）** |

> 详细报告：`zigpty-2026-03-27-falsify-experiment.md`

---

## 八、未来方向

- [ ] CR → AFM 串行 benchmark（验证搭配增量）
- [x] ~~AFM → Strike 管线实验~~ → 演化为 AFM Falsify（自带 adversarial 验证）
- [ ] AFM Falsify 跨仓库测试（当前仅 zigpty 一个仓库）
- [ ] 更大规模仓库测试（10K+ 行）
- [ ] 非安全领域比例统计（验证 AFM 不是安全工具）
- [ ] 社区 seed checklist（补广度短板）
- [ ] 多次 Falsify 合并实验（解决单次随机性）

---

## 八、文件索引

### 独立报告（12 份）
| 仓库 | CR | v3.1 | Radiate |
|------|-----|------|---------|
| 2fauth | `2fauth-2026-03-26-cr-debug.md` | `2fauth-2026-03-26-afm-v31.md` | `2fauth-2026-03-26-afm-radiate.md` |
| picolm | `picolm-2026-03-26-cr-debug.md` | `picolm-2026-03-26-afm-v31.md` | `picolm-2026-03-26-afm-radiate.md` |
| OpenBSE | `openbse-2026-03-26-cr-debug.md` | `openbse-2026-03-26-afm-v31.md` | `openbse-2026-03-26-afm-radiate.md` |
| zigpty | `zigpty-2026-03-26-cr-debug.md` | `zigpty-2026-03-26-afm-v31.md` | `zigpty-2026-03-26-afm-radiate.md` |

### 对比报告（4 份）
| 仓库 | 文件 |
|------|------|
| 2fauth | `2fauth-2026-03-26-comparison.md` |
| picolm | `picolm-2026-03-26-comparison.md` |
| OpenBSE | `openbse-2026-03-26-comparison.md` |
| zigpty | `zigpty-2026-03-26-comparison.md` |

### Falsify 实验（2026-03-27 追加）
| 文件 | 说明 |
|------|------|
| `zigpty-2026-03-27-falsify-experiment.md` | 三轮递进测试对比报告 |

### 总报告
- 本文件: `round3-2026-03-26-summary.md`

### /tmp 数据持久化状态
- 12 份 Round 3 原始报告已复制到 `docs/benchmarks/`
- 3 份 Falsify 原始报告在 `/tmp/afm-benchmark-round3/zigpty-falsify{,-v2,-v3}/`
- /tmp 源码可重新 clone
