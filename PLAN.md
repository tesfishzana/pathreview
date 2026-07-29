## Solution plan

**Issue:** [Faithfulness checker can never mark short claims as supported](https://github.com/ascherj/pathreview/issues/152)

---

### Understand

**Root cause — two compounding bugs in `rag/evaluator/faithfulness_checker.py`:**

**Bug 1 — `_extract_claims()` min-length filter is too aggressive.**
The current filter is `len(s.strip()) > 10`, which discards any sentence of 10 characters or fewer. Short but fully valid claims such as `"Knows Rust."` (10 chars stripped) are silently dropped before they ever reach the scoring logic. The condition should use a much smaller threshold (e.g., `> 3`) that only filters empty or near-empty fragments.

**Bug 2 — `_is_supported()` overlap threshold is too high for short claims.**
The method requires `len(meaningful_overlap) >= 2` non-stopword tokens. A short claim like `"Python expert."` shares only the single keyword `"python"` with a fully supporting context chunk, so it always fails the `>= 2` check and is scored as unsupported. The fix is to detect when the claim itself has fewer than 2 meaningful tokens and lower the required overlap to `>= 1` in that case.

**Expected behavior:** Short claims (`"Python expert."`, `"Knows SQL."`) that are clearly supported by context should be scored as supported, pushing the overall faithfulness score above 0.0.

**Actual behavior:** Those claims are either silently discarded in step 1 or always return `False` from step 2, causing feedback composed of short, fully-supported claims to score `0.0`.

**Confirmed reproduction:** Running the three named failing tests locally (`test_partial_support_returns_middle_score`, `test_multiple_context_chunks`, `test_multiple_claims_varying_support`) with `pytest -v` produces `assert 0.2 < 0.0` (all three return `0.0`).

---

### Map

**Primary file to change:**
- `rag/evaluator/faithfulness_checker.py`
  - `_extract_claims()` (line 55): lower `> 10` threshold
  - `_is_supported()` (line 74): add guard for claims with fewer than 2 meaningful tokens

**Test file (no new tests needed — existing failing tests are the acceptance criteria):**
- `tests/unit/test_faithfulness_checker.py`
  - `test_partial_support_returns_middle_score` (line 46)
  - `test_multiple_context_chunks` (line 86)
  - `test_multiple_claims_varying_support` (line 170)

No other files need to change — the bug is fully contained within the single `faithfulness_checker.py` module.

---

### Plan

1. **Lower the min-length threshold in `_extract_claims()`**
   Change `len(s.strip()) > 10` to `len(s.strip()) > 3`. This retains short claims while still dropping truly empty or single-character fragments produced by trailing punctuation splits.

2. **Count meaningful tokens in the claim inside `_is_supported()`**
   After filtering out stop words from `claim_tokens`, count how many meaningful tokens the claim contains (`claim_meaningful = claim_tokens - stop_words`).

3. **Apply a lower overlap threshold for short claims**
   If `len(claim_meaningful) < 2`, require `len(meaningful_overlap) >= 1` instead of `>= 2`. Otherwise keep the existing `>= 2` threshold for longer claims.

4. **Run `make check`** (ruff linter + black formatter + mypy type checker) and fix any issues.

5. **Run `make test-unit`** to confirm all three previously failing tests now pass and no existing passing tests regress.

---

### Inputs & outputs

| Function | Input | Output (before fix) | Output (after fix) |
|---|---|---|---|
| `_extract_claims("Python expert. Knows Rust. Skilled with Docker.")` | feedback string | `["Python expert", "Skilled with Docker"]` (≤10-char "Knows Rust" dropped) | `["Python expert", "Knows Rust", "Skilled with Docker"]` (all retained) |
| `_is_supported("Python expert", "Python and Docker expertise...")` | claim + context | `False` (only 1 overlap token, needs 2) | `True` (claim has <2 meaningful tokens → threshold lowered to 1) |
| `checker.check("Python expert. Knows Rust. Skilled with Docker.", [...])` | feedback + chunks | `0.0` | `~0.67` (2 of 3 claims supported) |

---

### Risks & unknowns

- **Threshold `> 3` may still admit noisy fragments** like `"Ok"` or `"No"` from unusual input. These are unlikely in real feedback but should be verified with the edge-case tests already present in the test file.
- **The `>= 1` fallback could increase false positives** for very short claims. The guard (`len(claim_meaningful) < 2`) keeps this scoped only to genuinely short claims — longer claims still require 2-token overlap.
- **The `stop_words` set is minimal** — common words like `"has"`, `"with"`, `"their"` are not included. Expanding it is out of scope for this fix but is a follow-up risk if false positives appear in integration testing.
- **`len(s.strip()) > 3` vs a different value** — need to verify that `"SQL"` (3 chars) passes (`> 3` would still filter it). May need to use `>= 3` or `> 2` if single-word claims are a real use case. Will confirm against edge-case test `test_multiple_claims_varying_support` output.

---

### Edge cases

| Input | Expected behavior |
|---|---|
| Claim `"SQL"` (3 chars, no period) | Currently filtered by `> 3` — test whether this needs `>= 3` or `> 2` |
| Claim with zero meaningful tokens (`"Is the a."`) | `claim_meaningful` is empty → `len(meaningful_overlap) >= 1` check still returns `False` (no overlap) |
| Claim with 1 meaningful token that IS in context (`"Python."`) | Extracted (5 chars > 3), `claim_meaningful = {"python"}`, overlap = 1 → `True` |
| Claim with 1 meaningful token NOT in context (`"Rust."`) | Extracted (4 chars > 3), `claim_meaningful = {"rust"}`, overlap = 0 → `False` |
| All-short-claim feedback (`"Knows SQL. Knows Git. Knows Docker."`) | All three extracted and independently scored |
| Feedback with >10 claims | Existing cap of `[:10]` still applies; no change needed |
| Empty feedback string | Early return `0.0` unchanged — not affected by this fix |
