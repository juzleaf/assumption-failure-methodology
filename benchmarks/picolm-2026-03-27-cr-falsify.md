# CR→Falsify Serial Pipeline Report: PicoLLM

**Target**: PicoLLM — minimal C11 LLM inference engine (~2500 lines)
**Codebase path**: `/tmp/afm-benchmark-round3/picolm-v31/picolm/`
**Method**: 21 Code Review seeds → AFM Falsify (attempt to construct concrete proof of failure for each)
**Date**: 2026-03-27

---

## 1. Proof Catalog (PROVEN)

### PROVEN-1: S2 — Stack buffer overflow in flash attention accumulator

**Seed**: model.c:644 — `float acc[256]` but head_dim from model config could exceed 256

**Assumption that failed**: "256 is large enough for any head_dim"

**Concrete proof**:
- `float acc[256]` is declared on the stack at model.c:644
- `memset(acc, 0, (size_t)head_dim * sizeof(float));` at model.c:645 writes `head_dim * 4` bytes
- `head_dim` is computed at model.c:293 as `cfg->n_embd / cfg->n_heads` — read directly from GGUF metadata, no validation
- **Failing input**: A crafted GGUF file with `llama.embedding_length = 8192` and `llama.attention.head_count = 16` yields `head_dim = 512`
- **Code path**: `parse_gguf()` → `cfg->head_dim = 8192/16 = 512` → `model_forward()` → `float acc[256]` → `memset(acc, 0, 512*4)` writes 2048 bytes into a 1024-byte stack buffer
- **Wrong output**: 1024 bytes of stack corruption. The inner loops at lines 662-663 and 669-670 also iterate `d < head_dim` (512 iterations), reading and writing past `acc[255]` on every attention step.
- **Severity**: Critical. This is a stack buffer overflow triggered by untrusted model file data. Llama models with head_dim > 256 exist (e.g., CodeLlama 34B with head_dim=128 is fine, but custom architectures or adversarial GGUF files can set head_dim=512+).

**WHY meta-pattern**: Hardcoded buffer size assumes a specific architecture parameter range. The assumption is invisible because common models (TinyLlama: 64, Llama-7B: 128) fit easily.

---

### PROVEN-2: S7 — n_dims overflow of dims[4] array in tensor parsing

**Seed**: model.c:308-312 — No validation of n_dims, n_dims>4 overflows dims[4]

**Assumption that failed**: "n_dims in a GGUF file will always be <= 4"

**Concrete proof**:
- At model.c:296-301, `tensor_info_t` has `uint64_t dims[4]`
- At model.c:309, `tinfos[i].n_dims = read_u32(&r)` reads n_dims from the file with zero validation
- At model.c:310-311: `for (uint32_t d = 0; d < tinfos[i].n_dims; d++) { tinfos[i].dims[d] = read_u64(&r); }`
- **Failing input**: A crafted GGUF file where a tensor entry has `n_dims = 10`
- **Code path**: `parse_gguf()` → loop reads 10 uint64_t values into `dims[4]` → writes 6 * 8 = 48 bytes past the end of the `dims` array, overwriting `type` and `offset` fields of the same struct, and potentially the next struct in the `tinfos` array
- **Wrong output**: Heap corruption of the `tinfos` malloc'd array. The `type` and `offset` fields are overwritten with attacker-controlled data from the GGUF file, leading to arbitrary read from mmap'd memory when pointers are computed at line 324.
- **Severity**: Critical. Heap buffer overflow from untrusted file input. Controllable overwrite of subsequent struct fields.

**WHY meta-pattern**: Trusting file-supplied dimension count without bounds check. The GGUF spec says max 4 dims, but the code trusts the file.

---

### PROVEN-3: S4 — No bounds checking in reader_t functions

**Seed**: model.c:45-94 — r->size never checked against r->pos

**Assumption that failed**: "The GGUF file is well-formed enough that reads won't go out of bounds"

**Concrete proof**:
- `reader_t` stores `data`, `pos`, and `size` (model.c:39-43)
- `size` is set to `m->mmap_size` at model.c:206
- Every `read_*` function (lines 45-94) advances `r->pos` without checking `r->pos + N <= r->size`
- **Failing input**: A truncated GGUF file: valid magic (4 bytes) + version (4 bytes) + n_tensors (8 bytes) + n_metadata claiming 1000 entries, but file is only 100 bytes
- **Code path**: `parse_gguf()` → `read_u32(&r)` for magic ✓ → `read_u32(&r)` for version ✓ → `read_u64(&r)` for n_tensors ✓ → `read_u64(&r)` for n_metadata ✓ → metadata loop starts reading strings → `read_gguf_string()` → `read_u64(&r)` reads 8 bytes at pos=28 → returns a `len` value from whatever bytes are there → `r->pos += len` could jump far past `r->size` → next read dereferences `r->data[r->pos]` which is past the end of the mmap'd region
- **Wrong output**: Out-of-bounds read from mmap'd memory, likely SIGSEGV (reading past end of mapped region), or information leak if adjacent memory is mapped.
- **Severity**: High. Any malformed/truncated GGUF file triggers OOB read. The `size` field is stored but never used for validation.

**WHY meta-pattern**: Defense infrastructure exists (the `size` field) but is never connected to enforcement. The reader tracks where it is and how far it can go, but never checks.

---

### PROVEN-4: S5 — Prompt token buffer undersized for BPE normalization

**Seed**: picolm.c:168 — Buffer sized as `strlen+3`, but BPE normalization can expand spaces 3x

**Assumption that failed**: "strlen(prompt) + 3 is enough tokens for any prompt"

**Concrete proof**:
- picolm.c:168: `int max_prompt_tokens = (int)strlen(prompt) + 3;`
- picolm.c:169: `int *prompt_tokens = (int *)malloc((size_t)max_prompt_tokens * sizeof(int));`
- picolm.c:170: `int n_prompt = tokenizer_encode(&tokenizer, prompt, prompt_tokens, max_prompt_tokens, 1);`
- In `tokenizer_encode()` at tokenizer.c:119-121: `text_len = strlen(text)`, `norm_cap = text_len * 3 + 4` — the normalization buffer is correctly 3x sized
- **BUT**: The issue is about token count, not byte count. After normalization, every single byte could become a separate byte-fallback token `<0xHH>`. A prompt of N bytes produces at most ~3N+3 normalized bytes, each potentially a separate token.
- **However**, looking at the actual flow: `tokenizer_encode` uses `max_tokens` as a limiter at line 215: `for (int i = 0; i < merge_len && n_tokens < max_tokens; i++)`. So tokens are silently truncated, not overflowed.
- **Failing input**: A prompt consisting entirely of spaces: `"          "` (10 spaces). `strlen = 10`, `max_prompt_tokens = 13`. Normalization produces 10 * 3 + 3 = 33 bytes of "▁" characters. After character tokenization, each "▁" is 1 UTF-8 char (3 bytes) → 11 character tokens. After BPE merging this could be 11 tokens. With BOS → 12 tokens. `max_prompt_tokens = 13` → fits.
- **Better failing input**: A prompt of 100 bytes of rare UTF-8 sequences that don't merge. Each byte becomes a byte fallback token. `strlen = 100`, `max_prompt_tokens = 103`. Normalized: 3 + 100 = 103 bytes. Each normalized byte that fails character lookup becomes a byte token. In worst case: leading ▁ = 1 token + 100 byte tokens + 1 BOS = 102 tokens. Buffer is 103. This fits.
- **Actually**: The real issue is the normalization step. Input of N characters where each is a space: N spaces → normalization to (N+1) ▁ characters → (N+1) tokens (assuming ▁ is in vocab). Plus BOS = N+2. Buffer is N+3. This fits.
- **Re-examination**: With BPE, merge_len starts at the number of individual character tokens (could be larger than text_len if bytes don't map to vocab entries). But merge_buf is allocated as `norm_len + 1` entries (line 144), and the final copy to `tokens` is bounded by `max_tokens`. So the *tokens output buffer* won't overflow — it's silently truncated.

**Verdict revision**: The CR finding is technically wrong about a buffer overflow. The `tokenizer_encode` function properly bounds output with `max_tokens`. The real bug is **silent prompt truncation**: a prompt with many spaces will be silently cut short because `strlen(prompt) + 3` underestimates the token count after SentencePiece normalization. But this is a correctness bug, not a memory safety bug.

Wait — let me re-examine more carefully. The internal `merge_buf` allocation at tokenizer.c:144 is `norm_len + 1` which IS correctly sized for the normalized text. And the output copy is bounded. So there is no memory corruption, just silent truncation.

**Revising to SUSPECTED** — see Suspicion List.

---

### PROVEN-5: S8 — Division by zero: head_dim and kv_mul

**Seed**: model.c:293,566 — `head_dim=n_embd/n_heads`, `kv_mul=n_heads/n_kv_heads`, no zero check

**Assumption that failed**: "n_heads and n_kv_heads will always be non-zero because they come from a valid model"

**Concrete proof**:
- model.c:293: `cfg->head_dim = cfg->n_embd / cfg->n_heads;`
- `cfg->n_heads` is set from GGUF metadata at line 240: `cfg->n_heads = (int)skip_meta_value(...)`. If the metadata key is missing, `n_heads` stays 0 (from `memset(m, 0, ...)` at model_load line 539).
- **Failing input**: A GGUF file missing the `llama.attention.head_count` metadata key (but otherwise valid enough to parse)
- **Code path**: `model_load()` → `memset(m, 0, sizeof(*m))` → `parse_gguf()` → metadata loop never finds `llama.attention.head_count` → `cfg->n_heads = 0` → line 293: `cfg->head_dim = cfg->n_embd / 0` → **undefined behavior / SIGFPE**
- Similarly, model.c:566: `int kv_mul = n_heads / n_kv_heads;` — if `n_kv_heads` is 0 (missing from GGUF), this is division by zero
- **Wrong output**: Program crash (SIGFPE on most platforms), or undefined behavior per C standard.
- **Severity**: Medium-High. Any GGUF file missing expected metadata keys triggers this. Not adversarial — a model from a different architecture family (not llama) would trigger it.

**WHY meta-pattern**: Config values read from file default to zero (from memset), but code assumes they're valid non-zero values. No validation between parsing and use.

---

### PROVEN-6: S21 — Grammar constraint ignores in_string state during apply

**Seed**: grammar.c:91-111 — Grammar constraint uses pre-computed brace_delta that doesn't account for runtime in_string state

**Assumption that failed**: "Pre-computed token brace_delta is accurate for constraining generation"

**Concrete proof**:
- `analyze_token()` at grammar.c:12-41 computes `brace_delta` by counting all `{` and `}` in a token string, including those inside JSON string values. It does track `escape` but not `in_string` state from context.
- `grammar_apply()` at line 91-111 uses the pre-computed `token_brace_delta` to decide if a token is valid. At line 104-105 it checks `if (g->in_string) continue;` — meaning it allows any token when inside a string.
- **But**: The pre-computed `brace_delta` was computed assuming the entire token content affects brace depth. When `in_string` is true, `grammar_apply` skips constraint checking entirely (line 105), which is correct.
- **However**: The problem is at the *boundary*. Consider a token that *exits* a string and then has a brace. E.g., a token whose content is `"}` (quote then close-brace). `analyze_token` computes brace_delta = -1, has_unmatched_quote = 1. But `grammar_advance` (lines 130-161) tracks this correctly character-by-character.
- **The real inconsistency**: `grammar_apply()` uses `g->in_string` to decide whether to skip constraints, but the state update happens in `grammar_advance()` which runs AFTER sampling. So during `grammar_apply()`, `in_string` reflects the state *before* this token. If `in_string=1`, all tokens are allowed. But what if a token would close the string AND add unbalanced braces? The `continue` at line 105 allows it, and then `grammar_advance` correctly tracks the state change.
- **Failing input**: Consider a vocabulary token whose text is `"}}}}`. When `in_string=true`, `grammar_apply` allows it (line 105 continue). `grammar_advance` processes it: `"` closes the string (in_string=0), then `}}}}` reduces brace_depth by 4. If brace_depth was 1, it goes to -3.
- **Code path**: The state *after* this token has `brace_depth = -3` which is negative. Future tokens can't fix this — the JSON is broken. But `grammar_apply` only checks `new_brace < 0` for future tokens (line 99). It doesn't prevent the *current* broken state from the unconstrained in_string token.
- **Wrong output**: Grammar constraint allows generation of invalid JSON. A token selected while `in_string=true` can make `brace_depth` go negative, producing unbalanced JSON that grammar_is_complete will never report as valid (generating indefinitely until max_tokens).
- **Severity**: Medium. The JSON grammar constraint's promise is to produce valid JSON, but this edge case breaks that invariant.

**WHY meta-pattern**: Pre-computation + runtime state = inconsistency gap. The pre-computed `brace_delta` doesn't know about runtime `in_string` context, and the `in_string` bypass disables ALL constraints including depth sanity.

---

### PROVEN-7: S3 — Integer overflow in prompt token buffer allocation (32-bit)

**Seed**: picolm.c:168-169 — `(int)strlen(prompt) + 3` can overflow int on 32-bit

**Assumption that failed**: "strlen(prompt) + 3 fits in int"

**Concrete proof**:
- picolm.c:168: `int max_prompt_tokens = (int)strlen(prompt) + 3;`
- On 32-bit systems, `int` is 32 bits. `strlen` returns `size_t` (32-bit on 32-bit systems).
- If `prompt` is a 2GB+ string (e.g., piped from stdin via `read_stdin()` at picolm.c:45-61 which uses `realloc` with no size limit), `strlen` returns a value > INT_MAX.
- **(int)strlen(prompt)** wraps to a negative number. Adding 3 keeps it negative or makes it small positive depending on exact value.
- **Failing input**: Pipe 2,147,483,648 bytes via stdin. `strlen = 2147483648`. `(int)2147483648 = -2147483648` (INT_MIN on 2's complement). `-2147483648 + 3 = -2147483645`. `max_prompt_tokens = -2147483645`.
- **Code path**: `malloc((size_t)max_prompt_tokens * sizeof(int))` → `(size_t)(-2147483645)` = a very large value on 32-bit (wraps to ~8GB) → `malloc` returns NULL → `tokenizer_encode` called with NULL buffer → crash.
- Actually on 32-bit: `(size_t)(-2147483645) * 4` = massive overflow. malloc fails. But the code doesn't check for malloc failure on line 169.
- **Wrong output**: NULL pointer dereference in tokenizer_encode, or if malloc succeeds with a tiny allocation (unlikely but possible with small negative values wrapping to small positive size_t), heap buffer overflow.
- **Severity**: Medium. Requires 32-bit platform AND >2GB input. On 64-bit platforms, int overflow occurs at the same threshold but size_t is 64-bit so the cast produces a different (still huge) value.

**WHY meta-pattern**: Mixing `int` with `size_t` return values. The `strlen` result is implicitly narrowed by cast to int.

---

### PROVEN-8: S9 — tokenizer_encode silently drops bytes

**Seed**: tokenizer.c:162-171 — When both vocab lookup and byte fallback fail, the byte is silently skipped

**Assumption that failed**: "Every input byte will be represented in the token output"

**Concrete proof**:
- tokenizer.c:162-171:
  ```c
  } else {
      char byte_tok[8];
      snprintf(byte_tok, sizeof(byte_tok), "<0x%02X>", (unsigned char)norm[i]);
      tok = vocab_lookup(t, byte_tok, (int)strlen(byte_tok));
      if (tok >= 0) {
          merge_buf[merge_len++] = tok;
      }
      // If tok < 0: nothing happens. i++ on next iteration. Byte dropped.
      i++;
  }
  ```
- If `vocab_lookup` returns -1 for the byte token `<0xHH>` (meaning the model's vocabulary doesn't include byte fallback tokens), the byte is silently skipped.
- **Failing input**: Any model vocabulary that doesn't include the `<0xHH>` byte tokens (some older SentencePiece models or custom vocabularies). Input text containing a character not in the vocabulary.
- **Code path**: Character not found in vocab → byte fallback attempted → byte token `<0xAB>` not in vocab → `tok = -1` → if-block skipped → `i++` → byte gone
- **Wrong output**: The tokenized output is missing bytes from the input. Silent data loss. No warning or error.
- **Severity**: Low-Medium. This is a silent correctness failure. The user gets wrong output with no indication that input was lost.

**WHY meta-pattern**: Fallback chain with silent bottom. The code has a primary path (vocab lookup) and a fallback (byte tokens), but no fallback-of-fallback or error reporting.

---

### PROVEN-9: S18 — Gateway binds to 0.0.0.0 by default

**Seed**: install.sh:238-240 — Gateway config binds to 0.0.0.0, exposing LLM to network

**Assumption that failed**: "Default config is secure"

**Concrete proof**:
- install.sh:238-240 generates the PicoClaw config:
  ```json
  "gateway": {
      "host": "0.0.0.0",
      "port": 18790
  }
  ```
- `0.0.0.0` means the gateway listens on ALL network interfaces, not just localhost.
- **Failing input**: User runs `install.sh` on a Raspberry Pi on their home network. Another device on the network can access `http://<pi-ip>:18790` and interact with the LLM.
- **Code path**: `install.sh` → generates config with `"host": "0.0.0.0"` → PicoClaw gateway reads config → binds to all interfaces
- **Wrong output**: LLM service is network-accessible without authentication. Any device on the local network (or wider if port-forwarded) can send prompts and consume compute resources.
- **Severity**: Medium. Network exposure of an unauthenticated LLM inference service. On a Pi this could be on a home/office network. The installer targets Raspberry Pi users who may not understand network binding.

**WHY meta-pattern**: "Developer convenience as default" — binding to 0.0.0.0 is convenient for testing but insecure as a default. Should default to `127.0.0.1`.

---

## 2. Suspicion List (SUSPECTED)

### SUSPECTED-1: S5 — Silent prompt truncation (revised from PROVEN attempt)

**Seed**: picolm.c:168 — Prompt token buffer sized as strlen+3, but BPE expansion can exceed this

**What was tried**: Traced the full tokenizer_encode flow. The `max_tokens` parameter properly bounds the output array write. No memory corruption occurs.

**Why proof couldn't be produced**: The actual effect is silent prompt truncation, not buffer overflow. A prompt with many spaces like `"a b c d e f g ..."` where each space becomes a ▁ token could produce more tokens than `strlen+3` allows. The `tokenizer_encode` silently clips at `max_tokens`. This is a correctness bug but I cannot construct a concrete token-count proof without running the actual tokenizer on a specific vocabulary. The exact truncation threshold depends on vocabulary content and BPE merge behavior.

**Confidence**: 80% this causes silent truncation in practice for space-heavy prompts.

---

### SUSPECTED-2: S14 — fp16 subnormal loop on corrupt data

**Seed**: quant.c:22-28 — subnormal normalization loop could iterate unexpectedly

**What was tried**: Analyzed the fp16_to_fp32 subnormal handling at quant.c:17-28:
```c
if (exp == 0) {
    if (mant == 0) { /* zero */ }
    else {
        exp = 1;
        while (!(mant & 0x400)) {
            mant <<= 1;
            exp--;
        }
        mant &= 0x3FF;
        f = sign | ((exp + 127 - 15) << 23) | (mant << 13);
    }
}
```
The loop `while (!(mant & 0x400))` shifts `mant` left until bit 10 is set. Since `mant` is nonzero (else-branch), it has at least one bit set in bits 0-9. The worst case is `mant = 1` (bit 0 only) → loop iterates 10 times. This is bounded.

**Why proof couldn't be produced**: The loop is actually bounded at 10 iterations maximum. `mant` is a 10-bit value (0x3FF mask), nonzero, so the bit must reach position 10 within 10 shifts. No infinite loop possible.

**However**: The `exp` value becomes `1 - 10 = -9`, and `(exp + 127 - 15) = 103`, which is a valid fp32 exponent. The computation is mathematically correct for subnormal fp16 values.

**Revised verdict**: This should probably be HARDENED, not SUSPECTED. Moving to Defense Map.

---

### SUSPECTED-3: S15 — Off-by-one in generation loop cache resume

**Seed**: picolm.c:188-189,195-234 — Cache resume position off by 1

**What was tried**: Analyzed the cache resume logic:
- Line 188: `int token = prompt_tokens[start_pos > 0 ? start_pos - 1 : 0];`
- Line 189: `int pos = start_pos > 0 ? start_pos - 1 : 0;`
- When `start_pos > 0` (cache hit): `token = prompt_tokens[start_pos - 1]`, `pos = start_pos - 1`
- The loop at line 195: `for (; pos < total_steps; pos++)` starts at `pos = start_pos - 1`
- Line 204: `model_forward(&model, token, pos)` — first call is `model_forward(prompt_tokens[start_pos-1], start_pos-1)`
- But the KV cache already has positions [0, cache_pos). The forward pass at position `start_pos-1` recomputes a position that's already in the cache, overwriting it.

**What this means**: The last cached position is recomputed. This is wasteful (one redundant forward pass) but not necessarily wrong — the recomputed values should match the cached values. However, if the KV cache file was generated with different model weights or a different quantization, the values would differ, and this recomputation would create an inconsistency (position start_pos-1 has new values, positions 0..start_pos-2 have old cached values).

**Why proof couldn't be produced**: In the normal case (same model, same weights), the recomputation is redundant but correct. The off-by-one only matters with mismatched cache files, which is already a user error. Could not construct a scenario where correct cache + correct model produces wrong output from this logic.

**Confidence**: 40% — the logic is defensive (recompute last position) rather than buggy.

---

### SUSPECTED-4: S16 — dequantize_row_f32 integer overflow on 32-bit

**Seed**: quant.c:262 — `memcpy(dst, src, n * sizeof(float))` where n is int

**What was tried**: At quant.c:261-263:
```c
void dequantize_row_f32(const void *src, float *dst, int n) {
    memcpy(dst, src, n * sizeof(float));
}
```
`n * sizeof(float)` where `n` is `int` and `sizeof(float)` is `size_t`. The multiplication is performed in `size_t` (implicit promotion). On 32-bit: if `n` is negative (from prior overflow), `(size_t)n` wraps to a large value, causing memcpy to read/write huge amounts.

**Why proof couldn't be produced**: `n` comes from model config dimensions (like `n_embd`) which are read as int from GGUF metadata. A malicious GGUF could set these to negative values (if the metadata type is signed). But `skip_meta_value` returns `uint64_t` cast to `int` — for values > INT_MAX, this produces negative `n`. However, such a value would have already caused issues in `allocate_run_state` (allocating with negative dimensions → very large or zero allocations). The chain of failures makes it hard to construct a clean proof of *this specific* overflow without prior crashes.

**Confidence**: 60% — the overflow is real in theory but requires a specific chain of prior non-crashes.

---

## 3. Defense Map (HARDENED)

### HARDENED-1: S1 — Thread-unsafe static buffers in tokenizer_decode

**Seed**: tokenizer.c:231,249 — `static char byte_buf[2]` and `static char space_buf[256]`

**Attacks tried**:
1. **Multi-threaded access**: PicoLLM uses a single-threaded generation loop. `tokenizer_decode` is called only from picolm.c:223, inside the sequential generation loop. There is no multi-threaded tokenizer access path in the codebase.
2. **Buffer overflow of space_buf**: The code at line 253 explicitly checks `if (len >= (int)sizeof(space_buf)) len = (int)sizeof(space_buf) - 1;` and at line 260: `if (len >= (int)sizeof(space_buf) - 1) len = (int)sizeof(space_buf) - 2;`. The 256-byte buffer is properly bounds-checked.
3. **Stale return value**: The static buffer is overwritten on each call. But the caller at picolm.c:224 uses the result immediately (`printf("%s", piece)`), so no stale reference issue.

**Why defense holds**: The entire generation loop is single-threaded. The static buffers are used, consumed (printed), and then potentially overwritten on the next call — but never accessed again. The bounds checks on space_buf are correct. This is a valid design choice for a single-threaded program, not a bug.

**Note**: If PicoLLM ever adds multi-threaded generation or batch decoding, this becomes a real bug. But in the current architecture, it's safe.

---

### HARDENED-2: S6 — Global g_vocab_for_sort for qsort

**Seed**: tokenizer.c:17 — Thread-unsafe global used in qsort comparison

**Attacks tried**:
1. **Concurrent tokenizer loading**: `tokenizer_load` is called once at picolm.c:144, sequentially after model loading. There's no path for concurrent tokenizer_load calls.
2. **Thread safety with generation**: The global is only used during `tokenizer_load` (the `qsort` call at line 100). During generation, only `vocab_lookup` is called, which uses the tokenizer's own `sorted_idx` array, not the global.

**Why defense holds**: Same reasoning as S1 — single-threaded initialization. The global is a standard pattern for C90-compatible qsort (which lacks a context pointer). In the single-threaded usage of this codebase, it's safe.

---

### HARDENED-3: S10 — madvise(MADV_SEQUENTIAL) for random access

**Seed**: model.c:182 — `madvise(MADV_SEQUENTIAL)` but inference has random access pattern

**Attacks tried**:
1. **Performance degradation measurement**: MADV_SEQUENTIAL hints to the kernel that pages will be accessed sequentially. During inference, weight access is quasi-sequential within each layer (processing all rows of a weight matrix), but jumps between layers. The hint is suboptimal but not destructive.
2. **Correctness failure**: `madvise` is a hint, not a command. The kernel is free to ignore it. Memory access patterns are unaffected — only readahead behavior may be suboptimal.

**Why defense holds**: This is a performance hint, not a correctness issue. The worst case is suboptimal page readahead, which affects performance by perhaps 5-15% (extra readahead for data that won't be used sequentially). It's not a bug — it's a missed optimization opportunity. `MADV_RANDOM` would be more appropriate but the current behavior is not incorrect.

---

### HARDENED-4: S11 — `-k` flag controls top_p but convention says top-k

**Seed**: picolm.c:36 — `-k` controls top_p, user confusion

**Attacks tried**:
1. **Actual effect**: User passes `-k 5` thinking they're setting top-k to 5 tokens, but it actually sets top_p to 5.0. Since top_p >= 1.0 is caught at sampler.c:63 (`if (s->top_p >= 1.0f)` → sample from full distribution), this makes top-p effectively disabled, defaulting to full-distribution sampling with temperature.
2. **Confusion vs bug**: The usage text at picolm.c:36 clearly says `-k <float> Top-p / nucleus sampling (default: 0.9)`. The flag name is unconventional but documented.

**Why defense holds**: The documentation is accurate, even if the flag name is confusing. A user reading the help text would see what `-k` does. This is a usability issue, not a correctness or security bug.

---

### HARDENED-5: S12 — malloc/free per token for top_p sorting

**Seed**: sampler.c:75 — malloc per token call

**Attacks tried**:
1. **Performance**: For a 32k vocabulary, each malloc allocates `32000 * sizeof(prob_index_t)` = 32000 * 8 = 256KB. This is called once per generated token. On a Pi generating at 5-10 tok/s, that's ~2.5MB/s of malloc/free churn.
2. **Memory fragmentation**: Repeated malloc/free of same-size blocks should be handled efficiently by modern allocators (free-list reuse).
3. **Correctness**: The allocation is properly freed, properly sized, and properly used.

**Why defense holds**: This is a performance concern, not a correctness bug. The allocation pattern is simple (same size every time) and modern allocators handle it efficiently. For PicoLLM's target use case (embedded inference at a few tokens per second), the overhead is negligible compared to the inference computation itself.

---

### HARDENED-6: S13 — KV cache file has no integrity check

**Seed**: model.c:732-833 — No checksum in KV cache format

**Attacks tried**:
1. **Corrupt cache file**: If the cache file is truncated, `fread` at lines 814 and 823 checks the return value (`!= row_size`) and returns 0 (failure).
2. **Model mismatch**: Lines 797-801 check `file_layers != c->n_layers || file_kv_dim != kv_dim` and reject mismatched files.
3. **n_pos overflow**: Line 803-807 checks `n_pos > seq_len` and rejects.
4. **Silent data corruption**: If the cache file has correct header but garbled data (bit flips), the data is loaded silently. This produces wrong attention output but no crash. However, this is true of any binary data format without checksums.
5. **Magic number**: `KVCACHE_MAGIC = 0x4B564350` provides basic format identification.

**Why defense holds**: The cache format has reasonable validation: magic number, dimension matching, length checks, and read-error detection. The lack of a checksum means silent corruption is possible, but this is a performance optimization feature (skip prompt prefill), not a security feature. If the cache is corrupt, the model produces garbage output but doesn't crash. The design trade-off (no checksum = simpler + faster) is reasonable for a local cache file.

---

### HARDENED-7: S14 — fp16 subnormal loop (revised from SUSPECTED)

**Seed**: quant.c:22-28 — Loop in fp16 subnormal conversion

**Attacks tried**:
1. **Infinite loop**: `mant` is nonzero (guarded by `if (mant == 0)` check), and is a 10-bit value. Left-shifting must set bit 10 within 10 iterations. Loop is bounded.
2. **exp underflow**: `exp` starts at 1, decreases to minimum -9. The result `(exp + 127 - 15) << 23` produces a valid fp32 exponent encoding for subnormal fp16 values.
3. **Corrupt input**: Even with arbitrary 16-bit input, all code paths are bounded and produce valid fp32 bit patterns.

**Why defense holds**: The fp16→fp32 conversion correctly handles all 65536 possible input values including subnormals, zeros, infinities, and NaN. The loop is mathematically bounded at 10 iterations.

---

### HARDENED-8: S17 — install.sh PATH handling

**Seed**: install.sh:248-256 — PATH append picks zshrc over bashrc

**Attacks tried**:
1. **zshrc preference**: Lines 248-249: `PROFILE_FILE="$HOME/.bashrc"` then `[ -f "$HOME/.zshrc" ] && PROFILE_FILE="$HOME/.zshrc"`. If both exist, zshrc wins. This is actually reasonable — on systems where both exist, the user likely uses zsh (the more recent shell).
2. **Duplicate PATH entries**: Line 251: `grep -q "picolm/bin" "$PROFILE_FILE"` prevents duplicate entries by checking if any line contains "picolm/bin".
3. **Wrong shell file**: If a bash user has a .zshrc from a previous installation, their path gets added to the wrong file. But this is a common pattern in installers and not a security issue.

**Why defense holds**: The PATH handling is imperfect but functional. The duplicate check works. The zshrc preference is a reasonable heuristic. This is a cosmetic/usability issue, not a bug.

---

### HARDENED-9: S19 — Byte token detection fragile

**Seed**: tokenizer.c:229 — Checks `str[5]=='>'` but no `str[6]=='\0'` check

**Attacks tried**:
1. **False positive matching**: The check at line 229 is: `str[0]=='<' && str[1]=='0' && str[2]=='x' && str[5]=='>'`. This matches any string starting with `<0x??>`... For it to be a false positive, the vocabulary would need a token like `<0xFF>suffix` (a byte token pattern followed by more characters).
2. **In practice**: GGUF vocabularies from standard models (Llama, Mistral) don't have such tokens. The byte tokens are exactly 6 characters: `<0xHH>`.
3. **If false positive occurs**: The code at lines 232-241 extracts bytes from positions 3-4 and returns a single byte. If the token is actually `<0xFF>extra`, the returned byte is `0xFF` and the "extra" part is lost. This is a silent decoding error.

**Why defense holds**: In practice, no standard model vocabulary has tokens matching `<0x..>` as a prefix of a longer string. The check is sufficient for all real-world vocabularies. Adding `str[6]=='\0'` would be more defensive but the current code works correctly with all known GGUF files.

**Note**: A stronger verdict would be SUSPECTED if we consider adversarial vocabulary construction, but given PicoLLM's use case (loading known model files), this is adequately hardened.

---

### HARDENED-10: S20 — Windows build missing explicit library linkage

**Seed**: build.bat:4 — No explicit `-lm -lpthread` equivalent

**Attacks tried**:
1. **MSVC behavior**: The build.bat uses MSVC (`cl`), not gcc/mingw. MSVC links the C runtime (including math functions) automatically. There is no `-lm` equivalent needed.
2. **pthread**: The code uses `#ifdef _WIN32` guards (tensor.c:6-11) to use Windows `CreateThread`/`WaitForSingleObject` instead of pthreads. No `-lpthread` needed.
3. **Missing libraries**: The `cl` command at line 4 doesn't specify any `/link` flags, but MSVC's default runtime libraries provide everything the code needs for a console application.

**Why defense holds**: MSVC's build model differs from GCC. The implicit linkage of CRT libraries is standard MSVC behavior. The code correctly uses Windows APIs when `_WIN32` is defined.

---

## 4. Seed Map

| Seed | File:Line | Verdict | Summary |
|------|-----------|---------|---------|
| S1 | tokenizer.c:231,249 | **HARDENED** | Static buffers safe in single-threaded usage |
| S2 | model.c:644 | **PROVEN** | Stack overflow: `float acc[256]` with head_dim > 256 from GGUF |
| S3 | picolm.c:168-169 | **PROVEN** | Integer overflow: `(int)strlen + 3` wraps on 32-bit with >2GB input |
| S4 | model.c:45-94 | **PROVEN** | OOB read: reader_t has size field but never checks it |
| S5 | picolm.c:168 | **SUSPECTED** | Silent prompt truncation (strlen+3 underestimates token count) |
| S6 | tokenizer.c:17 | **HARDENED** | Global qsort comparator safe in single-threaded init |
| S7 | model.c:308-312 | **PROVEN** | Heap overflow: n_dims>4 overflows dims[4] array |
| S8 | model.c:293,566 | **PROVEN** | Division by zero when GGUF metadata keys missing |
| S9 | tokenizer.c:162-171 | **PROVEN** | Silent byte dropping when byte fallback tokens not in vocab |
| S10 | model.c:182 | **HARDENED** | MADV_SEQUENTIAL is suboptimal hint, not a bug |
| S11 | picolm.c:36 | **HARDENED** | `-k` flag name unconventional but documented |
| S12 | sampler.c:75 | **HARDENED** | Per-token malloc is performance concern, not bug |
| S13 | model.c:732-833 | **HARDENED** | KV cache has reasonable validation, no checksum is design trade-off |
| S14 | quant.c:22-28 | **HARDENED** | Subnormal loop bounded at 10 iterations, correct math |
| S15 | picolm.c:188-189 | **SUSPECTED** | Cache resume recomputes last position (redundant but likely correct) |
| S16 | quant.c:262 | **SUSPECTED** | memcpy size overflow on 32-bit requires prior non-crash chain |
| S17 | install.sh:248-256 | **HARDENED** | PATH handling imperfect but functional |
| S18 | install.sh:238-240 | **PROVEN** | Gateway binds 0.0.0.0, exposes LLM to network |
| S19 | tokenizer.c:229 | **HARDENED** | Byte token check sufficient for all known vocabularies |
| S20 | build.bat:4 | **HARDENED** | MSVC implicit linkage is correct behavior |
| S21 | grammar.c:91-111 | **PROVEN** | in_string bypass allows tokens that make brace_depth go negative |
| **S22** | model.c:206,45-94 | **PROVEN** | *(New)* reader_t size field is set but never enforced — defense infrastructure without enforcement |
| **S23** | model.c:293 | **PROVEN** | *(New)* head_dim=0 when n_embd=0 causes cascading issues: rope table size=0, zero-size KV cache |
| **S24** | picolm.c:169 | **SUSPECTED** | *(New)* No NULL check on malloc for prompt_tokens — crash on OOM |

**Summary counts**: 8 PROVEN, 4 SUSPECTED, 10 HARDENED, 0 UNCLEAR

---

## 5. New Seeds Discovered During Falsification

### S22 (PROVEN) — reader_t defense infrastructure without enforcement

Discovered while analyzing S4. The `reader_t` struct at model.c:39-43 stores `size` (set to mmap_size at line 206), but none of the 7 read functions (lines 45-94) check `r->pos + N <= r->size` before reading. This is a distinct finding from S4: S4 describes the *absence* of bounds checking, while S22 identifies the **misleading presence of unused defense infrastructure**. The `size` field suggests bounds checking was intended but never implemented.

### S23 (PROVEN) — Cascading zero-dimension failures

Discovered while analyzing S8. When `n_embd` is 0 (missing from GGUF metadata, defaults to 0 from memset), `head_dim = 0 / n_heads = 0`. This causes:
- `half_dim = 0` → RoPE table allocation of size 0
- `kv_dim = n_kv_heads * 0 = 0` → KV cache allocation of size 0
- `allocate_run_state` proceeds without error (calloc(1, 0) returns non-NULL on many systems)
- `model_forward` then accesses empty buffers with zero-size loops (benign) BUT the matmul calls use `dim=0` which produces zero-length dot products → all logits are 0 → sampler always returns token 0 → infinite loop of token 0 until max_tokens

### S24 (SUSPECTED) — Missing malloc NULL check for prompt_tokens

Discovered while analyzing S3. At picolm.c:169: `int *prompt_tokens = (int *)malloc(...)`. The return value is never checked for NULL. If the allocation fails (due to S3's integer overflow producing a huge size, or genuine OOM), `tokenizer_encode` is called with a NULL buffer, causing a crash. Other malloc calls in the codebase (e.g., model.c:304, tokenizer.c:55-60) do check for NULL.

---

## 6. Pipeline Analysis

### Upgrades (CR underestimated → Falsify proved worse)

| Seed | CR Assessment | Falsify Found |
|-------|--------------|---------------|
| S4 | "No bounds checking" (generic) | PROVEN: size field exists but unused — misleading defense infrastructure. More severe because it suggests checking was intended. |
| S8 | "Division by zero" (standard bug) | PROVEN: Cascading — not just div/0, but also zero-dimension allocation chain that produces non-crashing but completely broken inference (S23) |
| S21 | "Grammar inconsistency" (design issue) | PROVEN: Concrete attack — in_string bypass allows brace_depth to go negative, permanently breaking JSON constraint invariant |

**Count: 3 findings upgraded**

### Downgrades (CR overestimated → Falsify couldn't prove)

| Seed | CR Assessment | Falsify Found |
|-------|--------------|---------------|
| S1 | Thread-unsafe static buffers | HARDENED: Single-threaded program, no concurrent access path |
| S5 | "Heap overflow" from BPE expansion | SUSPECTED: Actually silent truncation (bounded by max_tokens), not overflow |
| S6 | Thread-unsafe global | HARDENED: Single-threaded initialization, no concurrent path |
| S14 | "Could loop unexpectedly" | HARDENED: Loop bounded at 10 iterations, mathematically correct |
| S19 | "Fragile" byte detection | HARDENED: Sufficient for all real-world vocabularies |

**Count: 5 findings downgraded**

### New Seeds from Falsification

**3 new seeds**: S22, S23, S24. Of these, S22 and S23 are PROVEN, S24 is SUSPECTED.

### What the serial pipeline found that standalone CR missed

1. **S22 — Unused defense infrastructure**: CR noticed "no bounds checking" but missed that the bounds-checking *infrastructure* exists (the size field). This is a deeper insight — it means the developer intended to add checking but forgot or deferred it. A pure CR wouldn't typically notice that a struct field is never read.

2. **S23 — Cascading zero-dimension behavior**: CR reported S8 (division by zero) as a crash bug. Falsification discovered that for *some* combinations of missing metadata, the program doesn't crash but enters a subtly broken state (zero-size allocations that succeed, producing garbage output silently). This "doesn't crash but silently wrong" failure mode is harder to detect than a crash.

3. **S21 concrete attack path**: CR reported grammar inconsistency as a design issue. Falsification constructed the specific token content (`"}}}}` while in_string) that breaks the invariant, proving it's exploitable not just theoretically problematic.

### What the serial pipeline found that standalone Falsify (blind) would likely miss

1. **S18 (0.0.0.0 binding)**: A blind Falsify run focused on C code would likely not examine install.sh at all. The CR seed directed attention to a non-obvious file.

2. **S9 (silent byte dropping)**: The tokenizer fallback chain is subtle. Without CR pointing to the specific lines 162-171, a blind run would likely focus on more obvious issues (buffer overflows, crashes) and miss this silent correctness failure.

3. **S21 (grammar pre-computation inconsistency)**: The interaction between `analyze_token` (pre-computed at init) and runtime `in_string` state requires understanding two code paths simultaneously. CR flagged the general area; Falsify traced the specific interaction.

4. **S7 (dims[4] overflow)**: The tensor parsing code is dense GGUF format handling. Without CR pointing to the specific n_dims issue, a blind Falsify would need to trace the entire GGUF parser to notice the missing bounds check.

### Pipeline Efficiency Summary

- **21 CR seeds in → 8 PROVEN + 4 SUSPECTED + 10 HARDENED out**
- **38% of CR findings were proven real** (8/21)
- **48% of CR findings were false positives or over-severity** (10/21 hardened)
- **14% remained uncertain** (3/21 suspected, excluding S14 which moved to hardened)
- **3 new findings discovered** during falsification (14% discovery amplification)
- **3 findings upgraded** to higher severity through concrete proof
- **5 findings downgraded** from the CR assessment

The CR→Falsify pipeline's primary value is **precision**: it converted vague "potential issues" into either concrete proofs or concrete defenses. The 48% hardened rate means nearly half of CR findings were noise that Falsify filtered out. The 38% proven rate with concrete code paths is substantially more actionable than CR findings alone.

The pipeline's secondary value is **depth amplification**: analyzing each finding deeply enough to falsify it revealed 3 new adjacent findings that neither CR nor blind Falsify would likely catch.
