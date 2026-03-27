# AFM Radiate v3.2 Report: PicoLLM

**Target**: picolm — ultra-lightweight LLM inference engine in C
**Methodology**: AFM Radiate v3.2 (seed-first assumption engine with radiation search)
**Date**: 2026-03-26

---

## Seed Map

**Total seeds**: 25
**VULNERABLE**: 12
**HOLDS**: 8
**UNCLEAR**: 5

---

## VULNERABLE Findings

### V1: GGUF Parser — No Bounds Checking on Reader (model.c:45-77, 206)

**Observation**: The `reader_t` struct stores `size` (line 43) but no read function (`read_u8`, `read_u16`, `read_u32`, `read_u64`, `read_f32`, `read_gguf_string`) ever checks `r->pos < r->size` before reading.

**Premise**: "The GGUF file is well-formed because it came from a trusted source (HuggingFace, user's own model export)."

**Mechanism**: Every read advances `r->pos` blindly. A crafted or truncated GGUF file causes reads past the mmap'd region, triggering SIGSEGV or leaking adjacent memory.

**Impact**: Crash or potential information disclosure. Since this is an mmap'd file, reads past the end of the file but within the page boundary may return zeroes or stale data; reads past the mapping trigger segfault.

**Verdict**: **VULNERABLE** — `r.size` is stored but never consulted. Complete absence of bounds checking.

**WHY**: The developers assumed GGUF files are trusted local assets. On embedded Pi deployments, users download models from the internet via `curl` (install.sh:131). A corrupted download or malicious model file hits this path directly.

**Meta-pattern**: "Trusted input source assumption" — data from external sources treated as pre-validated.

**RADIATE**: Searched all `read_u*` / `read_f32` / `read_gguf_string` call sites. Found 20+ unprotected reads in the GGUF parser (lines 209-314). Also found the same pattern in tokenizer.c:70 where `slen` from GGUF controls malloc size (see V2).

---

### V2: Heap Buffer Overflow via `n_dims` in Tensor Info Parsing (model.c:309-311)

**Observation**: `tinfos[i].n_dims = read_u32(&r)` reads a uint32_t from the file. The loop `for (uint32_t d = 0; d < tinfos[i].n_dims; d++) { tinfos[i].dims[d] = read_u64(&r); }` writes to `dims[4]` — a fixed-size array of 4 elements.

**Premise**: "GGUF tensors have at most 4 dimensions."

**Mechanism**: If `n_dims > 4`, the loop writes past `dims[4]`, corrupting adjacent struct fields (`type`, `offset`) and potentially adjacent heap allocations.

**Impact**: Heap buffer overflow. Exploitable for arbitrary write via crafted GGUF file.

**Verdict**: **VULNERABLE** — no check that `n_dims <= 4`.

**WHY**: GGUF spec says max 4 dims, so the developer hardcoded the array. But the parser doesn't enforce spec compliance — it trusts the file to be spec-conformant.

**Meta-pattern**: "Spec compliance assumed, not enforced" — struct sizes match spec expectations but inputs aren't validated against those expectations.

**RADIATE from meta-pattern**: Same pattern appears at:
- model.c:304 — `n_tensors` from file controls malloc size, no upper bound check
- model.c:119/266/279 — `arr_len` from file controls iteration count, no bounds
- model.c:283 — `r.pos += arr_len * 4` can overflow
- tokenizer.c:70 — `slen` from GGUF controls malloc size (uint64_t on 32-bit → wraparound)

---

### V3: Grammar Constraint Masks Tokens Incorrectly When In String (grammar.c:94-105)

**Observation**: In `grammar_apply`, the brace/bracket depth check (lines 98-101) runs BEFORE the `in_string` guard (line 105). When `in_string == 1`, tokens containing `}` or `]` characters have their precomputed `brace_delta` applied, causing them to be masked as NEG_INF — even though the closing braces are inside a JSON string and structurally valid.

**Premise**: "Pre-computed token deltas accurately predict structural impact."

**Mechanism**: `analyze_token()` (line 12-41) counts ALL braces/brackets in a token's text, regardless of whether they appear inside quoted strings within the token. Then `grammar_apply` uses these deltas to reject tokens that would make depth negative. But when `g->in_string == 1`, a token like `value}` (closing a string then a brace) has `brace_delta = -1`, gets rejected at line 99-100, even though it should be allowed.

**Impact**: JSON grammar mode incorrectly suppresses valid tokens when string content contains brace/bracket characters. This can cause generation to fail or produce malformed JSON by forcing the model away from correct completions.

**Verdict**: **VULNERABLE** — the check order is wrong; structural checks should be skipped or deferred when `in_string`.

**WHY**: The `in_string` guard was added after the delta check, or the developer didn't reason about the ordering. The `token_has_unmatched_quote` field was computed (to track string transitions per token) but never integrated — evidence the developer recognized the problem but didn't complete the fix.

**Meta-pattern**: "State-dependent filtering with state lagging one token behind" — the grammar's `in_string` reflects state after the *previous* token, but filtering decisions apply to the *candidate* token.

**RADIATE**: `token_has_unmatched_quote` is computed at grammar.c:40, stored at grammar.c:58/66, allocated, freed — but never read in `grammar_apply`. Dead code confirming incomplete implementation.

---

### V4: Division by Zero if GGUF Missing Head Count Fields (model.c:293, 566)

**Observation**: `cfg->head_dim = cfg->n_embd / cfg->n_heads` at line 293. `cfg` is part of `model_t` which is `memset(0)` at line 539. If the GGUF file lacks `llama.attention.head_count`, `n_heads` remains 0. Similarly, `kv_mul = n_heads / n_kv_heads` at line 566.

**Premise**: "All required GGUF metadata fields are present."

**Mechanism**: Division by zero → crash (SIGFPE on most platforms, undefined behavior per C standard).

**Impact**: Crash on malformed or non-LLaMA-architecture GGUF files.

**Verdict**: **VULNERABLE** — no validation that critical config fields are non-zero after parsing.

**WHY**: The parser only handles LLaMA-family models. Non-LLaMA GGUFs (GPT-2, Mistral with different key names, etc.) may lack these fields.

**Meta-pattern**: "Critical computation depends on user-provided configuration without validation."

**RADIATE**: Checked all divisions in the codebase. Found additional instances:
- model.c:435 `int kv_dim = c->n_kv_heads * c->head_dim` — if head_dim is 0, downstream allocations are 0 but dequantize writes to them → crash
- model.c:293 `cfg->head_dim = cfg->n_embd / cfg->n_heads` — the first hit
- tensor.c:153 `ss = 1.0f / sqrtf(ss / (float)size + 1e-5f)` — `size` comes from config, if 0 → division by zero (but the epsilon saves it for the sqrt, the real issue is `size` controlling loop bounds elsewhere)

---

### V5: Prompt Token Buffer Silently Truncates (picolm.c:168-170)

**Observation**: `max_prompt_tokens = (int)strlen(prompt) + 3`. This assumes ≤1 token per byte of input. For SentencePiece BPE with byte fallback, the worst case is closer to `strlen(prompt) * 3 + 4` (every space becomes 3-byte `▁`, each byte becomes a separate token) plus BOS. The `tokenizer_encode` function respects `max_tokens` (tokenizer.c:215), so no buffer overflow occurs — but the prompt is silently truncated.

**Premise**: "Token count ≤ byte count + 3 for any reasonable input."

**Mechanism**: For text with many multibyte characters or binary content piped via stdin, the actual token count can exceed `strlen(prompt) + 3`. The tokenizer silently drops excess tokens. The user sees no error — the model generates from a truncated prompt.

**Impact**: Silent prompt corruption. The model answers a different question than asked. For automated PicoClaw tool-calling, this could cause incorrect tool invocations.

**Verdict**: **VULNERABLE** — silent data loss with no warning.

**WHY**: The developer estimated token count ≈ byte count, which is roughly true for English text with BPE merges but false for edge cases.

**Meta-pattern**: "Heuristic buffer sizing without overflow detection" — buffer size estimated from input length, but the actual consumption depends on content.

---

### V6: `malloc` in Hot Sampling Loop (sampler.c:75)

**Observation**: Every call to `sampler_sample` with `top_p < 1.0` allocates `vocab_size * sizeof(prob_index_t)` via `malloc`, sorts it with `qsort`, then `free`s it. For a 32K vocab, this is ~256 KB per token.

**Premise**: "Allocation overhead is negligible compared to model inference."

**Mechanism**: On Raspberry Pi (the primary target), `malloc`/`free` of 256 KB per token causes heap fragmentation and syscall overhead. With 256 tokens of generation, that's 256 malloc+qsort+free cycles.

**Impact**: Significant performance degradation on the target platform. On Pi Zero with limited RAM, repeated large allocations can trigger OOM killer.

**Verdict**: **VULNERABLE** — performance-critical path uses heap allocation per invocation on a memory-constrained target platform.

**WHY**: Developer wrote clean code (allocate, use, free) without considering the hot-loop context on embedded hardware.

**Meta-pattern**: "Desktop allocation patterns on embedded target."

---

### V7: Static Buffers in `tokenizer_decode` — Not Reentrant (tokenizer.c:231, 249)

**Observation**: `tokenizer_decode` returns pointers to `static char byte_buf[2]` and `static char space_buf[256]`. These are shared across all calls.

**Premise**: "Decode is called sequentially; the returned pointer is consumed before the next call."

**Mechanism**: In picolm.c:223-224, the return value is passed directly to `printf("%s", piece)` and flushed. Currently safe because single-threaded and consumed immediately. However, if anyone stores the returned pointer or calls `tokenizer_decode` twice before consuming the first result, data is silently corrupted.

**Impact**: Currently benign in the single call pattern. But the API contract is dangerous — any future multi-token batch decode or concurrent usage silently corrupts output.

**Verdict**: **VULNERABLE** — the function signature promises a usable string pointer but the implementation invalidates it on next call, with no documentation warning.

**WHY**: Common C pattern for avoiding allocation in decode paths. Acceptable in llama.cpp's tightly controlled call sites, but this project exposes it as a public API in tokenizer.h without the "static buffer" caveat.

---

### V8: Unvalidated Negative/Zero Arguments from CLI (picolm.c:86-96)

**Observation**: `atoi`/`atof` are used for all CLI arguments without range validation. `max_tokens = atoi(argv[++i])` — negative value causes `total_steps = n_prompt + max_tokens` to underflow. `num_threads = atoi(argv[++i])` — 0 or negative passed to `tensor_set_threads` (clamped to 1 at tensor.c:29, so safe). `context_override = atoi(argv[++i])` — negative value passes the check at model.c:290 (`max_seq_len > 0 && max_seq_len < cfg->max_seq_len`), so safe.

**Premise**: "Users provide reasonable positive integers."

**Mechanism**: `max_tokens < 0`: `total_steps = n_prompt + max_tokens` could be < n_prompt, causing zero generation tokens (benign). `temperature < 0`: `inv_temp = 1.0f / temperature` produces negative temperature, inverting the logit distribution (unexpected behavior). `top_p < 0`: Nucleus sampling with negative threshold — `cum >= s->top_p` is always true, sampling from just the top-1 token.

**Impact**: Unexpected model behavior with negative parameters. Not a crash but a usability/correctness issue.

**Verdict**: **VULNERABLE** — negative temperature silently inverts sampling; negative top_p silently becomes greedy.

---

### V9: Embedding Lookup Out-of-Bounds (model.c:577)

**Observation**: `const void *embd_row = (const uint8_t *)w->token_embd + (size_t)token * row_bytes` — `token` comes from `prompt_tokens[pos]` which comes from `tokenizer_encode`. During generation, `token = next` where `next` comes from `sampler_sample`.

**Premise**: "Sampled tokens are always in range [0, vocab_size)."

**Mechanism**: `sampler_sample` returns indices from the logits array which is `[0, vocab_size)`, so this is safe during generation. But during prefill, `token = prompt_tokens[pos]` where `prompt_tokens` contains token IDs from the tokenizer. The tokenizer uses vocab indices from the vocab array, which are `[0, vocab_size)` — also safe. However, if `token` is the BOS token (typically 1), the embedding lookup is `1 * row_bytes` which is valid.

**Verdict**: Actually **HOLDS** — both the tokenizer and sampler produce in-range token IDs. But no explicit bounds check exists — relies on invariants from other components.

Wait — I need to reconsider. During the cache-skip path, `token = prompt_tokens[start_pos - 1]` (line 188). If `start_pos == n_prompt` (cache covers entire prompt), then `token = prompt_tokens[n_prompt - 1]`. Then `pos = start_pos - 1 = n_prompt - 1`. At line 207, `pos < n_prompt - 1` is false, so we enter generation. `next = sampler_sample(...)`. Then `token = next`. This is fine.

Reclassifying: actually let me check if `token` itself is ever validated against vocab_size before being used to index the embedding table at line 577.

There is no bounds check. If a corrupted KV cache file causes `start_pos` to be wrong, or if the tokenizer returns an out-of-range index (it shouldn't, but the binary search in vocab_lookup returns -1 for not-found, and the byte fallback could also return -1), then the token index could be -1 or out of range.

Let me check: tokenizer.c:167-169 — if byte_tok lookup fails, `tok` is -1 but nothing is added to `merge_buf` (line 168 only adds if `tok >= 0`). So the only way to get -1 in `merge_buf` would be... actually no, -1 is never added. BOS at line 111 is always `t->bos_id` which is from GGUF (uint32_t), so it's in range if the model is valid.

**Revised Verdict**: **HOLDS** — token IDs stay in range through normal operation. But no defensive bounds check exists.

---

### V10: `token_has_unmatched_quote` Dead Code (grammar.c:40, 58, 66, 173)

**Observation**: `token_has_unmatched_quote` is computed per-token during `grammar_init`, allocated, populated, freed — but never read in `grammar_apply` or `grammar_advance`.

**Premise**: "All pre-computed token metadata is used for grammar enforcement."

**Mechanism**: The field was intended to help track string state transitions per candidate token during `grammar_apply`, enabling correct handling of tokens that open or close strings. Without it, the grammar cannot properly reason about whether a candidate token would change `in_string` state.

**Impact**: This is the missing piece that causes V3 (incorrect masking of brace-containing tokens during string state). The infrastructure was built but never wired in.

**Verdict**: **VULNERABLE** — dead code indicating abandoned fix for V3. The grammar constraint is less effective than intended.

---

### V11: KV Cache Path Allows Writing to Arbitrary Paths (picolm.c:99-100, model.c:739)

**Observation**: `--cache <file>` takes a user-provided path and writes KV cache data to it via `fopen(path, "wb")` (model.c:739). No path validation or sandboxing.

**Premise**: "The user controls the cache path and wouldn't overwrite important files."

**Mechanism**: `picolm model.gguf --cache /etc/passwd -p "test" -n 1` would overwrite `/etc/passwd` with binary KV cache data. On Pi deployments running as root (common for hobbyist setups), this is destructive.

**Impact**: Arbitrary file overwrite. Not a traditional vulnerability (user does it to themselves), but in PicoClaw automated-agent context, the cache path could come from an untrusted tool-call response.

**Verdict**: **VULNERABLE** — no path sanitization for file write operations.

---

### V12: Integer Overflow in `total_steps` Calculation (picolm.c:190)

**Observation**: `int total_steps = n_prompt + max_tokens`. Both are `int`. If `n_prompt` is near INT_MAX (from a very long prompt) or `max_tokens` is large, this overflows to negative, causing the generation loop `for (; pos < total_steps; pos++)` to never execute (or to wrap around).

**Premise**: "Prompt length + generation length fit in an int."

**Mechanism**: `n_prompt` comes from `tokenizer_encode` which returns an int bounded by `max_prompt_tokens` (which is `strlen(prompt) + 3`, itself an int). `max_tokens` is from `atoi`. If both are large positive ints, their sum overflows. The `total_steps > model.config.max_seq_len` check at line 191 would catch most cases since `max_seq_len` is typically small (2048-8192), but if `max_seq_len` is also large (from GGUF override), the overflow stands.

**Impact**: In practice, mitigated by small `max_seq_len`. But for models with large context (128K+), this becomes a real issue.

**Verdict**: **VULNERABLE** — integer overflow in control flow, mitigated only by typical model config values.

---

## HOLDS Findings

### H1: KV Cache Validation on Load (model.c:787-808)

The cache loader checks magic, layer count, kv_dim match, and n_pos bounds. Reasonably robust against corruption (not malicious crafting, but accidental mismatch). **HOLDS**.

Counter-evidence search: Could n_pos be manipulated to bypass checks? `n_pos` as int from uint32_t — if > INT_MAX, becomes negative, fails `n_pos > seq_len`. If 0, function returns 0 (treated as failure). If negative, loop doesn't execute. **HOLDS**.

### H2: Online Softmax Numerical Stability (model.c:659-672)

The online softmax implementation correctly tracks `max_score` and uses `correction = expf(max_score - score)` for reweighting. The initial `max_score = -1e30f` and `sum_exp = 0` are correct for the first iteration. When `score > max_score`, all accumulators are corrected. When `score <= max_score`, the exponential is bounded by `expf(score - max_score) <= 1.0f`. Numerically sound. **HOLDS**.

Counter-evidence: Could `expf(max_score - score)` underflow to 0? If `max_score - score < -87` (FP32 min exponent), yes — but this means the old contributions are negligible compared to the new score, so zeroing them is correct behavior. **HOLDS**.

### H3: FP16 Conversion Correctness (quant.c:11-78)

`fp16_to_fp32` and `fp32_to_fp16` handle all special cases: zero, subnormals, infinity, NaN, overflow, underflow. Round-to-nearest-even in `fp32_to_fp16` (line 71 adds bit 12). Checked edge cases: exponent boundary at 31, subnormal renormalization loop. Implementation matches IEEE 754. **HOLDS**.

### H4: BPE Merge Loop Correctness (tokenizer.c:176-212)

The merge loop finds the highest-score adjacent pair, merges them, shifts the array left. O(n^2) but correct. The `merged` buffer at line 189 has a 256-byte limit with `continue` guard. Token indices from `vocab_lookup` are valid. **HOLDS**.

### H5: Thread Safety in matmul (tensor.c:67-122)

Matmul divides output rows among threads, each writing to non-overlapping ranges of `out[]`. Input `x` and `W` are read-only. No shared mutable state. Pthread create/join is correct. **HOLDS**.

### H6: RoPE Implementation (tensor.c:210-256)

Interleaved (not rotary-half) RoPE: `(q0*cos - q1*sin, q0*sin + q1*cos)` for pairs `(q[2i], q[2i+1])`. Matches the LLaMA/GPT-NeoX convention. NEON path uses `vld2q_f32` for interleaved load, scalar fallback is straightforward. **HOLDS**.

### H7: SwiGLU FFN (model.c:686-696)

`gate = silu(W_gate @ x)`, `up = W_up @ x`, `hidden = gate * up`, `out = W_down @ hidden`. Correct SwiGLU implementation matching LLaMA architecture. **HOLDS**.

### H8: mmap Cleanup (model.c:190-201, 708-718)

`model_free` frees `mem_block`, `kv_block`, then calls `munmap_file`. All allocations have corresponding frees. The cleanup order is correct (free runtime state before unmapping weights). **HOLDS**.

---

## UNCLEAR Findings

### U1: GGUF Version 3 Compatibility (model.c:216-218)

Accepts GGUF v2 and v3 but the metadata key names (`llama.embedding_length` etc.) may differ between versions. Without test files for v3, unclear if all fields parse correctly. **UNCLEAR — needs-more-time**.

### U2: SentencePiece Byte Fallback Completeness (tokenizer.c:162-171)

The byte fallback tries `<0xHH>` format. Some tokenizers use `<0x0H>` (single hex digit) or different formats. Whether the target model (TinyLlama) uses this exact format is model-dependent. **UNCLEAR — needs-more-time**.

### U3: Build Correctness on Pi Zero 32-bit (Makefile:27-29)

The `pi-arm32` target uses `arm-linux-gnueabihf-gcc` with `-march=armv6 -mfpu=vfp`. The code uses `size_t` extensively; on 32-bit, `size_t` is 32 bits, and several computations like `(size_t)l * seq_len * kv_dim` could overflow for large models. **UNCLEAR — needs-more-time**.

### U4: Concurrent Grammar + Sampling State (grammar.c, sampler.c)

Both grammar and sampler maintain internal mutable state. Currently single-threaded use only. If PicoClaw ever runs multiple inference streams, both would need to be instanced. Not a current bug. **UNCLEAR**.

### U5: install.sh PATH Injection (install.sh:251-256)

The script appends to `.bashrc`/`.zshrc` by checking for `picolm/bin` substring. If a user has a different picolm installation, the check could pass for the wrong path. Edge case. **UNCLEAR — needs-more-time**.

---

## Root Causes (grouped by shared failed assumption)

### Root Cause 1: "Trusted File Input" (V1, V2, V4, V12)

**Failed assumption**: GGUF files and cache files are well-formed and from trusted sources.

**Reality**: On the target platform (Raspberry Pi), models are downloaded from the internet via `curl`. The install script has basic size checking (install.sh:136-139) but no integrity verification (no checksum). Truncated downloads, corrupted SD cards, or malicious files all hit the parser with no validation.

**Affected paths**: All of `parse_gguf()`, `kvcache_load()`, tensor info parsing, tokenizer data extraction.

### Root Cause 2: "State-Timing Mismatch in Grammar" (V3, V10)

**Failed assumption**: Pre-computed per-token metadata can be used directly with current grammar state.

**Reality**: The grammar's `in_string` state reflects the state AFTER processing the previous token. Token deltas were precomputed without string-context awareness. The `token_has_unmatched_quote` field was built to bridge this gap but never integrated. The result is incorrect token masking when JSON strings contain structural characters.

### Root Cause 3: "Desktop Development Assumptions on Embedded Target" (V5, V6, V8)

**Failed assumption**: Allocation patterns, buffer sizing heuristics, and input validation appropriate for desktop development work fine on Raspberry Pi.

**Reality**: malloc-per-sample, heuristic buffer sizing, and unvalidated CLI args have outsized impact on memory-constrained embedded systems where the project is explicitly designed to run.

---

## Assumption Radiation Map

```
V1 (no bounds check in reader)
  └─ RADIATE → V2 (n_dims overflow — same pattern: file value controls array write)
      └─ RADIATE → model.c:304 (n_tensors controls malloc — same pattern)
      └─ RADIATE → model.c:283 (arr_len controls pos advance — same pattern)
      └─ RADIATE → tokenizer.c:70 (slen from GGUF controls malloc — same pattern)
  └─ RADIATE → V4 (missing fields → div-by-zero — same root: unvalidated GGUF data)

V3 (grammar state-timing mismatch)
  └─ RADIATE → V10 (dead has_unmatched_quote — evidence of incomplete fix for V3)

V6 (malloc in hot loop)
  └─ RADIATE → V5 (heuristic buffer sizing — same root: desktop patterns on embedded)
  └─ RADIATE → V8 (no input validation — same root: assumes reasonable user input)
```

Seeds found by RADIATE: V2 was found by radiating from V1's "no bounds check" pattern. V10 was found by radiating from V3's grammar logic investigation.

---

## Impact Chains

### Chain 1: Malicious GGUF → Heap Corruption → Code Execution
`Crafted GGUF file` → `n_dims > 4` (V2) → `heap buffer overflow in tensor info` → `corrupted type/offset fields` → `arbitrary pointer in weight lookup` → `potential code execution`

### Chain 2: Truncated Model Download → Silent Misbehavior
`Incomplete curl download` (install.sh) → `truncated GGUF` → `reader reads past data` (V1) → `garbage config values` → `div-by-zero` (V4) or `massive allocation` → `crash`

### Chain 3: JSON Grammar Produces Invalid JSON
`JSON mode enabled` → `model generates string with braces` → `grammar incorrectly masks closing tokens` (V3) → `forced suboptimal token` → `malformed JSON output` → `PicoClaw tool-call parsing fails`

### Chain 4: Long Multilingual Prompt → Silent Truncation
`User pipes long CJK/emoji text via stdin` → `token count > strlen + 3` (V5) → `prompt silently truncated` → `model answers wrong question` → `PicoClaw makes incorrect tool call`

---

## Defense Map

| Defense | Coverage | Gaps |
|---------|----------|------|
| `max_tokens` cap in tokenizer_encode | Prevents buffer overflow | Does NOT warn on truncation (V5) |
| `tensor_set_threads` clamping | Prevents 0 or >16 threads | Does not validate at CLI level (V8) |
| KV cache magic + dimension check | Prevents mismatched cache load | Does not prevent crafted cache with valid header (V11) |
| `total_steps` clamped to `max_seq_len` | Prevents running past context window | Does not prevent integer overflow before the clamp (V12) |
| `GGUF_MAGIC` check | Prevents non-GGUF files | Does not validate internal consistency (V1, V2) |
| Grammar depth limit (50) | Prevents runaway nesting | String-state logic is broken (V3) |

---

## Unexplored (UNCLEAR, needs-more-time)

- **U1**: GGUF v3 parsing compatibility — needs actual v3 test files
- **U2**: Byte fallback token format across different tokenizer implementations
- **U3**: 32-bit `size_t` overflow on Pi Zero for large model configs
- **U4**: Multi-stream concurrent inference safety
- **U5**: install.sh PATH manipulation edge cases
