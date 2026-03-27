# zigpty 三路盲测对比报告

> 日期：2026-03-26 | 目标：pithings/zigpty
> 11 个 .zig 文件 + 13 个 .ts 文件，~3700 行 | Zig+TS PTY 库（Node.js N-API）
> 空 prompt 盲测，无引导

## 结果总览

| 维度 | CR+debug | AFM v3.1 | AFM Radiate |
|------|----------|----------|-------------|
| Findings/Seeds | 25 (1C/4H/10M/10L) | 58 (13V/44H/1U) | 23 (11V/10H/2U) |
| Tool calls | 25 | 27 | 39 |
| Tokens | 83K | 103K | 97K |
| 耗时 | 263s | 641s | 320s |

**v3.1 播种数异常高**：58 个 seeds 是 Round 3 所有 12 runs 中最多的。3700 行代码里找到 58 个可疑点，平均每 64 行一个 seed。44 个 HOLDS 说明大部分经过验证确认安全。

## 核心 Finding 归属分析

### 三方共识
1. **Windows UAF** — exit monitor 线程 `alloc.destroy(ctx)` 后 JS 侧仍可访问
   - CR: F-01 Critical | v3.1: S27 VULNERABLE | Radiate: S09 VULNERABLE
2. **FD/Handle 双重关闭** — Unix `close()` 中 ReadStream.destroy() + fs.closeSync() 双关
   - CR: F-06 Medium | v3.1: S32 VULNERABLE | Radiate: S07 相关（Windows conin 双关）
3. **Flow control 断裂** — XOFF/XON 只匹配完整单字符 chunk
   - CR: F-08 Medium | v3.1: S33 VULNERABLE | Radiate: S16 VULNERABLE
4. **Windows ConPTY readiness 死锁** — 等数据才 ready，但有些程序等输入才产出
   - CR: F-10 Medium | v3.1: S35 VULNERABLE | Radiate: 未单独列出
5. **setuid/setgid 缺 initgroups** — 子进程保留父进程的 supplementary groups
   - CR: F-04 High | v3.1: S09 VULNERABLE | Radiate: S03 VULNERABLE

### 三方判定分歧

| Finding | CR | v3.1 | Radiate | 分析 |
|---------|-----|------|---------|------|
| napi_external 无 finalize | F-12 Low | S28 **HOLDS** | S02 **VULNERABLE** | v3.1 深入分析后认为 exit monitor 是真正 owner，JS ref 是次要的。Radiate 关注僵尸进程场景 |
| macOS P_COMM_OFFSET 243 | F-03 High | S01 **HOLDS** | S04 **VULNERABLE** | v3.1 验证了 arm64/x86_64 历史稳定性 + bounds check 保护；CR 和 Radiate 更关注 ABI 脆弱性 |
| PID recycling | F-13 Medium | S34 **HOLDS** | 未独立列出 | v3.1 指出这是所有 PTY 库的共同问题，_closed flag 有缓解 |
| page_allocator 内存浪费 | 未提及 | S23 **VULNERABLE** | 未提及 | v3.1 独家：每个 16 字节 DataChunk 分配 4KB page，256x 浪费 |

### CR 独家发现
- **F-02 High**: Unix exitMonitorThread 和 JS 线程竞态——exit callback 触发 destroy 时 JS 可能还在写
- **F-05 High**: Windows writeInput() 死循环——WriteFile 返回 0 bytes 不退出
- **F-07 Medium**: **SIGUSR1=10/SIGUSR2=12 硬编码 Linux 值**，macOS 上 10=SIGBUS 12=SIGSYS，发 SIGUSR1 会 crash 子进程
- **F-09 Medium**: Windows resize 竞态——resize 时 ConPTY 可能已关闭
- **F-11 Medium**: waitFor 回调中的 indexOf 用法可能导致假匹配
- **F-14 Low**: macOS 不设 IUTF8 标志
- 大量 Low 级别 finding（共 10 个）覆盖面广

**F-07 是本轮最有实战价值的独家发现**：信号常量跨平台硬编码是真实可复现的 bug，macOS 用户发 SIGUSR1 会直接 crash 子进程。

### v3.1 独家发现
- **S21 VULNERABLE**: Windows writeInput() **阻塞 JS 主线程**——同步写入如果子进程不消费 stdin，整个 Node.js event loop 冻结
- **S23 VULNERABLE**: **page_allocator 256x 内存浪费**——每个 ConPTY 输出 chunk 分配 4KB page 给 16 字节 struct
- **S39 VULNERABLE**: waitFor **zombie timer**——进程退出清了 listener 但没取消 pending timer，30 秒超时延迟
- **S03 VULNERABLE**: macOS `getPtyName` **返回类型说谎**——TS 声明返回 `{pty: string}` 但 macOS 实现永远返回 undefined
- **S42 VULNERABLE**: Standalone Terminal close() **关 master/slave 但 ReadStream 可能还有 pending reads**
- **S44 VULNERABLE**: loadNative() glibc-first fallback——Alpine/musl 上先试 glibc 静默失败
- **44 个 HOLDS 带完整验证**——这是 v3.1 在这个仓库上的独特价值

**Meta-patterns（独家）**：
- "Defensive double-close"（S32）——"确保关了" 反而造成更大问题
- "Fragile string matching"（S33）——equality 用在该用 indexOf 的地方
- "Platform-asymmetric lifecycle management"——Windows 做了 unref 但 Unix 没有
- "Synchronous blocking in async context"（S21）——同步系统调用藏在 async API 后面

### Radiate 独家发现
- **S01 VULNERABLE**: `openPty` slave fd 泄漏——raw `open()` API 返回 slave fd 但无 close path
- **S10 VULNERABLE**: Unix tsfn 未 unref——Node.js 进程无法正常退出（Windows 做了但 Unix 遗漏）
- **S14 VULNERABLE**: Terminal.close() 无条件报告 exitCode=0——进程 crash 也说"正常退出"
- **S17 VULNERABLE**: stopped 进程返回 exit_code=-1 而非信号信息
- **S19 VULNERABLE**: macOS execChild 修改全局 environ 指针——多线程 fork 不安全
- **S13 VULNERABLE**: WriteQueue 静默丢弃非 EAGAIN 错误

**四个核心 Pattern（独家）**：
1. **Unix/Windows 生命周期不对称** — Windows 有 unref/finalize，Unix 缺失
2. **close 序列 INVALID_HANDLE 检查不一致**
3. **错误静默吞掉**（WriteQueue + Terminal）
4. **NAPI external 缺安全网**（无 finalize + 无 alive 标记）

**辐射链 S06→S07→S09→S02** 构成完整的"生命周期同步缺陷"链——这是 Radiate 的结构性价值。

## 定性分析

### 这个仓库的独特性

zigpty 是 Round 3 中**最小但最密集**的目标：
- **跨语言** — Zig（系统层）+ TypeScript（用户层），两种内存模型交汇
- **跨平台** — Linux/macOS/Windows 三套不同实现
- **N-API 边界** — 原生线程和 JS 事件循环的同步是核心难点
- **并发密集** — PTY I/O 本质上是多线程的（read thread、exit monitor、JS main thread）

### 深度对比

**CR 在这个仓库上的突破：SIGUSR 硬编码**。F-07 是典型的"查常量表就能发现"的跨平台 bug，但需要知道 macOS 和 Linux 的信号编号不同。这是领域知识的价值，和 OpenBSE 里 CR 找到 HFG 不一致类似。

**v3.1 在这个仓库上的深度最大化**：58 个 seeds 中 44 个 HOLDS，意味着 v3.1 检查了大量"看起来可疑但实际正确"的代码。在一个 N-API + 多线程 + 跨平台的项目中，这种区分能力极其重要——很多"可疑代码"其实是特定平台的惯用法。

**Radiate 在这个仓库上的链路分析最清晰**：S06→S07→S09→S02 的辐射链把四个看似独立的 finding 连成一条"Windows 生命周期管理缺陷"的故事线。

### 激进度趋势（Round 3 全部 4 组）

| 仓库 | 语言 | 规模 | CR | v3.1 V | Radiate V |
|------|------|------|-----|--------|-----------|
| 2fauth (TS+Vue) | TS | 18K | 24 | 14 | 10 |
| picolm (C) | C | 2.5K | 21 | 8 | 12 |
| OpenBSE (Rust) | Rust | 15K | 21 | 7 | 11 |
| zigpty (Zig+TS) | Zig/TS | 3.7K | 25 | 13 | 11 |

**观察**：
- CR 广度在 21-25 之间稳定，不受语言/规模影响
- v3.1 VULNERABLE 数波动大（7-14），取决于代码库的"假设密度"
- Radiate VULNERABLE 在 10-12 之间最稳定
- zigpty 的 v3.1 13V 是 Round 3 第二高（仅次于 2fauth 14V），反映了 N-API 跨线程 + 跨平台的高假设密度

### v3.1 的 44 个 HOLDS 是什么

在 zigpty 上，v3.1 投入了大量精力验证：
- arena allocator 在 fork 后是否安全（HOLDS——exec 替换地址空间）
- `sigaction` 重置在 child 中的正确性（HOLDS）
- `waitpid` 在 zombie 场景下的行为（HOLDS——zombie 被 waitpid 回收）
- ConPTY pipe 继承语义（HOLDS——CreatePseudoConsole 内部 duplicate）
- napi_external 无 finalize 的安全性（HOLDS——exit monitor 是真 owner）
- `appendQuoted` Windows 命令行转义的完整性（HOLDS——覆盖了反斜杠 + 引号边界）

**这些验证对库的维护者极其有价值**——等于有人帮你检查了"这些看起来可疑的代码确实是对的"。

## 结论

1. **跨语言/跨平台仓库是三路差异的最佳展示场**：Zig+TS+N-API 的复杂性让每种方法各有独家价值
2. **CR 的平台知识发现了最有实战价值的 bug**：SIGUSR 信号常量硬编码是真实可复现的跨平台 bug
3. **v3.1 的 58 seeds 展示了极致的假设挖掘深度**：44 HOLDS 不是浪费，是"验证了什么是安全的"
4. **Radiate 的辐射链把 4 个独立 finding 连成一条 story**：结构性理解 > 点状发现
5. **N-API 边界是假设最密集的区域**：原生线程 vs JS 事件循环的同步假设几乎每个都值得验证

## 文件索引
- CR+debug: `docs/benchmarks/zigpty-2026-03-26-cr-debug.md`
- AFM v3.1: `docs/benchmarks/zigpty-2026-03-26-afm-v31.md`
- AFM Radiate: `docs/benchmarks/zigpty-2026-03-26-afm-radiate.md`
- 本文件: `docs/benchmarks/zigpty-2026-03-26-comparison.md`
