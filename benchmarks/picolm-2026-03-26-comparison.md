# picolm 三路盲测对比报告

> 日期：2026-03-26 | 目标：RightNow-AI/picolm (commit cf3f2df)
> 13 个 C 源文件，~2,500 行代码 | 纯 C，嵌入式 LLM 推理引擎
> 空 prompt 盲测，无引导

## 结果总览

| 维度 | CR+debug | AFM v3.1 | AFM Radiate |
|------|----------|----------|-------------|
| Findings/Seeds | 21 (1C/4H/7M/9L) | 23 (8V/8H/3U/4noise) | 25 (12V/8H/5U) |
| Tool calls | 26 | 42 | — |
| Tokens | 83K | 110K | — |
| 耗时 | 221s | 394s | — |

## 核心 Finding 归属分析

### 三方共识
1. **GGUF 解析无边界检查** — reader_t.size 存了但从不检查，所有 read_u* 都裸读
2. **n_dims 堆溢出** — dims[4] 固定大小但 n_dims 来自文件无上限
3. **32 位整数溢出** — prompt token 分配在 Pi（32-bit ARM）上可溢出
4. **tokenizer_decode 静态缓冲区** — 非线程安全，计划的 server mode 会炸

### v3.1 独家发现
- **V1 Q5_K crash**：README 声称支持 Q5_K 量化但无对应 dequantize kernel，直接 exit(1)。同类：Q4_1/Q5_0/Q5_1/Q8_1 全是 enum-only 无实现。
  - 这是经典 AFM 发现：**假设 "README 承诺的功能都已实现" 失效**
- **V3 Grammar 阻断合法 JSON 字符串内 token**：检查负花括号深度先于检查 in_string，导致 JSON 字符串内的 `}` 被错误 mask
  - 深层逻辑 bug，CR 没找到
- **V5 死代码 token_has_unmatched_quote**：预计算、分配、填充、释放，但从未被读取。32KB 浪费 + 误导性存在
- **V6 性能声明矛盾**：README "~8 tok/s" vs BLOG "30+ tok/s" on Pi 4
- **Meta-patterns**: "脚手架创造虚假信心"（V1, V5）、"可选路径绕过核心设计规则"（V7, V8）

### Radiate 独家发现
- **RADIATE 辐射效果显著**：从 V1 的 "trusted input source" meta-pattern 辐射出 20+ 未保护的 read 调用点，并追踪到 tokenizer.c:70 的同类问题
- **"Spec compliance assumed, not enforced" meta-pattern**：GGUF spec 说最多 4 维，开发者硬编码 dims[4]，但解析器不校验 spec 合规性。从这个 meta-pattern 辐射出 4 个同类位置
- 12V 是三方最多，但需要检查是否有过度断言

### CR 独家/更细致的
- **Gateway 默认 0.0.0.0** — 网络暴露
- **KV cache 无完整性检查** — 损坏的缓存静默产生垃圾输出
- **madvise hint 次优** — MADV_SEQUENTIAL 用在随机访问 mmap 上
- 分类更细（1C/4H/7M/9L），Low 级别覆盖更广

## 定性分析

### 深度对比——这个仓库差异最明显

**CR** 找 bug 的方式：逐文件扫描，找到一个报一个。21 个 finding 是 21 个独立的点。

**v3.1** 的做法不同：
- V1（Q5_K crash）不是代码 bug，是 **设计承诺与实现的不一致** — CR 不会想到去对比 README 声称的功能和实际实现
- V3（Grammar 逻辑 bug）追踪了操作顺序的假设：开发者假设"先检查花括号再检查字符串"和"先检查字符串再检查花括号"等价，但不等价
- V5（死代码）不只是"有未使用代码"，而是"这个存在创造了虚假信心——让人以为 grammar 有 string-aware masking"

**Radiate** 在 v3.1 基础上的增量：
- 辐射搜索把 "一个位置的 bug" 变成 "一整类 bug 的完整地图"
- 从 V1 的 meta-pattern 辐射出来的 20+ 调用点，CR 只找到其中的子集

### 小仓库放大了方法论差异

picolm 只有 13 个文件 2500 行。在这个规模上：
- CR 的广度优势几乎消失——13 个文件 CR 也能全覆盖
- 但深度差异反而更明显——V1/V3/V5 这类需要"对比设计意图和实现"的发现，CR 完全没有
- Radiate 的辐射在小仓库上更有效——从一个 meta-pattern 出发，13 个文件都能 grep 到

### 激进度趋势

| 仓库 | v3.1 V | Radiate V | CR findings |
|------|--------|-----------|-------------|
| 2fauth (18K 行 TS) | 14 | 10 | 24 |
| picolm (2.5K 行 C) | 8 | 12 | 21 |

有趣翻转：在 picolm 上 **Radiate 比 v3.1 更激进**（12V vs 8V）。可能原因：
- 纯 C 项目 meta-pattern 更清晰（"trusted input" 模式在整个解析器中反复出现）
- RADIATE 操作在这种高密度小项目上特别有效——一个 meta-pattern 辐射出一整片

## 结论

1. **小仓库放大深度差异**：CR 广度优势消失，AFM 的假设链分析和 Radiate 辐射价值凸显
2. **v3.1 独家发现质量极高**：Q5_K crash（设计 vs 实现）、Grammar 逻辑 bug、死代码误导——全是 CR 不会找到的
3. **Radiate 辐射在 C 项目上特别有效**：meta-pattern + grep = 完整 bug 地图
4. **语言切换无障碍**：从 TS→C，方法论表现一致甚至更好
5. **纯 C 项目的 bug 模式天然适合假设分析**：边界检查、整数溢出、内存安全——都是关于"开发者假设了什么"

## 文件索引
- CR+debug: `docs/benchmarks/picolm-2026-03-26-cr-debug.md`
- AFM v3.1: `docs/benchmarks/picolm-2026-03-26-afm-v31.md`
- AFM Radiate: `docs/benchmarks/picolm-2026-03-26-afm-radiate.md`
- 本文件: `docs/benchmarks/picolm-2026-03-26-comparison.md`
