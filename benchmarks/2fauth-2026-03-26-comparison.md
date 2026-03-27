# 2fauth-worker 三路盲测对比报告

> 日期：2026-03-26 | 目标：nap0o/2fauth-worker (commit adf13ce)
> 178 文件，18K 行代码 | TS + Vue + Hono + Cloudflare Worker
> 空 prompt 盲测，无引导

## 结果总览

| 维度 | CR+debug | AFM v3.1 | AFM Radiate |
|------|----------|----------|-------------|
| Findings/Seeds | 24 | 22 (14V/4H/4noise) | 21 (10V/8H/3U) |
| Tool calls | 49 | 49 | 66 |
| Tokens | 138K | 136K | 140K |
| 耗时 | 345s | 375s | 353s |
| 幻觉率 | 未全检 | 5/5 TRUE (抽检) | 5/5 TRUE (抽检) |

## 交叉验证：5 个关键 finding 全部 TRUE

| Claim | 来源 | 验证结果 |
|-------|------|----------|
| ENCRYPTION_KEY 泄露到前端 | 三方共识 | TRUE — authService.ts:79, authRoutes.ts:143 |
| Passkey CRUD 无 user scoping | 三方共识 | TRUE — webAuthnService.ts:201-230，无 userId 过滤 |
| XSS template literal 注入 | v3.1 + CR | TRUE — backupRoutes.ts:106/266/410/558，4 处 OAuth callback |
| SQL LIKE 通配符未转义 | CR 独有 | TRUE — vaultRepository.ts:40-42，影响低（参数化防了 SQL 注入） |
| Export password 本地常量遮蔽全局 | Radiate 独有 | TRUE — vaultService.ts:188 本地 5 vs config.ts 全局 12 |

## Finding 归属分析

### 三方共识（核心问题）
1. **ENCRYPTION_KEY 泄露** — 三方都找到，最严重
2. **Passkey CRUD 无 user scope** — 多租户穿越
3. **postMessage wildcard origin** — OAuth token 泄露
4. **Telegram webhook 无验证** — secret_token 被注释
5. **OAUTH_ALLOW_ALL='2' 隐藏绕过** — health check 只检查 'true'/'1'
6. **Import type='raw' 绕过验证** — 在 try/catch 外

### CR 独有
- SQL LIKE 通配符（低危但真实）
- CORS origin 反射 + credentials
- Rate limiter 静默失败
- JWT expiry 不一致
- 分页参数未校验
- 批量操作无上限
- 总计 ~6 个独有 finding

### v3.1 独有
- **XSS 完整利用链**（template literal + CSP unsafe-inline + unsafe-eval）— 这是最有价值的独有发现，因为它不只是指出 XSS，而是追踪了 CSP 配置为什么没防住
- 4 个 HOLDS 有验证说明（CSRF double-submit、OAuth state、Drizzle 参数化、PKCE）

### Radiate 独有
- **RADIATE 辐射发现**：从 export password 弱化辐射到 backup password 同类问题（meta-pattern: "局部常量遮蔽全局安全策略"）
- 4 个根因归类
- 8 个 HOLDS 带 challenge question

## 定性分析

### 深度比较
- **CR**: 找到最多 finding（24），但每个都是独立的点。修复建议是逐点补丁
- **v3.1**: XSS 利用链展示了假设链分析的价值——不只是"有 XSS"，而是"为什么 XSS 能利用成功"（CSP 配置假设了 inline script 是安全的）
- **Radiate**: 根因归类和辐射发现展示了系统性思维——"export password 弱化"不是孤立问题，是"局部常量遮蔽全局策略"这个 pattern 的表现

### 校准比较
- **v3.1 (14V/4H)**: 最激进。本次抽检 5/5 TRUE，但 14V 中是否有过度断言需要全量检查
- **Radiate (10V/8H/3U)**: 最保守。8 个 HOLDS 意味着更多"我验证了防御确实存在"
- **CR (24 mixed)**: 不使用 V/H 分类，但包含 7 个 Low 级别 finding

### 效率比较
- Token 用量几乎相同（136-140K），说明方法论不增加显著开销
- Radiate 多用了 17 个 tool calls（66 vs 49），因为 RADIATE 操作需要额外 grep 搜索
- 三者耗时相当（345-375s）

## 结论

1. **零幻觉**：抽检 5 个关键 finding 全部代码验证确认
2. **三方互补明显**：CR 赢广度（24 vs 22 vs 21），AFM 赢深度（利用链、根因归类）
3. **Radiate 辐射有效**：meta-pattern 发现是独家价值
4. **v3.1 激进但精准**：14V 抽检无误报，XSS 链是最深的独家发现
5. **AFM 的 HOLDS 有独立价值**：不只是"没发现问题"，而是"验证了防御存在且有效"

## 文件索引
- CR+debug: `docs/benchmarks/2fauth-2026-03-26-cr-debug.md`
- AFM v3.1: `docs/benchmarks/2fauth-2026-03-26-afm-v31.md`
- AFM Radiate: `docs/benchmarks/2fauth-2026-03-26-afm-radiate.md`
- 本文件: `docs/benchmarks/2fauth-2026-03-26-comparison.md`
