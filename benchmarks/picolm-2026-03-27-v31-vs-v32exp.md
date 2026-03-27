# picolm: v3.1 vs v3.2-exp 对照实验

> 日期：2026-03-27
> 目标：验证 v3.2（v3.1 + Falsify 四级 verdict 内联）是否优于 v3.1
> 条件：空 prompt 盲测，同一 codebase（picolm），同模型（Opus 4.6）

---

## 一、量化对比

| 维度 | v3.1 (control) | v3.2-exp | 差异 |
|------|---------------|----------|------|
| Seeds | 28 | 27 | 持平 |
| 正面发现 | 11 VULNERABLE | 8 PROVEN + 9 SUSPECTED = 17 | v3.2 区分了确信度 |
| 防御确认 | 12 HOLDS | 5 HARDENED | v3.2 更少但更强 |
| 不确定 | 5 UNCLEAR | 5 UNCLEAR | 持平 |
| 根因 | 5 | 3 | v3.1 更细分 |
| Tool calls | 32 | 25 | **v3.2 更高效** |
| Tokens | 94K | 91K | 持平 |
| 时间 | ~392s | ~399s | 持平 |

---

## 二、逐条交叉映射

### v3.2 PROVEN vs v3.1 对应

| v3.2 PROVEN | v3.1 对应 | v3.1 Verdict | 匹配？ |
|-------------|-----------|-------------|--------|
| P1: prompt buffer 堆溢出（空格膨胀） | S01 | **HOLDS**（找到 max_tokens 截断守卫） | ⚠️ **关键分歧** |
| P2: grammar delta 忽略 string context | S20 | VULNERABLE | ✅ |
| P3: n_tensors 整数溢出 | S13 | VULNERABLE | ✅ |
| P4: n_heads=0 除零 | S11 辐射提及 | VULNERABLE（附带提及） | ✅ v3.2 独立出来 |
| P5: acc[256] 栈溢出 | S11 | VULNERABLE | ✅ |
| P6: scores 硬编码 4 bytes | S15 | VULNERABLE | ✅ |
| P7: KV cache 分配溢出 | RC1 辐射提及 | 未独立判定 | ⚠️ v3.2 独立 PROVEN |
| P8: embedding OOB（级联自 P6） | S12 | **HOLDS** | ⚠️ **关键分歧** |

### v3.1 VULNERABLE vs v3.2 对应

| v3.1 VULNERABLE | v3.2 对应 | v3.2 Verdict | 匹配？ |
|-----------------|-----------|-------------|--------|
| S11: acc[256] 栈溢出 | P5 | PROVEN | ✅ |
| S13: n_tensors 溢出 | P3 | PROVEN | ✅ |
| S14: reader 无边界检查 | 无 | **未发现** | ❌ v3.2 漏了 |
| S15: scores 类型假设 | P6 | PROVEN | ✅ |
| S18: sampler malloc 无 NULL 检查 | Seed 12 | SUSPECTED | ⚠️ v3.2 降级 |
| S20: grammar delta/string 不一致 | P2 | PROVEN | ✅ |
| S21: in_string 状态滞后 | P2 合并 | PROVEN（合并） | ✅ |
| S22: brace_depth 下溢链 | 无 | **未发现** | ❌ v3.2 漏了 |
| S23: vec_dot malloc 无检查 | Seed 22 | SUSPECTED | ⚠️ v3.2 降级 |
| S24: MADV_SEQUENTIAL 错配 | 无 | **未发现** | ❌ v3.2 漏了 |
| S27: n_dims > 4 溢出 | 无 | **未发现** | ❌ v3.2 漏了 |

---

## 三、关键分歧分析

### 分歧 1: prompt buffer（P1 vs S01）

**v3.1 说 HOLDS**：找到 `tokenizer_encode` 的 `n_tokens < max_tokens` 守卫（line 215），认为输出被截断不会溢出。

**v3.2 说 PROVEN**：构造了 10 个空格的证明——归一化膨胀后 34 tokens，但 buffer 只有 13 slots。

**谁对？** 需要验证。如果 `max_tokens` 参数传入的就是 `strlen(prompt) + 3`，那截断守卫确实防止了溢出（但有静默截断的质量问题）。v3.1 的分析更准确——有守卫在。v3.2 的证明假设守卫不存在。**v3.1 更准确。**

### 分歧 2: embedding lookup（P8 vs S12）

**v3.1 说 HOLDS**：追踪所有 token 来源路径，确认全部有界。

**v3.2 说 PROVEN**：构造了一个级联攻击——P6 的 scores 解析错位导致 tokenizer 数据损坏，进而产生越界 token ID。

**谁对？** v3.2 的级联攻击逻辑上成立，但前提是 P6 先被触发。这是一个有条件的 PROVEN。v3.2 的级联推理更深，但独立来看 v3.1 的判定也对。**各有道理，v3.2 的级联思维更完整。**

### 分歧 3: OOM 类（S18/S23 vs SUSPECTED）

**v3.1 说 VULNERABLE**：明确的 NULL 指针解引用路径。

**v3.2 说 SUSPECTED**：OOM 是非确定性的，无法构造可复现的测试用例。

**谁对？** v3.2 的降级标准更严格——"我能不能写一个别人粘贴就能跑的 test case？"对 OOM 来说确实不行。但 v3.1 的判定也合理——缺少 NULL 检查就是缺少。**v3.2 标准更诚实，v3.1 判定更实用。**

---

## 四、v3.2 独家（v3.1 未独立发现）

| 发现 | 类型 | 质量 |
|------|------|------|
| P4: n_heads=0 除零 | 独立 PROVEN（v3.1 只在辐射中提及） | 高——明确的崩溃路径 |
| P7: KV cache 分配溢出 | 独立 PROVEN（v3.1 只在 RC1 中提及） | 中——和 P3 同类 |
| P8: embedding OOB 级联 | 级联 PROVEN（v3.1 判 HOLDS） | 中——依赖 P6 前提 |

## 五、v3.1 独家（v3.2 完全未发现）

| 发现 | 类型 | 质量 |
|------|------|------|
| S14: reader 无边界检查（size 字段死代码） | VULNERABLE | 高——独立发现 |
| S22: brace_depth 下溢级联 | VULNERABLE（链式） | 中——依赖 S20 |
| S24: MADV_SEQUENTIAL 性能 | VULNERABLE（性能） | 低——非安全 |
| S27: n_dims > 4 堆溢出 | VULNERABLE | **高——v3.2 完全漏了** |

---

## 六、verdict 质量对比

### v3.2 的改进

1. **确信度分层**：PROVEN/SUSPECTED 的区分比 VULNERABLE 更诚实。v3.1 把 OOM 类和确定性溢出混在一起都叫 VULNERABLE
2. **证明纪律**：v3.2 被迫构造具体证明，导致 P1 虽然判断可能有误，但推理过程更详尽
3. **级联推理**：P8 的 P6→tokenizer→embedding 级联是 v3.1 没做到的深度
4. **Meta-pattern 更多样**：3 个不同的 meta-pattern vs v3.1 侧重单一 GGUF trust

### v3.2 的退步

1. **广度下降**：4 个 v3.1 独家发现（S14/S22/S24/S27）被漏掉
2. **过度构造证明**：P1 花大量篇幅构造了一个实际被守卫拦截的"证明"
3. **HARDENED 过少**：只有 5 个 vs v3.1 的 12 个 HOLDS，丢失了大量防御地图信息
4. **根因合并过度**：3 个根因 vs v3.1 的 5 个，丢了 RC2(OOM) 和 RC4(madvise) 和 RC5(死代码)

---

## 七、效率对比

| 维度 | v3.1 | v3.2-exp |
|------|------|----------|
| 独立发现数 | 4 独家 | 3 独家 |
| 误判 | 0 明显 | 1（P1 假 PROVEN） |
| Tool calls | 32 | 25（**-22%**） |
| Tokens | 94K | 91K |
| 信息密度 | 11V + 12H = 23 有意义判定 | 8P + 9S + 5H = 22 有意义判定 |

---

## 八、结论

### 核心发现

1. **v3.2 的证明纪律增加了深度但牺牲了广度**：8 PROVEN + 9 SUSPECTED 的总覆盖面（17）实际上比 v3.1 的 11 VULNERABLE 更大，但漏了 4 个 v3.1 独家发现
2. **v3.2 产生了 1 个假 PROVEN**（P1——忽略了已有守卫），说明构造证明的过程如果不够仔细反而引入新的假阳性
3. **v3.2 的 SUSPECTED 类 = v3.1 的弱 VULNERABLE**：OOM 类发现被诚实降级，这是好事
4. **v3.2 的 HARDENED 远少于 v3.1 的 HOLDS**：防御地图信息大量丢失
5. **两者互补大于替代**：v3.1 独家 4 个 + v3.2 独家 3 个

### 假设验证

> 假设："去掉 subagent 开销，只保留证明纪律，能拿到 Falsify 80% 的 verdict 质量，但只用 v3.1 的 token 成本"

**部分成立：**
- ✅ Token 成本确实和 v3.1 持平（91K vs 94K）
- ✅ 确信度分层确实更诚实（PROVEN vs SUSPECTED 区分有价值）
- ❌ 广度下降——证明纪律消耗了本可用于扫描的注意力
- ❌ 产生了 1 个假 PROVEN——没有 subagent 的交叉验证，自己构造的证明没人审
- ⚠️ HARDENED 太少——证明纪律让 agent 倾向于继续攻击而不是确认防御

### 建议

**不吸收。v3.1 保持不变。**

原因：
1. v3.2-exp 的广度退步（-4 独家）抵消了深度收益（+3 独家）
2. 假 PROVEN 比 v3.1 的"保守 HOLDS"危害更大——假阳性消耗维护者信任
3. HARDENED 缺失导致防御地图残缺
4. Falsify 作为独立工具的价值在于 subagent 交叉验证，去掉这个核心机制就去掉了它的主要优势

**保留 Falsify 为独立工具**：需要证据时用 Falsify，日常分析用 v3.1。合并不如组合。
