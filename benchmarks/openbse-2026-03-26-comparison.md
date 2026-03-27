# OpenBSE 三路盲测对比报告

> 日期：2026-03-26 | 目标：bbrannon4/OpenBSE (Rust)
> 55 个 .rs 文件，8 个 crate，建筑能耗模拟引擎（对标 EnergyPlus）
> 空 prompt 盲测，无引导

## 结果总览

| 维度 | CR+debug | AFM v3.1 | AFM Radiate |
|------|----------|----------|-------------|
| Findings/Seeds | 21 (2C/6H/9M/4L) | 19 (7V/9H/3U) | 13 (11V/2H) |
| Tool calls | 50 | 131 | 74 |
| Tokens | 68K | 154K | 79K |

## 核心 Finding 归属分析

### 三方共识（7 个核心问题）
1. **BDF history 缺失** — `run_with_envelope()` 不调 `update_bdf_history()`，zone 温度历史无法推进
   - CR: F-01 Critical | v3.1: Seed 4 area (grouped) | Radiate: Finding 2 High
2. **Static ControlSignals** — `run_with_envelope()` 每个 timestep 用同一组静态信号
   - CR: F-02 Critical | v3.1: Seed 4 HOLDS (认为是 intentional simple path) | Radiate: Finding 3 High
3. **CP_WATER 不一致** — psychrometrics 4180 vs chiller/water_heater 4186
   - CR: F-04 High | v3.1: 未独立列出 | Radiate: Finding 4 Medium
4. **Graph cycle-breaking 注释与代码矛盾** — comment "most" vs code `min_by_key`
   - CR: F-03 High | v3.1: Seed 5 HOLDS | Radiate: Finding 9 VULNERABLE
5. **AUTOSIZE 容差过宽** — ±1.0 范围
   - CR: F-08 High | v3.1: Seed 6 HOLDS | Radiate: Finding 8 VULNERABLE
6. **No leap year** — 7+ 处硬编码 Feb=28
   - CR: F-06 High | v3.1: Seed 1 VULNERABLE | Radiate: Finding 10 VULNERABLE
7. **Chiller magic clamp -2.0°C** — 温度钳位无能量反馈
   - CR: F-07 (chiller fabricates flow, 相关) | v3.1: Seed 12 VULNERABLE | Radiate: Finding 5 VULNERABLE

### 三方判定分歧（最有趣的部分）

| Finding | CR | v3.1 | Radiate | 分析 |
|---------|-----|------|---------|------|
| Static ControlSignals | Critical | **HOLDS** (intentional) | **VULNERABLE** | v3.1 发现 main.rs 有真实控制循环，认为 `run_with_envelope()` 是简化路径——更深的理解 |
| Graph cycle-breaking | High | **HOLDS** (行为合理) | **VULNERABLE** | v3.1 分析了实际行为发现 min_by_key 是合理策略，只是注释错；Radiate 更关注文档误导风险 |
| AUTOSIZE 容差 | High | **HOLDS** (实际无害) | **VULNERABLE** | v3.1 验证了算术路径确认不会产生误报；Radiate 指出其他组件缺少 guard |

**核心观察**：v3.1 的 9 个 HOLDS 不是"没找到问题"，而是**验证了防御为什么有效**。这在这个仓库上特别有价值——OpenBSE 有很多看起来可疑但实际上是 EnergyPlus 惯例的代码。

### CR 独家发现
- **F-05 HFG 不一致** — heat_recovery 用 2.454e6 vs psychrometrics 2.50094e6（1.9% 差异）
- **F-07 Chiller fabricates 0.001 kg/s flow** — 泵关了还在算
- **F-09 DX coil power overcharge at part load** — 使用 available_cap 而非 actual_delivered
- **F-10 CHW coil SHR 应用到水侧** — 物理上不合理
- **F-12 Pump heat-to-fluid 用 total power** — 应只用废热部分
- **F-13 tsat_fn_press Newton-Raphson 初始猜测差** — 低压下偏离 100°C
- **F-14 Boiler 允许效率 >100%**
- **F-15 Heat recovery 用湿空气质量流量配干空气湿度比**
- **F-20 EPW parser 静默跳过坏行**
- **F-21 Thermostat proportional band 硬编码**

CR 在这个仓库上展现了极强的**领域物理知识**——HFG 不一致、SHR 水侧误用、pump heat 公式、LMTD/crossflow 混用——这些需要暖通空调（HVAC）专业知识才能识别。

### v3.1 独家发现
- **Seed 2 month=0 panic** — YAML 输入 `month: 0` 导致 u32 underflow → panic。追踪了完整路径：YAML → u32 反序列化 → `day_of_year()` → underflow
- **Seed 7 DX coil bypass factor 懒初始化失败** — 当 autosizing 产生 0 容量时，BF 永远不初始化，downstream 产生 w≈0 的极端状态
- **Seed 8 DX coil parallel capacity 路径** — curve-based 和 BF-based 两条容量计算路径假设一致但实际可以分叉
- **Seed 19 Outdoor humidity 硬编码 0.008** — 正确数据就在一个参数之外
- **5 个 Root Cause 归类** + Assumption Radiation Map + Impact Chains + Defense Map

**Meta-patterns 独家**：
- "Reference implementation limitations inherited as design assumptions"（E+ 约束被无条件继承）
- "Lazy initialization assumes preconditions will be met"
- "Output clamping without feedback to input calculations"
- "Placeholder approximations surviving to production"

### Radiate 独家发现
- **Finding 1 Pump heat-to-fluid 公式** — 这里 Radiate 比 CR 更深：不只说"用了 total power"，而是**和 fan.rs 对比**，证明 fan 做对了但 pump 做错了，是同一个开发者的不一致
- **Finding 6 CHW coil 无除湿** — 指出 DX coil 有 ADP 模型但 CHW coil 跳过了，尽管有 `rated_shr` 字段暗示意图
- **Finding 12 Humidifier 忽略蒸汽显热**
- **Finding 13 SetPlantLoad 永远 0** — dead code path 分析

**三个结构性 Pattern（独家价值）**：
1. **CLI vs Library gap** — main.rs 有完整的 predictor-corrector 循环，core library 的 `run_with_envelope()` 本质上不可用
2. **Inconsistent E+ fidelity** — 有些组件忠实复现 E+，有些偷工减料，用户无法判断哪些可信
3. **Hardcoded constants instead of shared references** — cp_water、days_in_months、thermostat band 全是本地常量

## 定性分析

### 这个仓库的独特性

OpenBSE 是 Round 3 中最特殊的目标：
- **领域深度要求极高** — 需要理解热力学、心理学(psychrometrics)、HVAC 控制论、数值方法（BDF/Newton-Raphson/NTU-effectiveness）
- **E+ 对标意识** — 很多代码"看起来可疑"但实际是 E+ 惯例（fan heat、365 天年、autosize sentinel）
- **真实工程软件** — 不是 demo/toy，是有 300+ 测试的严肃项目

### 深度对比

**CR 在这个仓库上最强**：21 个 finding 中大量依赖物理直觉（HFG 差异、SHR 水侧误用、pump 废热分解、boiler 效率上限）。这些是"领域专家式"发现，需要知道建筑能耗模拟的物理公式才能判断对错。

**v3.1 在这个仓库上最深**：不是找到最多 finding，而是每个 finding 都有完整的假设链。特别是 Seed 7（DX coil BF 懒初始化）的 forward chain：BF=0 → ADP=0 → w_out=0 → downstream psychrometric 爆炸。这种级联分析 CR 完全没有。

**Radiate 在这个仓库上最结构化**：13 个 finding（最少），但 11 个是 VULNERABLE（最高比例 85%）。三个 Pattern 概括了整个代码库的系统性问题，其中 "CLI vs Library gap" 是最有实际价值的发现——告诉任何想用 OpenBSE 作为库（而非 CLI）的人：core API 不可用。

### v3.1 的 HOLDS 在这个仓库上价值最高

| HOLDS | 含义 |
|-------|------|
| Static signals (Seed 4) | main.rs 有真实控制循环，这是 intentional simple path |
| Fan heat (Seed 9) | 看起来反直觉但匹配 E+ 公式，验证正确 |
| Float time comparison (Seed 14) | 分析了 timestep 枚举限制，确认 fp 精度足够 |
| Boiler zero flow (Seed 11) | SP-modulated 模式正确处理了独立流量 |
| Heating coil fallback (Seed 10) | 明确注释的过渡期设计 |
| Vec allocation (Seed 18) | Rust 借用检查器的正确 workaround |
| Wind coefficient (Seed 17) | 注释推导看起来错但数值恰好对 |

在一个对标 EnergyPlus 的项目中，**判断什么不是 bug 和判断什么是 bug 同样重要**。v3.1 的 HOLDS 分析避免了误报，这对开发者来说比多找几个 Low finding 更有价值。

### 激进度趋势（Round 3 累计）

| 仓库 | 规模 | CR | v3.1 V | Radiate V |
|------|------|-----|--------|-----------|
| 2fauth (TS) | 18K 行 | 24 | 14 | 10 |
| picolm (C) | 2.5K 行 | 21 | 8 | 12 |
| OpenBSE (Rust) | ~15K 行 | 21 | 7 | 11 |

OpenBSE 延续了趋势：**CR 广度稳定在 ~21，v3.1 最保守（7V），Radiate 居中偏激进（11V）**。

### Token 效率异常

v3.1 在 OpenBSE 上消耗了 **154K tokens + 131 tool calls**——远超其他仓库（2fauth 136K/49calls，picolm 110K/42calls）。原因：这个代码库需要大量交叉引用（8 个 crate，组件间物理耦合），v3.1 的假设验证需要读很多文件。Radiate 79K/74calls 效率高得多——Pattern-level 分析不需要逐文件验证。

## 结论

1. **领域密集型仓库放大 CR 优势**：HVAC 物理知识让 CR 找到了 AFM 无法发现的领域 bug（HFG、SHR、pump heat 公式）
2. **v3.1 的防御验证（HOLDS）在这类仓库上最有价值**：区分"E+ 惯例"和"真 bug"是这个项目的核心难点
3. **Radiate 的 Pattern 分析最有战略价值**："CLI vs Library gap" 告诉用户一个比任何单独 bug 都重要的结论
4. **假设链分析的深度无可替代**：Seed 7 的 BF 懒初始化→cascade failure 是 CR 永远不会产生的分析
5. **三方搭配最优**：CR 兜底领域广度，v3.1 深挖假设链 + 过滤误报，Radiate 提炼系统性 Pattern

## 文件索引
- CR+debug: `docs/benchmarks/openbse-2026-03-26-cr-debug.md`
- AFM v3.1: `docs/benchmarks/openbse-2026-03-26-afm-v31.md`
- AFM Radiate: `docs/benchmarks/openbse-2026-03-26-afm-radiate.md`
- 本文件: `docs/benchmarks/openbse-2026-03-26-comparison.md`
