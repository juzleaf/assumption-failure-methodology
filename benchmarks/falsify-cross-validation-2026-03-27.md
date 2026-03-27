# Falsify 跨仓库逐条交叉验证报告

> 日期：2026-03-27
> 方法：逐条对比 CR / AFM v3.1 / Radiate / Falsify 的代码位置和 bug 内容
> 数据来源：4 仓库 × 4 方法 = 16 份报告 + 1 份 CR→Falsify 串行

---

## 一、Falsify 独家发现——诚实统计

**核心问题**：Falsify 的 PROVEN 发现中，有多少是其他三种方法完全没发现的？

| 仓库 | Falsify PROVEN | 完全独家 PROVEN | 与其他方法重叠 PROVEN | 独家率 |
|------|---------------|----------------|----------------------|--------|
| 2fauth | 8 | **2** | 6 | 25% |
| picolm | 6 | **0** | 6 | 0% |
| OpenBSE | 2 | **1***| 1 | 50%* |
| zigpty | 5 | **0** | 5 | 0% |
| **合计** | **21** | **3** | **18** | **14%** |

*OpenBSE 的 FS15（boiler 流量守恒）：v3.1 看了同一段代码但判定 HOLDS，Falsify 通过链式推理推翻为 PROVEN。严格说 v3.1 "看到了但判错了"，不是"没看到"。

**2fauth 独家 PROVEN 详情**：
1. **Rate limit reset 路径前缀不匹配** — `/api/auth/` vs 实际 `/api/oauth/`，成功登录后计数器永不清零。四种方法都检查了 rate limit，但只有 Falsify 对比了 reset 路径与实际路由。
2. **WebAuthn Passkey 登录完全跳过白名单** — OAuth 流程调用 verifyWhitelist，WebAuthn 完全没有。需要跨代码路径对照才能发现。

**结论**：Falsify 的主要价值**不是发现新 bug**（21 PROVEN 中仅 3 个独家），而是**证明已知 bug**和**过滤噪音**。

---

## 二、Falsify 覆盖率——对照 v3.1

| 仓库 | v3.1 VULN 总数 | Falsify 覆盖 | Falsify PROVEN | Falsify 降级 | Falsify 未涉及 |
|------|---------------|-------------|----------------|-------------|---------------|
| 2fauth | 16 | 12 (75%) | 6 | 6→SUSPECTED | 4 |
| picolm | 8 | 5 (63%) | 2 | 3→HARDENED/UNCLEAR | 3 |
| OpenBSE | 7 | 2 (29%) | 0 | 2→HARDENED/SUSPECTED | **5** |
| zigpty | 13 | ~8 (62%) | 3 | ~5 | ~5 |
| **合计** | **44** | **~27 (61%)** | **11** | **~16** | **~17** |

**v3.1 独家发现（Falsify 完全没涉及的）**：
- picolm: Q5_K 声称支持但未实现、README/BLOG 性能数字矛盾、死代码
- OpenBSE: DX coil bypass factor 懒初始化、DX coil auto-SHR 不一致、chiller -2.0°C clamp、室外湿度比硬编码 0.008
- 2fauth: WebAuthn rpID 伪造、CSP unsafe-inline、Emergency 后4位验证、JWT 过期配置不一致

**模式**：v3.1 在"声明 vs 实现差距"（Q5_K、README 矛盾）、"文档一致性"、"领域知识型发现"（DX coil 物理、湿度比）方面有 Falsify 不可替代的独家价值。

---

## 三、Falsify 覆盖率——对照 Radiate

| 仓库 | Radiate VULN 总数 | Falsify 覆盖 | Falsify PROVEN | Falsify→HARDENED | Falsify 未涉及 |
|------|------------------|-------------|----------------|-----------------|---------------|
| 2fauth | 10 | 9 (90%) | 4 | **2** | 1 |
| picolm | 11 | 7 (64%) | 5 | 2 | 4 |
| OpenBSE | 10 | 5 (50%) | **1** | **3** | 5 |
| zigpty | 11 | ~6 (55%) | 3 | — | ~5 |
| **合计** | **42** | **~27 (64%)** | **13** | **~7** | **~15** |

**Falsify 推翻 Radiate VULNERABLE → HARDENED 的具体案例**（共 7 个）：
1. 2fauth CORS 反射 — Falsify 证明 CSRF double-submit 有效防御
2. 2fauth backup key fallback — Falsify 证明 health check 阻止 fallback 触发
3. picolm static buffers — 当前单线程安全
4. picolm sampler malloc — 性能问题非正确性 bug
5. OpenBSE AUTOSIZE sentinel — 无物理参数在碰撞范围
6. OpenBSE graph 注释矛盾 — 代码正确，注释错但无害
7. OpenBSE 闰年硬编码 — TMY 行业惯例，内部一致

**Radiate 独家发现（Falsify 完全没涉及的）**：
- picolm: CLI 负数参数未校验、KV cache 路径可写任意文件
- OpenBSE: pump heat-to-fluid 公式、humidifier 蒸汽显热、heat recovery C_min 假设
- 2fauth: auto-backup 密码最低6字符（辐射发现）
- zigpty: openPty slave fd 泄漏、Unix tsfn 未 unref、Terminal.close() 固定 exitCode=0

**模式**：Radiate 的辐射搜索（从已知 bug 找同类）产生了 Falsify 无法替代的发现。Falsify 的贡献是纠正 Radiate 的判定精度（~17% 被推翻为 HARDENED）。

---

## 四、Falsify 噪音过滤价值

### 4.1 各方法"判错率"估算

以 Falsify HARDENED 作为"确认安全"的参照（有攻击验证支持），统计其他方法把 HARDENED 项标为问题的比例：

| 被 Falsify 判定 HARDENED 的项目 | CR 标为问题 | v3.1 标为 VULN | Radiate 标为 VULN |
|-------------------------------|------------|---------------|------------------|
| picolm static buffers | CR-1 High | V4 VULN | R-V7 VULN |
| picolm fp16 loop | CR-14 Low | — | — |
| picolm vec_dot 32KB | — | V8 VULN | — |
| OpenBSE 闰年 | F-06 High | S1 VULN | R10 VULN |
| OpenBSE AUTOSIZE | F-08 High | S6 HOLDS ✓ | R8 VULN |
| OpenBSE graph 注释 | F-03 High | S5 HOLDS ✓ | R9 VULN |
| 2fauth CORS | CR-3 High | v31-6 HOLDS ✓ | Rad-3 VULN |
| 2fauth backup key | CR-12 Medium | v31-16 HOLDS ✓ | Rad-16 VULN |

**观察**：
- **v3.1 的 HOLDS 判定很准**：在 Falsify 判定 HARDENED 的项目中，v3.1 有 4 次也判定了 HOLDS（OpenBSE AUTOSIZE、graph 注释、2fauth CORS、backup key）。v3.1 和 Falsify 在"确认安全"方面高度一致。
- **Radiate 的 VULNERABLE 判定偏激进**：6/7 项被 Falsify 推翻的 HARDENED 中，Radiate 都标了 VULNERABLE。
- **CR 的严重度偏高**：multiple HARDENED 项被 CR 标为 High。

### 4.2 判定精度排序

| 方法 | 判定类型 | 被 Falsify 推翻率 | 评估 |
|------|---------|------------------|------|
| Falsify | PROVEN | 0%（自定义） | 最严格标准 |
| v3.1 HOLDS | 确认安全 | ~0%（4/4 被 Falsify HARDENED 验证） | **与 Falsify 高度一致** |
| v3.1 VULN | 有问题 | ~30%（~5/17 被降级） | 中等精度 |
| Radiate VULN | 有问题 | ~17%（7/42 被推翻为 HARDENED） | 偏激进 |
| CR findings | 有问题 | ~45%（CR→Falsify 串行中 48% HARDENED） | 噪音最高 |

---

## 五、v3.1 HOLDS vs Falsify HARDENED 一致性

**这是最重要的发现之一**：v3.1 的 HOLDS 和 Falsify 的 HARDENED 高度一致。

| 项目 | v3.1 判定 | Falsify 判定 | 一致？ |
|------|----------|-------------|--------|
| OpenBSE AUTOSIZE | HOLDS | HARDENED | ✅ |
| OpenBSE graph 注释 | HOLDS | HARDENED | ✅ |
| OpenBSE fan heat 公式 | HOLDS | HARDENED | ✅ |
| 2fauth CORS | HOLDS | HARDENED | ✅ |
| 2fauth backup key | HOLDS | HARDENED | ✅ |
| 2fauth PKCE codeVerifier | HOLDS | SUSPECTED | ⚠️ 轻微分歧 |
| picolm embedding lookup | — | — | — |
| OpenBSE boiler SP-modulated | **HOLDS** | **PROVEN** | ❌ **Falsify 推翻** |

**6/7 一致，1/7 被推翻**（boiler 流量守恒）。

这说明：
1. v3.1 的 HOLDS 判定可信度很高（~86%）
2. Falsify 的独特价值在于那 ~14% 的推翻——用攻击验证纠正 v3.1 的推理判定
3. boiler 案例特别有意义：v3.1 看了代码说"设计正确"，Falsify 通过链式推理（结合迭代缺失）证明不正确

---

## 六、Falsify 在不同项目类型上的表现

| 维度 | 2fauth (Web/TS) | picolm (嵌入式/C) | OpenBSE (物理/Rust) | zigpty (系统/Zig) |
|------|-----------------|-------------------|--------------------|--------------------|
| Seeds | 25 | 23 | 16 | 18 |
| PROVEN | 8 (32%) | 6 (26%) | 2 (13%) | 5 (28%) |
| 独家 PROVEN | 2 | 0 | 1* | 0 |
| 覆盖 v3.1 VULN | 75% | 63% | **29%** | ~62% |
| 覆盖 Radiate VULN | 90% | 64% | **50%** | ~55% |
| HARDENED 推翻他人 | 2 | 3 | 3 | 1 |

**OpenBSE 是明显短板**：证伪率最低（13%）、覆盖率最低（29% v3.1 / 50% Radiate）、seeds 最少（16）。

**原因分析**：
1. 物理模拟引擎的 bug 多是"公式/常量不一致"——需要领域知识，无法通过构造输入来证伪
2. 架构层面的问题（缺失迭代求解器）可以 PROVEN，但组件级物理问题（DX coil、pump）超出 Falsify 的能力
3. Falsify 的 adversarial 思维在"输入→输出"链条清晰的系统上最有效（Web 应用、系统库）

---

## 七、CR→Falsify 串行管线客观评估

### picolm 五方法对照

| 指标 | CR | v3.1 | Radiate | Falsify 盲测 | CR→Falsify |
|------|-----|------|---------|-------------|------------|
| 总 findings/seeds | 21 | 19 | 25 | 23 | 24 |
| 最高确信发现 | 1C+4H | 8V | 12V | 6 PROVEN | 10 PROVEN |
| 独家发现 | 2 | 3 | 2 | 1 | 3 |
| 噪音率 | ~45% | ~37% | ~25% | ~0% | ~0% |

### 串行的真实增量

**增量 1：PROVEN 数量**
- CR→Falsify 10 PROVEN > Falsify 盲测 6 PROVEN（+67%）
- 但 10 PROVEN 中 4 个与盲测重叠，3 个是串行独有（div-by-zero、tokenizer 丢字节、Gateway），1 个盲测反而更强（prompt truncation 盲测 PROVEN 串行 SUSPECTED），2 个同 bug 不同角度
- **真实独有增量：3 个 PROVEN**

**增量 2：噪音过滤**
- CR 21 findings → 10 HARDENED（48% 被过滤）
- 这是管线最大价值——从"可能有问题"变成"确认没问题"

**增量 3：邻接发现**
- 3 个新 seed（reader_t 未使用防御基础设施、零维度级联故障、malloc NULL 检查）
- 这些是深度分析时的自然产物

### 串行的代价

| 管线 | Tokens | PROVEN | Token/PROVEN |
|------|--------|--------|-------------|
| CR 单独 | 83K | ~1 | 83K |
| v3.1 单独 | 110K | 8V (无证明) | — |
| Radiate 单独 | 104K | 12V (无证明) | — |
| Falsify 盲测 | ~98K | 6 | 16K |
| CR→Falsify | 83K+86K=169K | 10 | 17K |
| CR+v3.1+Radiate+Falsify 全部 | 395K | — | — |

串行 token 是盲测的 ~1.7x，但 PROVEN +67%。性价比合理。

---

## 八、修正之前的汇总报告

### 之前说错的

1. ~~"Falsify 的独家价值不是发现新 bug"~~ → 修正：2fauth 上有 2 个完全独家 PROVEN（其他三方法全漏），但 picolm 和 zigpty 上确实没有独家检测能力
2. ~~"82 seeds → 21 PROVEN（26%）"~~ → 数字正确，但需要标注 21 PROVEN 中仅 3 个（14%）是四方法中的独家发现
3. ~~"CR→Falsify 串行比盲测多 67% PROVEN"~~ → 数字正确，但 10 PROVEN 中真正的独有增量是 3 个，另外 4 个与盲测重叠

### 真正的 Falsify 价值排序

1. **证明质量**（最大价值）——把模糊的"可能有问题"转化为可运行的具体证明，21 PROVEN 中 18 个与其他方法重叠但其他方法没有给出证明
2. **噪音过滤**（第二价值）——24 HARDENED，推翻了 7 个 Radiate VULNERABLE + 多个 CR High findings
3. **判定纠错**（第三价值）——v3.1 HOLDS vs Falsify HARDENED 86% 一致，但那 14% 的推翻（如 boiler 流量守恒）是高价值洞察
4. **独家检测**（最小价值）——仅 3/21 PROVEN 是四方法独家，且集中在 2fauth 一个仓库
5. **自我纠错**（方法论价值）——跨仓库稳定（3/4），证明 PROVEN 标准有效

### 各方法不可替代性排序

| 方法 | 独家发现数（跨 4 仓库） | 独家类型 |
|------|----------------------|---------|
| **CR** | ~14 | 领域知识（信号常量、物理公式、数值算法）、Low 级 DoS/配置 |
| **v3.1** | ~10 | 声明 vs 实现差距、文档矛盾、设计规则违反、深层物理 |
| **Radiate** | ~8 | 辐射链（从已知 bug 找同类）、内部不一致 Pattern |
| **Falsify** | **~3** | 跨路径对照（WebAuthn vs OAuth）、链式推理推翻 HOLDS |

**CR 的独家发现最多**。这与 Round 3 的结论一致（CR 赢广度），但交叉验证让数字更精确。

---

## 九、给开源文档的诚实表述

### 要说
- "Falsify 的核心价值是证明质量，不是检测能力——21 PROVEN 中 86% 与其他方法重叠，但只有 Falsify 给出了具体的失败证明"
- "Falsify 过滤了 7 个其他方法的误判（VULNERABLE → HARDENED），减少了维护者的审查负担"
- "v3.1 的 HOLDS 判定和 Falsify 的 HARDENED 判定 86% 一致——两种方法在'确认安全'方面互相验证"
- "在 Web 应用上 Falsify 有 2 个四方法完全独家的 PROVEN，但在其他三种项目类型上没有独家检测能力"
- "物理模拟引擎是 Falsify 的短板——领域知识型 bug 无法通过构造输入来证伪"
- "CR→Falsify 串行的真实增量是 3 个独有 PROVEN + 48% 噪音过滤，不是之前说的'+67%'"

### 不要说
- ~~"Falsify 发现了其他方法找不到的 bug"~~（仅在 1/4 仓库成立）
- ~~"21 PROVEN 都是独家贡献"~~（86% 与其他方法重叠）
- ~~"自我纠错证明 Falsify 优于其他方法"~~（这是方法论特性，不是检测能力优势）
