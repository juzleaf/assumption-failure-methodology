# Code Review + Systematic Debugging Report

## Repository: PicoLLM (picolm)

An ultra-lightweight LLM inference engine in C, supporting GGUF model files with quantized weights, targeting Raspberry Pi and embedded devices. ~2,500 lines of C across 7 source files.

## Summary

PicoLLM is a well-structured, minimal LLM inference engine. The codebase is clean and readable, with good cross-platform support (Linux/macOS/Windows/RISC-V). However, the review uncovered several significant issues: a thread-safety problem with static buffers in the tokenizer decoder, multiple buffer overflow risks (especially the fixed-size flash attention accumulator), missing bounds checks on GGUF parsing that could lead to out-of-bounds reads on malformed files, an integer overflow in prompt token allocation, and logic issues in the generation loop's first-token handling. There are also several medium-severity issues around error handling and resource cleanup.

## Findings

### Finding 1: Thread-Unsafe Static Buffers in tokenizer_decode

- **Severity**: High
- **File(s)**: `/tmp/afm-benchmark-round3/picolm-cr/picolm/tokenizer.c` (lines 231, 249)
- **Description**: `tokenizer_decode` uses `static char byte_buf[2]` and `static char space_buf[256]` as return buffers. These are shared across all calls. While the current code is single-threaded for decoding, any future multi-threaded usage or even calling `tokenizer_decode` twice before consuming the first result (e.g., in a printf format string with two calls) will silently corrupt data.
- **Evidence**:
```c
static char byte_buf[2];      // line 231
static char space_buf[256];   // line 249
```
- **Impact**: Data corruption if the function is called more than once before the caller consumes the result. The second call overwrites the first call's return value.
- **Recommendation**: Either (a) accept a caller-provided buffer, (b) use thread-local storage (`__thread` / `_Thread_local`), or (c) document clearly that the return value is only valid until the next call.

### Finding 2: Stack Buffer Overflow in Flash Attention Accumulator

- **Severity**: Critical
- **File(s)**: `/tmp/afm-benchmark-round3/picolm-cr/picolm/model.c` (line 644)
- **Description**: The flash attention accumulator is declared as `float acc[256]` on the stack, with a comment saying "head_dim is typically 64-128". However, `head_dim` is computed from model config as `n_embd / n_heads` and is never validated. A model with a large head dimension (e.g., n_embd=8192, n_heads=8 gives head_dim=1024) would write 1024 floats (4096 bytes) into a 256-float (1024-byte) stack buffer, causing stack buffer overflow.
- **Evidence**:
```c
float acc[256]; /* head_dim is typically 64-128 */
memset(acc, 0, (size_t)head_dim * sizeof(float));
```
Then later:
```c
for (int d = 0; d < head_dim; d++) {
    acc[d] = acc[d] * correction + fp16_to_fp32(vt[d]);
}
```
- **Impact**: Stack smashing, potential code execution if a crafted GGUF model file is loaded. This is both a correctness bug and a security issue.
- **Recommendation**: Either dynamically allocate `acc` based on `head_dim`, or use `s->xb2` as the accumulator (which is already allocated to `n_embd` size), or add a `MAX_HEAD_DIM` compile-time constant with a runtime check.

### Finding 3: Integer Overflow in Prompt Token Buffer Allocation

- **Severity**: High
- **File(s)**: `/tmp/afm-benchmark-round3/picolm-cr/picolm/picolm.c` (line 168-169)
- **Description**: `max_prompt_tokens` is computed as `(int)strlen(prompt) + 3`, then used as `(size_t)max_prompt_tokens * sizeof(int)` for malloc. For a very long prompt (close to INT_MAX), `strlen(prompt) + 3` could overflow the `int` type, wrapping to a small or negative value, which then gets cast to `size_t` (a very large number on 64-bit, or a small allocation on 32-bit with a negative int). On 32-bit systems (Pi Zero), the negative-to-size_t conversion is especially dangerous — it would allocate a tiny buffer while writing many tokens into it.
- **Evidence**:
```c
int max_prompt_tokens = (int)strlen(prompt) + 3;
int *prompt_tokens = (int *)malloc((size_t)max_prompt_tokens * sizeof(int));
```
- **Impact**: Heap buffer overflow on very long prompts, especially on 32-bit systems (which this project explicitly targets for Raspberry Pi).
- **Recommendation**: Use `size_t` for `max_prompt_tokens`, add overflow checks, and validate the prompt length is reasonable before allocating.

### Finding 4: No Bounds Checking in GGUF reader_t Functions

- **Severity**: High
- **File(s)**: `/tmp/afm-benchmark-round3/picolm-cr/picolm/model.c` (lines 45-94)
- **Description**: All reader_t functions (`read_u8`, `read_u16`, `read_u32`, `read_u64`, `read_f32`, `read_gguf_string`) advance `r->pos` without checking against `r->size`. A malformed or truncated GGUF file can cause out-of-bounds reads from the mmapped memory region. The `size` field is stored in the struct but never checked.
- **Evidence**:
```c
static uint8_t read_u8(reader_t *r) {
    uint8_t v = r->data[r->pos];  // no bounds check
    r->pos += 1;
    return v;
}
```
The `reader_t` struct has a `size` member that is initialized in `parse_gguf` but never used:
```c
reader_t r = { .data = (const uint8_t *)m->mmap_addr, .pos = 0, .size = m->mmap_size };
```
- **Impact**: Out-of-bounds reads on malformed GGUF files. Since the data is mmapped, this will cause SIGSEGV on Linux/macOS or access violation on Windows if reading past the mapped region. Not exploitable for code execution in the typical case (PROT_READ only), but causes an unhandled crash.
- **Recommendation**: Add bounds checking to each reader function. At minimum, check `r->pos + N <= r->size` before reading N bytes, and return an error/abort gracefully on failure.

### Finding 5: Prompt Token Allocation Size Underestimates BPE Token Count

- **Severity**: Medium
- **File(s)**: `/tmp/afm-benchmark-round3/picolm-cr/picolm/picolm.c` (line 168)
- **Description**: The maximum prompt token buffer is sized as `strlen(prompt) + 3`. This is a character-count-based estimate. However, after SentencePiece normalization in `tokenizer_encode`, each space character becomes a 3-byte UTF-8 sequence, and each original byte could become a separate token. In the worst case (many spaces), the normalized text is up to 3x the original length, and the initial character-level tokenization produces up to `norm_len` tokens (which can be up to `3 * strlen(prompt) + 3`). These are then BPE-merged down, but the initial `merge_buf` allocation in `tokenizer_encode` is based on `norm_len`, while the final output is bounded by `max_tokens`. The issue is that the *output* buffer `prompt_tokens` with `max_prompt_tokens = strlen(prompt) + 3` could be too small if BPE merges don't reduce enough tokens for a text consisting mostly of rare byte sequences.
- **Evidence**:
```c
int max_prompt_tokens = (int)strlen(prompt) + 3;  // picolm.c:168
// In tokenizer_encode, worst case output is merge_len tokens
// merge_len can be up to norm_len = text_len * 3 + 3
```
- **Impact**: In pathological cases (text of mostly spaces or unusual UTF-8 byte sequences), the output token count could exceed the allocated buffer, causing heap buffer overflow.
- **Recommendation**: Size the buffer as `strlen(prompt) * 3 + 4` to match the worst-case normalized-text length, or better yet, have `tokenizer_encode` return an error when `max_tokens` is insufficient and handle that in the caller.

### Finding 6: Global Variable for qsort Comparison (Thread-Unsafe)

- **Severity**: Medium
- **File(s)**: `/tmp/afm-benchmark-round3/picolm-cr/picolm/tokenizer.c` (line 17)
- **Description**: `g_vocab_for_sort` is a global variable used to pass context to the `cmp_sorted` qsort comparator. This is inherently thread-unsafe. If two tokenizers are loaded concurrently, they will race on this global.
- **Evidence**:
```c
static char **g_vocab_for_sort; /* global for qsort comparison */

static int cmp_sorted(const void *a, const void *b) {
    int ia = *(const int *)a;
    int ib = *(const int *)b;
    return strcmp(g_vocab_for_sort[ia], g_vocab_for_sort[ib]);
}
```
- **Impact**: Data corruption if tokenizer loading is ever parallelized (unlikely in current usage, but a latent defect).
- **Recommendation**: Use `qsort_r` (POSIX) / `qsort_s` (C11) which accept a context parameter, or accept the limitation with a comment.

### Finding 7: Missing Validation of n_dims in GGUF Tensor Info Parsing

- **Severity**: High
- **File(s)**: `/tmp/afm-benchmark-round3/picolm-cr/picolm/model.c` (lines 308-312)
- **Description**: When parsing tensor info, `n_dims` is read from the file and used to index into a fixed-size `dims[4]` array. If a malformed GGUF file has `n_dims > 4`, the loop will write past the end of `tinfos[i].dims`, corrupting memory.
- **Evidence**:
```c
tinfos[i].n_dims = read_u32(&r);
for (uint32_t d = 0; d < tinfos[i].n_dims; d++) {
    tinfos[i].dims[d] = read_u64(&r);  // overflow if n_dims > 4
}
```
- **Impact**: Heap buffer overflow on crafted GGUF files. Since `tinfos` is heap-allocated, this overwrites adjacent heap memory.
- **Recommendation**: Add `if (tinfos[i].n_dims > 4) { /* error */ }` before the loop.

### Finding 8: Unchecked Division by Zero in Config Derivation

- **Severity**: Medium
- **File(s)**: `/tmp/afm-benchmark-round3/picolm-cr/picolm/model.c` (line 293, 566)
- **Description**: `head_dim = n_embd / n_heads` and `kv_mul = n_heads / n_kv_heads` are computed without verifying that `n_heads` and `n_kv_heads` are non-zero. If a GGUF file doesn't contain these metadata keys, they remain 0 (from `memset(m, 0, ...)` in `model_load`), causing division by zero.
- **Evidence**:
```c
cfg->head_dim = cfg->n_embd / cfg->n_heads;  // model.c:293
// ...
int kv_mul = n_heads / n_kv_heads;            // model.c:566
```
- **Impact**: SIGFPE (floating point exception / division by zero) crash on malformed model files missing required metadata.
- **Recommendation**: Validate all required config values after parsing metadata. If any critical values are 0, return an error from `parse_gguf`.

### Finding 9: tokenizer_encode Skips Byte Tokens Silently

- **Severity**: Medium
- **File(s)**: `/tmp/afm-benchmark-round3/picolm-cr/picolm/tokenizer.c` (lines 162-171)
- **Description**: In the character-to-token conversion loop, if a character cannot be found in the vocabulary and the byte-level fallback token (`<0xHH>`) is also not in the vocabulary, the byte is silently dropped — `i` advances but no token is appended to `merge_buf`.
- **Evidence**:
```c
tok = vocab_lookup(t, byte_tok, (int)strlen(byte_tok));
if (tok >= 0) {
    merge_buf[merge_len++] = tok;
}
// if tok < 0: byte is silently dropped, i++ happens
i++;
```
- **Impact**: Silent data loss during tokenization. The prompt's content will be subtly different from what the user intended, with no warning.
- **Recommendation**: At minimum, print a warning to stderr. Ideally, return an error or use a dedicated unknown-token ID.

### Finding 10: madvise(MADV_SEQUENTIAL) Suboptimal for Random Access Pattern

- **Severity**: Low
- **File(s)**: `/tmp/afm-benchmark-round3/picolm-cr/picolm/model.c` (line 182)
- **Description**: After mmapping the model file, `MADV_SEQUENTIAL` is used. However, during inference, the model weights are accessed in a pattern that is not purely sequential — different layers' weights are accessed per-token, and the embedding lookup is a random access by token ID. `MADV_RANDOM` or the default `MADV_NORMAL` would be more appropriate for the weight access pattern during inference.
- **Evidence**:
```c
madvise(addr, m->mmap_size, MADV_SEQUENTIAL);
```
- **Impact**: Suboptimal page pre-fetching behavior. The OS kernel may aggressively read-ahead sequentially when the actual access pattern is more random, wasting I/O bandwidth and memory pressure. On RAM-constrained devices (the primary target), this could cause unnecessary page evictions.
- **Recommendation**: Use `MADV_NORMAL` (default) or `MADV_RANDOM`. Consider using `MADV_SEQUENTIAL` only during the initial GGUF parsing phase, then switching to `MADV_RANDOM` before inference.

### Finding 11: Usage Help Shows Wrong Flag Semantics (-k for top_p)

- **Severity**: Low
- **File(s)**: `/tmp/afm-benchmark-round3/picolm-cr/picolm/picolm.c` (line 36)
- **Description**: The `-k` flag is documented as "Top-p / nucleus sampling" but the letter `k` typically refers to "top-k" sampling in LLM tooling convention. The actual variable it controls is `top_p`. This is confusing for users familiar with LLM inference.
- **Evidence**:
```c
fprintf(stderr, "  -k <float>     Top-p / nucleus sampling (default: 0.9)\n");
// ...
} else if (strcmp(argv[i], "-k") == 0 && i + 1 < argc) {
    top_p = (float)atof(argv[++i]);
}
```
- **Impact**: User confusion. Users expecting `-k` to set top-k (integer number of candidates) will instead set top-p (probability mass cutoff), getting unexpected generation behavior.
- **Recommendation**: Rename the flag to `-p` (but that's taken for prompt). Use `--top-p` or rename current `-p` to `--prompt` and use `-p` for top-p. Alternatively, keep `-k` but add a `--top-p` alias and update the help text.

### Finding 12: sampler_sample Allocates and Frees on Every Call for top_p

- **Severity**: Medium
- **File(s)**: `/tmp/afm-benchmark-round3/picolm-cr/picolm/sampler.c` (line 75)
- **Description**: When `top_p < 1.0`, `sampler_sample` calls `malloc` and `free` for a `prob_index_t` array of size `vocab_size` on every single token generation. For a 32k vocabulary, this is ~256KB allocated and freed per token. On memory-constrained embedded systems (the primary target), this causes heap fragmentation and is a significant performance bottleneck in the hot path.
- **Evidence**:
```c
prob_index_t *sorted = (prob_index_t *)malloc((size_t)vocab_size * sizeof(prob_index_t));
// ... sort, sample ...
free(sorted);
```
- **Impact**: Performance degradation and heap fragmentation on every generated token when top_p < 1.0.
- **Recommendation**: Pre-allocate the sort buffer in `sampler_init` and store it in the `sampler_t` struct, or add it to the run_state_t scratch space.

### Finding 13: KV Cache File Has No Integrity Check

- **Severity**: Medium
- **File(s)**: `/tmp/afm-benchmark-round3/picolm-cr/picolm/model.c` (lines 732-833)
- **Description**: The KV cache save/load mechanism writes raw FP16 data with a minimal header (magic + dimensions). There is no checksum or hash to verify data integrity. A corrupted or partially written cache file (e.g., from a crash during save) will be loaded without error, silently producing garbage output.
- **Evidence**: The only validation is magic number and dimension matching:
```c
if (header[0] != KVCACHE_MAGIC) { ... }
if (file_layers != c->n_layers || file_kv_dim != kv_dim) { ... }
```
No file size validation against expected size, no CRC/checksum.
- **Impact**: Silent garbage generation from corrupted cache files. The user would see nonsensical output with no indication that the cache is corrupt.
- **Recommendation**: At minimum, validate the file size matches the expected `header_size + 2 * n_layers * n_pos * kv_dim * sizeof(uint16_t)`. Ideally, add a CRC32 at the end of the file.

### Finding 14: fp16 Subnormal Renormalization Can Underflow Exponent

- **Severity**: Low
- **File(s)**: `/tmp/afm-benchmark-round3/picolm-cr/picolm/quant.c` (lines 22-28)
- **Description**: In `fp16_to_fp32`, the subnormal handling loop decrements `exp` while shifting `mant` left to find the leading bit. Since `exp` starts at 1 and `mant` can have up to 10 leading zeros (for the smallest subnormal), `exp` can go as low as -9. The final computation `(exp + 127 - 15) << 23` would then be `(exp + 112) << 23`. For `exp = -9`, this gives `103 << 23`, which is a valid FP32 exponent. So this is technically correct, but the loop condition `!(mant & 0x400)` combined with the `exp--` can loop at most 10 times before `mant` is guaranteed to have bit 10 set (since `mant != 0` is checked). The code is correct but fragile — if a caller passes arbitrary 16-bit values (not valid FP16), the loop could run unexpectedly many times.
- **Evidence**:
```c
while (!(mant & 0x400)) {
    mant <<= 1;
    exp--;
}
```
- **Impact**: Minimal in practice; the loop is bounded by the 10-bit mantissa width. But with corrupt data, behavior could be surprising.
- **Recommendation**: Add a loop iteration bound (e.g., `max 10 iterations`) as a safety guard.

### Finding 15: Potential Off-by-One in Generation Loop Token Handling

- **Severity**: Medium
- **File(s)**: `/tmp/afm-benchmark-round3/picolm-cr/picolm/picolm.c` (lines 188-189, 195-234)
- **Description**: When `start_pos > 0` (cache hit), the initial token is set to `prompt_tokens[start_pos - 1]` and pos starts at `start_pos - 1`. The loop then has a guard `if (pos < start_pos) { token = prompt_tokens[pos]; continue; }` which fires for exactly one iteration (when `pos == start_pos - 1`). This means the token at `start_pos - 1` is forwarded through the model, and the output is used to get the next token at `prompt_tokens[start_pos]` (via the prefill branch). This logic appears correct but is unnecessarily confusing and wastes one forward pass re-computing a position that was already cached.
- **Evidence**:
```c
int token = prompt_tokens[start_pos > 0 ? start_pos - 1 : 0];
int pos = start_pos > 0 ? start_pos - 1 : 0;
// ...
for (; pos < total_steps; pos++) {
    if (pos < start_pos) {
        token = prompt_tokens[pos];
        continue;  // skips the forward pass? No — token was already set above
    }
```
Wait — actually re-reading: the `continue` skips `model_forward`. So `pos` advances from `start_pos - 1` to `start_pos` without doing inference. Then at `pos == start_pos`, `model_forward` runs with the token set to `prompt_tokens[start_pos - 1]`. But the *output* of that forward pass needs to be the logits for position `start_pos`, meaning the model sees token at `start_pos - 1` at position `start_pos`. This is off by one — the model is being told this token is at position `start_pos` when it was originally at position `start_pos - 1`.
- **Impact**: When using KV cache, position IDs will be shifted by 1 compared to the original encoding. This means RoPE embeddings are wrong for all subsequent tokens, producing degraded generation quality.
- **Recommendation**: The cached token at the boundary needs to be forwarded at its correct position. The simplest fix: set `pos = start_pos` and `token = prompt_tokens[start_pos]` directly, since positions `[0, start_pos)` are already in the KV cache and don't need re-computation.

### Finding 16: dequantize_row_f32 Uses Implicit int-to-size_t Conversion

- **Severity**: Low
- **File(s)**: `/tmp/afm-benchmark-round3/picolm-cr/picolm/quant.c` (line 262)
- **Description**: `dequantize_row_f32` computes `n * sizeof(float)` where `n` is `int`. If `n` is very large and `sizeof(float)` is 4, the multiplication could overflow on 32-bit systems before being passed to `memcpy`.
- **Evidence**:
```c
void dequantize_row_f32(const void *src, float *dst, int n) {
    memcpy(dst, src, n * sizeof(float));  // n * 4 can overflow int on 32-bit
}
```
- **Impact**: On 32-bit targets, if `n > 536,870,912`, the size wraps. Unlikely in practice since model dimensions are much smaller, but the code targets 32-bit ARM (Pi Zero).
- **Recommendation**: Cast to `(size_t)n * sizeof(float)`.

### Finding 17: install.sh Appends to PATH Without Checking for Duplicates Properly

- **Severity**: Low
- **File(s)**: `/tmp/afm-benchmark-round3/picolm-cr/install.sh` (lines 248-256)
- **Description**: The PATH modification check `grep -q "picolm/bin"` is overly broad — it matches any line containing "picolm/bin", even if commented out. Running the installer multiple times with different `PICOLM_DIR` values would add multiple PATH entries. Also, the script unconditionally chooses `.zshrc` if it exists, even if the user's login shell is bash.
- **Evidence**:
```bash
PROFILE_FILE="$HOME/.bashrc"
[ -f "$HOME/.zshrc" ] && PROFILE_FILE="$HOME/.zshrc"

if ! grep -q "picolm/bin" "$PROFILE_FILE" 2>/dev/null; then
```
- **Impact**: Shell profile pollution on repeated installations with different directories.
- **Recommendation**: Use a more precise grep pattern (e.g., the exact `export PATH` line) and detect the actual running shell from `$SHELL`.

### Finding 18: install.sh Gateway Binds to 0.0.0.0 By Default

- **Severity**: Medium
- **File(s)**: `/tmp/afm-benchmark-round3/picolm-cr/install.sh` (lines 238-240)
- **Description**: The default PicoClaw gateway configuration binds to `0.0.0.0:18790`, making it accessible from all network interfaces. On a Raspberry Pi connected to a network, this exposes the LLM inference gateway to all devices on the LAN (and potentially the internet if the Pi has a public IP).
- **Evidence**:
```json
"gateway": {
    "host": "0.0.0.0",
    "port": 18790
}
```
- **Impact**: Unauthorized access to LLM inference endpoint. Anyone on the same network could send prompts to the user's Pi.
- **Recommendation**: Default to `127.0.0.1` (localhost only). Users who want network access can change it explicitly.

### Finding 19: tokenizer_decode Byte Token Detection is Fragile

- **Severity**: Low
- **File(s)**: `/tmp/afm-benchmark-round3/picolm-cr/picolm/tokenizer.c` (line 229)
- **Description**: The byte token pattern matching checks `str[5] == '>'` to detect `<0xHH>` format. This assumes the hex digits are exactly 2 characters. A vocab entry like `<0x0>` (1 hex digit) or `<0x100>` (3 hex digits) would not be detected. Additionally, there is no check that `str[6] == '\0'`, so a token starting with `<0xAB>more_text` would be incorrectly treated as a byte token, with `more_text` silently dropped.
- **Evidence**:
```c
if (str[0] == '<' && str[1] == '0' && str[2] == 'x' && str[5] == '>') {
```
- **Impact**: Incorrect decoding of non-standard byte tokens, though standard GGUF models use the expected `<0xHH>` format.
- **Recommendation**: Add `&& str[6] == '\0'` to the condition, or use a proper parsing approach (check total length is 6, validate hex chars at positions 3-4).

### Finding 20: Windows Build Missing pthread and math Library Linkage

- **Severity**: Low
- **File(s)**: `/tmp/afm-benchmark-round3/picolm-cr/picolm/build.bat` (line 4)
- **Description**: The Windows build command uses `cl` without linking any math or threading libraries. The `tensor.c` threading code has `#ifdef _WIN32` guards using Windows API (`CreateThread`), which is correct. However, `cl` does not automatically link `kernel32.lib` for some build configurations, and the math functions (`cosf`, `sinf`, `expf`, `sqrtf`, `powf`) require a runtime that `cl` should provide by default via MSVCRT. This is likely fine for MSVC but the build command is minimal and fragile.
- **Evidence**:
```batch
cl /O2 /W3 /Fe:picolm.exe picolm.c model.c tensor.c quant.c tokenizer.c sampler.c grammar.c
```
No explicit `/link kernel32.lib` or error handling.
- **Impact**: Build may fail on non-standard MSVC configurations. Low risk since the CI tests this.
- **Recommendation**: Add explicit library linking: `cl /O2 /W3 /Fe:picolm.exe *.c /link kernel32.lib` for robustness.

### Finding 21: Grammar Constraint Ignores in_string State During Masking

- **Severity**: Medium
- **File(s)**: `/tmp/afm-benchmark-round3/picolm-cr/picolm/grammar.c` (lines 91-111)
- **Description**: In `grammar_apply`, when `g->in_string` is true, the function skips all constraint checks for non-EOS tokens (`if (g->in_string) continue;`). This means that when inside a JSON string, no constraints are applied at all. However, the pre-computed `token_brace_delta` values were computed without awareness of string context — a token containing `"}"` (closing brace inside a string literal) would have a negative brace_delta even though it shouldn't affect structural depth. This inconsistency means the `brace_depth` tracking in `grammar_advance` (which correctly tracks string state) and the constraint masking in `grammar_apply` (which uses pre-computed deltas ignoring string context) can diverge.

    Specifically: the `grammar_advance` function correctly ignores structural characters inside strings, but `grammar_apply`'s pre-computed deltas don't account for string context. The `continue` when `in_string` is true papers over this for the most part, but when a token *starts* a string (contains `"`), its pre-computed delta may include structural characters that appear after the opening quote, incorrectly constraining the token.

- **Evidence**:
```c
// In grammar_apply:
if (g->in_string) continue;  // skip all checks inside string
// But token_brace_delta was computed counting all { and } in the token,
// including those that appear after a quote character in the same token
```
- **Impact**: In edge cases, valid JSON tokens could be incorrectly masked (false negative — a valid token gets -inf), or invalid tokens could be allowed. For example, a token like `": {"` has brace_delta=+1, but if we're already in a string when this token is generated, the brace context is wrong. The `grammar_advance` would correctly handle it (the `{` comes after `"` closes the string), but the masking logic might incorrectly allow or reject tokens.
- **Recommendation**: This is a fundamental limitation of pre-computed per-token deltas. The simplest improvement is to not mask tokens at all when `in_string` (which is what the code does), and accept that grammar constraints are weaker inside strings. Document this limitation.

## Statistics

- Total findings: 21
- Critical: 1, High: 4, Medium: 7, Low: 9
